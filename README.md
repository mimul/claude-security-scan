Claude Code용 보안 스킬 2종

## 스킬의 구분

| 타이밍 | 스킬 | 대상 |
|-----------|--------|------|
| 출시 전 | `/security-full-scan` | 모든 파일 정적 분석 + 종속성 CVE |
| 배포 후 | `/security-scan` | 런타임 동작 및 HTTP 동작 테스트 |

## 설치

2026/04/16, GitHub 공식 CLI gh에 기술을 패키지 관리하는 새로운 부속 명령이 gh skill 이 추가 되었다. GitHub 리포지토리에 게시된 기술을 gh 를 통해 설치, 업데이트 및 게시할 수 있다. 

```bash
> gh skill install mimul/claude-security-scan

> brew install trivy semgrep nuclei  # 사용할 보안 도구 설치
> brew install --cask zap.           # OWASP ZAP 설치
```

## 업데이트

```bash
> gh skill update --dry-run  # 무엇이 업데이트되는지 먼저 확인
> gh skill update            # 업데이트
```

## 스킬 개요

### `/security-full-scan` — 모든 파일 정적 분석 (언어 및 프레임 워크 독립적)

Claude가 프로젝트 구조를 자동으로 감지하고 모든 소스 파일의 정적 분석과 종속성 CVE 스캔을 병렬로 실행하고 Semgrep이 정적 분석, Trivy으로 의존성 CVE, 시크릿 누출, Misconfiguration, SBOM 분석 등을 진행한다.

**특징**

- Claude 소스 분석 : 전체 소스 파일의 정적 분석, 의존성 알려진 CVE 스캔 + 시크릿 누출 체크
- Semgrep: 정적 코드 분석 (SAST), 깊이 있는 코드 레벨 취약점 탐지
- Trivy: 의존성 CVE, 시크릿 누출, Misconfiguration, SBOM 분석

### `/security-scan` — 런타임 검증

Claude가 스테이징 환경에 배치된 서버에 대해 HTTP 동적 테스트를 수행하고 OWASP ZAP이 웹 애플리케이션 스캐닝을 하고 Nuclei가 빠르고 정확한 템플릿 기반 취약점 스캔을 진행한다.

**특징**

- Claude 엔드포인트별로 HTTP 보안 헤더(HSTS, CSP, X-Frame-Options 등), OWASP Top 10(A01~A10), JWT 알고리즘 혼란 공격 · 쿠키 속성, SQL / NoSQL / 명령 주입, 프롬프트 인젝션(AI 기능이 있는 경우)
- OWASP ZAP: 종합 웹 애플리케이션 스캐닝 (Passive Scan, AJAX 크롤링, 헤더 분석)
- Nuclei: 빠르고 정확한 템플릿 기반 취약점 스캔 (수천 개의 최신 취약점 템플릿)

**전제 :** `security-agent.config.yml`과`STAGING_URL`환경 변수 필요

### 각 스킬이 커버할 수 없는 영역

| 영역 | 추천 수단 |
|------|---------|
| 아키텍처/비즈니스 로직 수준 취약점 | [소프트웨어 보안의 철학, 전략, 원칙](https://www.mimul.com/blog/ai-security-rule/)을 기반으로 해당 프로젝트에서 사용하는 언어에 특성을 가미한 <br/>[security-style.md](https://github.com/mimul/axum-rusty/blob/main/.claude/rules/rust-security-style.md) 파일을 rules에 적용해서 코드 리뷰, 리팩토링에서 활용해서 보완 필요|
| 비즈니스 로직 결함 | 침투 테스트 |
| 인프라 및 클라우드 설정 (IAM 등) | 인프라 담당자 검토 |
| 제로 데이 취약점 | CVE 데이터베이스 외부로 인해 검색 불가 |

### 실행 방법

```
/security-full-scan /project --mode=augmented
/security-scan staging --mode=augmented
```

`/security-scan` 은 `security-agent.config.yml` 설정을 카피해서 필요한 정보는 수정해야 한다.

## 디렉토리 구성

```
claude-security-skills/
├── skills/
│   ├── security-full-scan.md       # /security-full-scan 스킬
│   └── security-scan.md            # /security-scan 스킬
└── templates/
    └── security-agent.config.yml   # /security-scan 설정 파일
```
