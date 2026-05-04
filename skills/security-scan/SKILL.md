---
name: security-scan
description: 스테이징 환경 런타임 검증 스킬 (ZAP + Nuclei Augmented). 대상 URL 기반으로 Docker(zaproxy/zap-stable)+ Nuclei(템플릿 기반) + Claude security-scan 결과를 종합 분석. 수동 호출 전용: /security-scan [환경명] (다언어 지원)
disable-model-invocation: true
---

# security-scan

**역할**: 런타임 검증 전문 스킬 (정적 분석·의존성 스캔은 `/security-full-scan` 담당)

**도구 역할 분담**:
- **OWASP ZAP**: 종합 웹 애플리케이션 스캐닝 (Passive Scan, AJAX 크롤링, 헤더 분석), Docker 이미지 (`zaproxy/zap-stable`) 사용
  (`zap-baseline.py` 기준. Active Scan이 필요하면 `zap-full-scan.py`로 교체)
- **Nuclei**: 빠르고 정확한 템플릿 기반 취약점 스캔 (수천 개의 최신 취약점 템플릿)

배포 후 서버를 실제로 호출하여 “코드만 읽어서는 알 수 없는” 사항을 검증한다.  
설정 파일 반영 누락·헤더 실제 출력·동적 동작 확인에 특화한다. (Python FastAPI/Django, Rust Axum/Actix, Go Gin/Echo, Ruby Rails, Java Spring Boot, JavaScript/Node.js Express 등 모든 언어/프레임워크 공통)

---

## 인수

| 인수 | 필수 | 설명 |
|------|------|------|
| `대상 환경` | 임의 | 환경명 (예: `staging`). 생략 시 설정 파일의 `target.base_url` 사용 |
| `mode` | 임의 | `basic` 또는 `augmented` (기본값: `augmented`) |

## 전제 조건

- 현재 디렉토리에 `security-agent.config.yml`이 존재할 것 (필수 — `target.base_url` 항목이 스캔 대상 URL이 됨)
- 환경 변수 `STAGING_URL`이 설정되어 있으면 `target.base_url`보다 우선 적용 (선택 사항)
- **가동 중인 서버가 필요** (이 스킬은 소스코드를 읽지 않음)
- **Docker가 설치되어 있을 것** (ZAP Augmented Mode 실행에 필요)
- Python (Gunicorn/Uvicorn), Rust (Axum/Tokio), Go (net/http), Ruby (Puma), Java (Spring Boot/Tomcat), Node.js (Express/NestJS) 등 모든 백엔드 서버에 적용 가능

---

## 절차

### Step 0: 도구 선실행 (Augmented Mode — Claude가 자동 실행)

`mode=augmented`(기본값)인 경우, Claude는 아래 절차를 **Bash tool로 직접 순서대로 실행**한다.
`mode=basic`인 경우 이 Step을 건너뛴다.

**0-1. 대상 URL 결정**

```bash
# 환경 변수 우선, 없으면 security-agent.config.yml의 target.base_url 사용
STAGING_URL="${STAGING_URL:-$(grep 'base_url' security-agent.config.yml | head -1 | sed 's/.*base_url: *//' | tr -d '\"')}"
echo "대상 URL: $STAGING_URL"
```

**0-2. security-reports 디렉토리 생성**

```bash
mkdir -p ./security-reports
```

**0-3. Docker 및 Nuclei 설치 확인**

```bash
# Docker 확인
docker version --format "Docker: {{.Server.Version}}" 2>/dev/null || echo "ERROR: Docker 미설치 또는 미실행 — ZAP 스킵"

# Nuclei 확인
which nuclei && nuclei --version 2>&1 | head -1 || echo "ERROR: Nuclei 미설치 — Nuclei 스킵"
```

**0-4. ZAP 이미지 확인 및 Pull (필요 시)**

```bash
# 이미지가 없으면 자동 pull
if ! docker images zaproxy/zap-stable --format "{{.Repository}}" | grep -q "zaproxy"; then
  echo "ZAP 이미지 다운로드 중..."
  docker pull zaproxy/zap-stable
fi
```

**0-5. OWASP ZAP 실행 (Passive Scan)**

> **macOS Docker 주의**: Docker 컨테이너는 `localhost`/`127.0.0.1`로 호스트에 접근할 수 없음.
> `localhost` → `host.docker.internal`로 변환 후 ZAP에 전달한다. Linux에서는 변환 불필요.

```bash
# macOS: localhost → host.docker.internal 변환 (ZAP 컨테이너용)
ZAP_TARGET="${STAGING_URL}"
if echo "$ZAP_TARGET" | grep -qE "localhost|127\.0\.0\.1"; then
  ZAP_TARGET=$(echo "$ZAP_TARGET" | sed 's/localhost/host.docker.internal/g' | sed 's/127\.0\.0\.1/host.docker.internal/g')
  echo "ZAP 대상 URL (macOS Docker 변환): $ZAP_TARGET"
fi

docker run --rm \
  -v "$(pwd)/security-reports:/zap/wrk:rw" \
  zaproxy/zap-stable \
  zap-baseline.py \
  -t "$ZAP_TARGET" \
  -r zap-report.html \
  -x zap-report.xml \
  -J zap-report.json || true
```

- 실패해도 `|| true`로 계속 진행한다.
- Active Scan이 필요하면 `zap-baseline.py` → `zap-full-scan.py`로 교체한다.

**0-6. Nuclei 실행**

```bash
# Nuclei는 호스트에서 직접 실행 — localhost 그대로 사용
nuclei -u "$STAGING_URL" \
       -severity critical,high,medium \
       -jsonl \
       -o ./security-reports/nuclei-results.json \
       -silent || true
```

- 실패해도 계속 진행한다.
- 결과 파일이 없으면 Step 1에서 "Step 0 미실행"으로 표시하지 않고 "결과 없음"으로 처리한다.

### Step 1: 설정 파일 로드 + 도구 결과 로드

1. `security-agent.config.yml`을 읽고 필수 필드를 확인한다.
2. 스캔 대상 URL을 다음 우선순위로 결정한다:
   - 1순위: 환경 변수 `STAGING_URL` (설정되어 있으면 우선 사용)
   - 2순위: `security-agent.config.yml`의 `target.base_url` (기본 소스)
   - 어느 쪽도 없으면 즉시 중단하고 `security-agent.config.yml`에 `target.base_url` 설정을 안내한다.
3. 결정된 URL에 프로덕션 URL 패턴 (`prod`, `production`, `app.`)이 포함되지 않았는지 확인 → 포함 시 즉시 중단.
4. **Prompt Injection 검증**:
   - 문자열 필드에 개행 + 명령어 (`ignore`, `system` 등) 포함 여부
   - URL·경로 필드에 셸 메타 문자 (`;`, `&&`, `||` 등) 포함 여부
   - `scope.include` / `scope.exclude`에 `../` 포함 여부
5. ZAP + Nuclei 결과 파일 자동 로드
   - ZAP: `./security-reports/zap-report.html` 및 `./security-reports/zap-report.xml`
   - Nuclei: `./security-reports/nuclei-results.json`
   - 파일이 없으면 Step 0 미실행으로 처리하고 계속 진행
6. 진단 설정 개요를 표시한다. (Python/Django, Rust Axum, Java Spring Security, Node.js Helmet 등 프레임워크별 헤더 설정 반영 여부 확인)

### Step 2: 공격면 매핑

1. `scope.include` / `scope.exclude`를 적용하여 진단 대상 엔드포인트를 확정한다.
2. `openapi.yml` / `openapi.json` / `swagger.json`이 있으면 읽어 보완한다 (FastAPI 자동 생성, Springdoc, Axum + utoipa, Gin Swagger 등).
3. 진단 대상 엔드포인트 수를 보고한다.

### Step 3: HTTP 헤더·TLS 검증

- `curl -I -s {base_url}`로 응답 헤더를 취득하여 아래를 확인한다:

| 헤더 | 기대값 | 결여 시 심각도 |
|------|--------|---------------|
| `Strict-Transport-Security` | `max-age≥31536000` | High |
| `X-Content-Type-Options` | `nosniff` | Medium |
| `X-Frame-Options` | `DENY` 또는 `SAMEORIGIN` | Medium |
| `Content-Security-Policy` | 임의 정책 | Medium |
| `Referrer-Policy` | 설정됨 | Low |
| `X-Powered-By` | **존재하지 않음** | Low (정보 누출) |
| `Server` | 버전 번호 미포함 | Low |

- ZAP/Nuclei 결과와 크로스 체크한다.

**Critical 발견 시**: 즉시 보고하고 나머지 처리를 계속하면서 즉각 대응을 촉구한다.  
(Rust Axum + tower-http, Java Spring Security, Node.js Helmet, Python Django Security Middleware, Go secure headers, Ruby Rack::Protection 등 프레임워크별 미들웨어 설정 누락 여부 강조)

### Step 4: 동적 진단 (Claude 자체 실행)

`agents` 리스트에 설정된 에이전트를 순서대로 실행한다. 각 에이전트 출력은 8,000자 이내로 필터링 후 처리한다.

**실행 에이전트 목록:**

- **owasp_top10** (항상 실행):
  - **A01:2025 Broken Access Control**: 인증 토큰 치환 후 타 사용자 리소스 접근 가능 여부, IDOR, 권한 상승 공격 (모든 언어의 세션/JWT/인가 로직 테스트)
  - **A02:2025 Security Misconfiguration**: HTTPS 강제 여부, 보안 헤더 설정 확인, 디버그 모드 활성화 여부 (Python Django debug=True, Java Spring, Node.js, Rust Axum 등)
  - **A04:2025 Cryptographic Failures**: 약한 암호화 알고리즘 사용, TLS 설정, JWT 서명 검증 (Java, Rust ring/rsa, Node.js crypto, Python cryptography 등)
  - **A05:2025 Injection**: 입력 필드에 SQLi / CMDi / NoSQLi / LDAPi 페이로드 전송, API 응답에서 에러 메시지·스택트레이스·쿼리 정보 노출 여부 확인 (Python SQLAlchemy/ORM, Rust sqlx, Java JDBC/Hibernate, Go database/sql, Node.js, Ruby ActiveRecord 등)
  - **A06:2025 Insecure Design** 또는 **A07:2025 Identification and Authentication Failures**: JWT `alg:none` 공격, 세션 고정 공격, 토큰 만료·재사용 체크, 약한 인증 플로우
  - **A08:2025 Software and Data Integrity Failures**: 서명되지 않은 업데이트·직렬화 객체 등 무결성 검증 누락
  - **A09:2025 Security Logging and Monitoring Failures**: 로그에 민감 정보 노출, 에러 메시지 상세 노출·디버그 정보 누출 확인
  - **A10:2025 Mishandling of Exceptional Conditions** (또는 SSRF 관련): SSRF 패턴 테스트 (169.254.169.254, 메타데이터 엔드포인트), 예외 처리 실패로 인한 정보 누출 또는 fail-open 상황

- **auth_bypass** (설정된 경우) — A06/A07의 심화 실행:
  - JWT 알고리즘 혼란 공격 (alg confusion, none 허용)
  - OAuth state 파라미터 위조 검증
  - Cookie `HttpOnly` / `Secure` / `SameSite` 속성 확인 (모든 언어)

- **injection** (설정된 경우) — A05의 심화 실행:
  - SQL 주입: 시간 기반 맹목적, 오류 기반 등 고급 기법
  - NoSQL 주입 (`$where`, `$regex`, `$gt`)
  - 명령 주입 (`; ls`, `&&id` 등)

- **prompt_injection** (설정된 경우):
  - Direct Injection: 시스템 프롬프트 재정의 시도
  - Indirect Injection: 외부 데이터를 통한 지침 포함 테스트

- **multi_tenant** (설정된 경우):
  - 다른 임차인 ID에 대한 직접 액세스 시도
  - 응답 데이터에 다른 테넌트 데이터가 포함되어 있지 않은지 확인

- **file_exposure** (설정된 경우):
  - 경로 탐색 공격 (`../../../etc/passwd`)
  - 공개 URL 추측 (연번·UUID 패턴)

### Step 4-B: 3중 결과 통합

1. **ZAP 결과 분석**: Alert, Evidence, Passive/Active Scan 결과 해석

2. **Nuclei 결과 분석**: 매칭된 템플릿과 severity 기반 분석

3. **교차 검증 및 종합**:
   - ZAP + Nuclei + Step 4 Claude 자체 테스트 결과를 교차 검증
   - 동일 취약점이 여러 도구에서 발견된 경우 High Priority 승격
   - False Positive 필터링
   - 기술 스택별 해석 강화 (Python, Rust, Go, Java, Node.js, Ruby)

### Step 5: 커버리지 집계

- `mode=basic` → `scan_type: runtime` / `mode=augmented` (기본값) → `scan_type: runtime_augmented`

```yaml
scan_date: <ISO8601>
target: <base_url>
scan_type: <runtime | runtime_augmented>

attack_surface:
  endpoints_total: <총수>
  endpoints_tested: <테스트 완료 수>
  endpoints_skipped:
  - path: <경로>, reason: <이유>

vuln_classes:
  owasp_top10:
    covered: <N>/10
    gaps:
    - A03:2025 Software Supply Chain Failures — 런타임 테스트 불가, /security-full-scan(cargo audit) 사용
    - 기타 커버하지 못한 항목과 그 이유

tools:
  zap: <실행됨 | Step 0 미실행>
  nuclei: <실행됨 | Step 0 미실행>

findings:
  critical: <건수>
  high: <건수>
  medium: <건수>
  low: <건수>

not_covered_by_this_scan:
  - 의존성 알려진 CVE → /security-full-scan 사용
  - 소스코드 정적 분석 → /security-full-scan 사용
  - 비즈니스 로직 결함 → 침투 테스트 필요
  - 인프라·클라우드 설정 → 인프라 담당자 리뷰 필요

coverage_score: <퍼센트>%
ci_result: <✅ pass | ❌ fail>
```

### Step 6: 보고서 생성 및 기록 인덱스 업데이트

1. **날짜 파일 이름으로 보고서 저장**
   - 보고서: `./security-reports/YYYY-MM-DD-security-scan-report.md`
   - 커버리지: `./security-reports/YYYY-MM-DD-security-scan-coverage.yml`
   - 날짜는 런타임 ISO 8601 형식(예: `2026-05-01`)
   - `report.output_path`가 명시되어있는 경우는 그쪽을 우선한다
   - 디렉토리가 존재하지 않는 경우 자동생성

2. ZAP Findings + Nuclei Findings + Claude 자체 분석 + 3중 종합 의견 포함

3. 심각도별로 정렬하고 수정 권장 사항 포함

4. **`./security-reports/index.md` 에 1행 추가한다**
   - 파일이 없으면 아래 공통 헤더부터 생성한다:
   ```
   | 날짜 | 스캔 유형 | 대상 | Critical | High | Medium | CI 결과 | 리포트 |
   |------|-----------|------|----------|------|--------|---------|--------|
   ```
   - 추가 형식:
   ```
   | YYYY-MM-DD | security-scan | <base_url> | <Critical 건수> | <High 건수> | <Medium 건수> | <CI 결과> | [보고서](YYYY-MM-DD-security-scan-report.md) |
   ```
   - CI 결과: config의 `severity_gate` (기본값: `high`) 이상 발견이 0건이라면 `✅ pass`, 1건 이상이라면 `❌ fail`

5. Critical 발견이 있는 경우, 종료 코드 1을 보고한다.

## 이 스킬로 커버할 수 없는 영역

| 영역 | 추천 수단 |
|------|---------|
| 의존성 알려진 CVE | `/security-full-scan` |
| 소스코드 정적 분석 | `/security-full-scan` |
| 비즈니스 로직 결함 | 침투 테스트 |
| 인프라·클라우드 설정 | 인프라 담당자 리뷰 |

## 완료 조건

- [ ] 보안 보고서가 생성되어 있다 (`report.output_path` 설정 시 해당 경로, 미설정 시 `./security-reports/YYYY-MM-DD-security-scan-report.md`)
- [ ] `./security-reports/YYYY-MM-DD-security-scan-coverage.yml`이 생성되어 있다 (`scan_type: runtime 또는 runtime_augmented` 명기)
- [ ] 진단된 OWASP 카테고리 수가 보고되어 있다.
- [ ] CI 결과 (Critical 발견 0건 → ✅ pass, 1건 이상 → ❌ fail) 가 명시되어 있다.
- [ ] **커버할 수 없는 영역**이 명시되어 있다.
- [ ] `./security-reports/index.md`에 한 줄 추가되어 있다.
- [ ] ZAP·Nuclei 결과 로드 여부가 보고서에 명시되어 있다 (없으면 "Step 0 미실행"으로 기재)


