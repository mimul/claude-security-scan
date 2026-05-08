---
name: security-full-scan
description: 릴리스 전 전체 정적 분석 스킬 (Semgrep + Trivy Augmented). 대상 경로 필수. Semgrep(코드 SAST) + Trivy(의존성·시크릿) + Claude 자체 security-full-scan을 모두 취합하여 종합 분석. 수동 호출 전용은 /security-full-scan (다언어 공통)
disable-model-invocation: true
---

# security-full-scan

**역할**: 릴리스 전에 전체 파일 정적 분석 스킬 (언어·프레임워크 비의존)

**분석 방식**: Semgrep + Trivy + Claude 자체 Full Scan으로 **3중 취합 분석**

**도구 역할 분담**:
- **Semgrep**: 정적 코드 분석 (SAST), 깊이 있는 코드 레벨 취약점 탐지
- **Trivy**: 의존성 CVE, 시크릿 누출, Misconfiguration, SBOM 분석

프로젝트 구조를 자동 감지하고, 아래를 병렬 실행한다:
- **소스 모듈 서브에이전트** (감지된 모듈마다 1개): 전체 소스 파일의 정적 분석 (Python, Rust, Go, Ruby, Java, JavaScript/TypeScript 등)
- **패키지 서브에이전트**: 의존성 알려진 CVE 스캔 + 시크릿 누출 체크

> **컨텍스트 상한 주의**: 리포지토리가 대규모인 경우, 전체 파일이 컨텍스트 윈도우에 들어가지 않을 수 있다. **스킵이 발생하면 절대 조용히 무시하지 말고, 반드시 스킵한 파일 목록과 함께 PARTIAL 스캔으로 보고한다.** (대형 Python/Django, Rust Cargo 프로젝트, Java Spring, Node.js 모노레포 등에서 특히 주의)

---

## 인수

| 인수 | 필수 | 설명 |
|------|------|------|
| `대상 경로` | 필수 | 명시적으로 스캔하고 싶은 디렉토리 (생략 시 자동 감지) |
| `mode` | 임의 | `basic` 또는 `augmented` (기본값: `augmented`) |

---

## 절차

### Step 0: 도구 선실행 (권장)

**Augmented Mode 사용 시 아래 명령어를 실행하세요** (대상 경로 기준):

```bash
TARGET_PATH="YOUR_TARGET_PATH"   # 예: . 또는 ./backend

# 1. Semgrep - 깊이 있는 코드 SAST
semgrep scan --config=auto \
             --severity=ERROR,WARNING \
             --sarif-output ./security-reports/semgrep.sarif \
             $TARGET_PATH || true

# 2. Trivy - 의존성, 시크릿, misconfig, SBOM
trivy fs --format json \
         --output ./security-reports/trivy-results.json \
         --severity HIGH,CRITICAL \
         --scanners vuln,secret,misconfig,license \
         $TARGET_PATH || true
```

### Step 1: 프로젝트 구조 감지 + 도구 결과 로드

아래 순서로 구조를 파악한다:

1. **소스 디렉토리의 특정**
   - `package.json` / `pyproject.toml` / `requirements.txt` / `Cargo.toml` / `go.mod` / `Gemfile` / `pom.xml` / `build.gradle` / `setup.py` 등을 `node_modules/` / `.git/` / `dist/` / `build/` / `target/` / `__pycache__` / `venv/` 등을 제외하고 검색한다.
   - 각 매니페스트 파일 위치로부터 소스 디렉토리를 추정한다 (예: `package.json` → `src/`/`lib/`/`app/`, `Cargo.toml` → `src/`, Python → `src/` 또는 프로젝트 루트, Java → `src/main/java` 등).
   - 인수로 경로가 지정된 경우 그것을 우선한다.

2. **언어·프레임워크의 감지**
   - 매니페스트 파일 내용으로부터 언어와 프레임워크를 특정한다.
   - 예: `next`/`express` → JavaScript/Node.js, `fastapi`/`django`/`flask` → Python, `actix-web`/`axum`/`rocket` → Rust, `gin`/`echo`/`fiber` → Go, `rails` → Ruby, `spring-boot` → Java 등. dependencies에서 `react`, `sqlalchemy`, `sqlx`, `gorm`, `activerecord`, `springframework` 등을 확인.

3. **./security-reports/ 폴더에서 semgrep.sarif와 trivy-results.json 자동 로드**
   - 결과 파일이 없으면 "대상 경로 기준으로 Step 0 명령을 먼저 실행해주세요" 안내한다.

4. **스캔 전에 파일 목록을 확정한다 (필수)**:
   - 각 소스 디렉토리의 파일을 `find` 명령으로 열거하고, **총 파일 수를 기록한다** (`*.py`, `*.rs`, `*.go`, `*.rb`, `*.java`, `*.js`, `*.ts`, `*.jsx`, `*.tsx` 등).
   - 이 시점에 확정된 총 파일 수가 커버리지 계산의 분모가 된다.

5. **모듈 분할의 결정**
   - 파일 목록과 규모를 바탕으로 분할 방식을 결정한다.
   - 소스 디렉토리가 여러 개인 경우 (모노레포, Python 패키지, Rust workspace, Go modules, Java multi-module, Ruby gems 등): 모듈마다 서브에이전트를 할당한다.
   - 소스 디렉토리가 1개인 경우: 서브에이전트 1개로 전체를 커버한다.
   - 모듈이 4개 이상인 경우: 관련 모듈을 그룹화한다 (컨텍스트 비용 절감).
   - 「이 스캔은 아래 모듈을 대상으로 합니다: [모듈 목록] / 총 파일 수: [N]건」이라고 명시한다.

### Step 2: 서브에이전트를 병렬 실행

아래를 **동시에** 기동한다 (병렬 실행):

---

#### 소스 모듈 서브에이전트 (모듈별)

감지된 각 소스 모듈의 파일을 정적 분석한다.

**분석 절차 (필수):**

1. **분석 시작 전에 전체 파일을 열거한다**: `find <src_dir> -type f \( -name "*.py" -o -name "*.rs" -o -name "*.go" -o -name "*.rb" -o -name "*.java" -o -name "*.js" -o -name "*.ts" ... \)`으로 대상 파일의 완전한 리스트를 취득하고, **총수 N을 기록한다**.
2. **전체 파일을 순서대로 읽어 분석한다**: 아래 우선순위 순으로 읽어 나간다.
3. **컨텍스트 상한에 도달한 경우**: 분석을 즉시 중지하고, 「읽지 못한 파일 목록」을 기록한다. **미분석 파일을 스킵한 채로 스캔 완료로 보고해서는 안 된다.**

**읽기 우선순위 (컨텍스트 상한 시에만 의미 있음):**

| 우선순위 | 파일 종류 | 범용 패턴 예 (다언어) |
|----------|----------|---------------------|
| 1 | HTTP 엔트리포인트·라우터 | `routes/`, `controllers/`, `api/`, `handlers/`, Python: Flask/FastAPI views, Rust: axum/actix handlers, Go: gin handlers, Ruby: Rails controllers, Java: Spring @RestController, JS: Express routers |
| 2 | 미들웨어·필터 | `middleware/`, `filters/`, `interceptors/`, Python: Flask middleware, Rust: tower, Go: middleware, Java: Spring Interceptor |
| 3 | 인증·인가 | `auth/`, `security/`, `permissions/`, Python: Flask-Login/Django auth, Rust: axum extractors, Java: Spring Security |
| 4 | 외부 입력을 받는 계층 | 요청 바디 처리 파일 (Python: Pydantic, Rust: serde Deserialize, Go: struct tags, Java: @RequestBody) |
| 5 | 데이터 접근 계층 | `repository/`, `models/`, `db/`, Python: Django ORM/SQLAlchemy, Rust: sqlx/diesel, Go: GORM, Java: JPA/Hibernate |
| 6 | 외부 서비스 연동 | `clients/`, `services/`, `integrations/` |
| 7 | 유틸리티·헬퍼 | `utils/`, `lib/`, `helpers/` |

**보고 의무 (필수):**
- 분석 완료 후 반드시 `FILES_ANALYZED: <분석 완료 수> of <총수>`를 보고한다.
- 미분석 파일이 있는 경우 **파일 경로를 전부 열거한다**.
- `분석 완료 수 < 총수`인 경우, 결과를 **PARTIAL SCAN**으로 명시한다.

**검출 대상 (신뢰도 8/10 이상만 보고):**
- **인젝션**: SQL/NoSQL/명령/템플릿 인젝션, Path Traversal (Python: f-string SQL, Rust: raw query, Java: Statement, JS: template literals)
- **인증·인가 결함**: JWT 오구현, 인가 바이패스 (Python: Flask-Login 미사용, Java: Spring Security Config 누락 등)
- **하드코딩된 인증 정보·API 키**
- **XSS**: innerHTML, template unsafe rendering (Python: Jinja2 autoescape off, JS: React dangerouslySetInnerHTML, Ruby: ERB raw)
- **PII·비밀번호 로그 출력**
- **SSRF**: 사용자 입력으로 URL 제어 시

**제외 규칙 (보고하지 않음):**
- DoS·리소스 고갈·레이트 리미팅 (기본 제외)
- 클라이언트 사이드 인증 체크 결여
- 환경 변수 경유 공격
- Markdown·문서 파일
- Rust `unsafe` 미사용 일반 패턴, Python `eval`이 아닌 안전한 동적 코드, Java Reflection 일반 사용 등

**거짓 양성 필터링:**
- 실제 공격 경로가 존재하는가
- 프레임워크 내장 보호 (Django ORM, sqlx, Spring Security, Express sanitizer 등)가 작동하는 경우 제외

---

#### 패키지 서브에이전트

> **Trivy와의 관계**: Trivy(Step 0)가 의존성 CVE와 시크릿을 포괄 스캔한다. 패키지 서브에이전트는 언어별 전용 도구(cargo audit, npm audit 등)로 더 정밀한 결과를 추가하는 역할이다. Trivy 결과가 있는 경우에도 병행 실행하여 교차 검증한다. Trivy가 없거나 Step 0을 건너뛴 경우에는 이 서브에이전트가 의존성 스캔의 주 수단이 된다.

**의존성 스캔:**

| 매니페스트 | 명령 | 언어 |
|------------|------|------|
| `package.json` | `npm audit` | JavaScript/TypeScript |
| `requirements.txt`/`pyproject.toml` | `pip-audit` / `safety` | Python |
| `Cargo.toml` | `cargo audit` | Rust |
| `go.mod` | `govulncheck ./...` | Go |
| `Gemfile` | `bundle audit` | Ruby |
| `pom.xml`/`build.gradle` | `dependency-check` | Java |

**시크릿 누출 체크:**
- `gitleaks detect --source . --report-format json` 또는 `trufflehog filesystem .`

---

### Step 3: 결과의 통합·중복 제외

1. Step 2 소스 서브에이전트 결과 수집 (Claude security-Full-Scan):

- Step 2에서 병렬 실행된 소스 모듈 서브에이전트의 분석 결과를 취합한다
- 프로젝트 전체 맥락, 비즈니스 로직, 아키텍처 수준 취약점을 중심으로 정리한다

2. Semgrep 결과 반영 (코드 SAST):

- Step 0에서 생성된 `semgrep.sarif`를 읽어 이슈 목록을 추출한다
- 각 이슈를 프로젝트 맥락 기반으로 재평가하여 False Positive를 제거한다

3. Trivy 결과 반영:

- Step 0에서 생성된 `trivy-results.json`을 읽어 의존성 CVE, 시크릿, Misconfiguration 이슈를 추출한다
- Trivy 결과를 프로젝트 컨텍스트와 함께 해석한다

4. 종합 교차 검증:

- 위 1 + 2 + 3 결과를 교차 검증하고 중복 제거
- 동일한 파일/모듈에서 발견된 문제들을 연계 분석
- Semgrep(코드)과 Trivy(의존성/시크릿)가 동시에 지적한 부분을 High Priority로 승격
- Claude 자체 분석에서만 발견된 고수준·로직 취약점 강조
- 언어별 특화 취약점 체크 (Rust, Python, Java 등)

### Step 4: 우선순위화 및 수정 가이드 작성

(Step 3의 통합·중복 제거 결과를 기반으로 수정 가이드 작성에 집중)

1. Top Priority Issues 별도 강조 (Claude + Semgrep + Trivy 3중에서 중복 발견된 항목)
2. 심각도별 정렬 (Critical → High → Medium → Low)
3. 언어·프레임워크별 구체적 수정 방안 제시 (예: Python `bandit`, Rust `cargo audit fix`, Java `Spring Security` config, Go `net/http` secure headers 등)

**Critical 발견 시**: 즉시 보고하고 즉시 대응 촉구.

### Step 5: 커버리지 집계

```yaml
scan_date: <ISO8601>
scan_type: full_static

project:
  detected_modules: <감지된 모듈명 리스트>
  languages: <감지된 언어 리스트> 
  frameworks: <감지된 프레임워크 리스트>

coverage:
  modules:
    - name: <모듈명>
      files_total: <총 파일 수>
      files_analyzed: <분석 완료 수>
      context_limit_reached: <true/false>
  skipped_files:
    - path: <스킵한 파일>
      reason: <이유 (컨텍스트 상한·바이너리 등)>

packages:
  scanned: <스캔 완료 매니페스트 수>
  tools_unavailable: <이용 불가능했던 도구 리스트>
  vulns_high_critical: <건수>

findings:
  critical: <건수>
  high: <건수>
  medium: <건수>
  low: <건수>

not_covered_by_this_scan:
  - 배포 후 런타임 동작 → /security-scan 사용
  - 비즈니스 로직 결함 → 침투 테스트 (인력) 필요
  - 인프라·클라우드 설정 → 인프라 담당자 리뷰 필요
  - 제로데이 취약성 → CVE 데이터베이스 외이므로 검출 불가
  - 멀티스텝 공격 체인 → 침투 테스트 필요

# coverage_score = files_analyzed / files_total × 100 (전 모듈 합산)
coverage_score: <files_analyzed 합계 / files_total 합계 × 100>%
scan_status: <COMPLETE | PARTIAL>
ci_result: <pass/fail>
```

### Step 6: 보고서 생성·이력 인덱스 갱신

1. 날짜 붙인 파일명으로 보고서를 저장한다
   - 보고서: `./security-reports/YYYY-MM-DD-security-full-scan-report.md`
   - 커버리지: `./security-reports/YYYY-MM-DD-security-full-scan-coverage.yml`
   - 날짜는 실행 시 ISO 8601 형식 (예: 2026-05-03)
   - 디렉토리가 없으면 생성한다

2. 심각도별 정렬·수정 우선순위·구체적인 수정 명령을 포함한다 (Rust: cargo audit fix 등).

3. `./security-reports/index.md` 에 1행 추가한다
   - 파일이 없으면 아래 공통 헤더부터 생성한다:
   ```
   | 날짜 | 스캔 유형 | 대상 | Critical | High | Medium | CI 결과 | 리포트 |
   |------|-----------|------|----------|------|--------|---------|--------|
   ```
   - 추가 형식:
   ```
   | YYYY-MM-DD | security-full-scan | 전체 소스 (<총 파일 수>files / <scan_status>) | <Critical 건수> | <High 건수> | <Medium 건수> | <CI 결과> | [보고서](YYYY-MM-DD-security-full-scan-report.md) |
   ```
   - CI 결과: High 이상 발견이 0건이라면 `✅ pass`, 1건 이상이라면 `❌ fail`

4. High 이상 발견이 있는 경우, 종료 코드 1을 보고한다.

## 이 스킬로 커버할 수 없는 영역

| 영역  | 추천 수단  |
|------|---------|
|배포 후 HTTP 헤더·동적 동작|/security-scan|
|비즈니스 로직 결함 (사양 지식 필요)|침투 테스트|
|인프라·클라우드 설정 (IAM·네트워크 ACL)|인프라 담당자 리뷰|
|제로데이 취약성|CVE 데이터베이스 외이므로 검출 불가|
|멀티스텝 공격 체인|침투 테스트|

## 완료 조건

- [ ] 감지된 모듈·언어·프레임워크가 명시되어 있다
- [ ] `FILES_ANALYZED: X of Y` 가 보고되어 있다
- [ ] 스킵된 파일이 있으면 파일 경로가 전부 열거되어 있다
- [ ] `./security-reports/YYYY-MM-DD-security-full-scan-coverage.yml` 이 생성되어 있다 (scan_type: full_static 명기)
- [ ] `not_covered_by_this_scan` 이 명시되어 있다
- [ ] Semgrep·Trivy 결과 로드 여부가 보고서에 명시되어 있다 (없으면 "Step 0 미실행"으로 기재)
- [ ] `./security-reports/YYYY-MM-DD-security-full-scan-report.md` 이 생성되어 있다
- [ ] CI 결과 (High 이상 발견 0건 → ✅ pass, 1건 이상 → ❌ fail) 가 명시되어 있다
- [ ] `./security-reports/index.md` 에 1행 추가되어 있다

