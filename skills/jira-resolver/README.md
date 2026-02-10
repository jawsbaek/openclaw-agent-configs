# Jira Resolver Skill

Jira 티켓 기반 장애 자동 대응을 위한 스킬 모음

## 스크립트 목록

### 1. check-signoz-metrics.sh
SigNoz 메트릭을 ClickHouse에서 직접 조회

**사용법:**
```bash
./scripts/check-signoz-metrics.sh [interval_minutes] [error_threshold]

# 예시: 최근 10분간 에러율 5% 이상 서비스 조회
./scripts/check-signoz-metrics.sh 10 5.0
```

**출력:**
- JSON Lines 형식 (serviceName, total_calls, error_calls, error_rate_pct)
- 임계값 초과 시 stderr로 경고 메시지

**의존성:**
- Docker (signoz-clickhouse 컨테이너)
- jq

---

### 2. auto-restart-service.sh
Docker 서비스 자동 재시작 및 Jira 코멘트 추가

**사용법:**
```bash
./scripts/auto-restart-service.sh <service_name> [jira_issue_key]

# 예시
./scripts/auto-restart-service.sh productcatalog KAN-18
./scripts/auto-restart-service.sh flagd KAN-16
```

**지원 서비스:**
- productcatalog / product-catalog
- checkout
- frontend / frontend-proxy
- flagd
- load-generator
- ad, recommendation, fraud-detection, product-reviews

**동작:**
1. Docker 컨테이너 상태 확인
2. `docker restart` 실행
3. 재시작 후 상태 확인
4. (옵션) Jira 이슈에 자동 코멘트 추가

**의존성:**
- Docker
- jira-control 스킬

---

### 3. cleanup-duplicate-issues.sh
중복 Jira 이슈 자동 정리

**사용법:**
```bash
./scripts/cleanup-duplicate-issues.sh [dry_run]

# Dry run (실제 종료 안 함)
./scripts/cleanup-duplicate-issues.sh true

# 실제 실행
./scripts/cleanup-duplicate-issues.sh false
```

**중복 패턴:**
- flagd EventStream 관련
- ProductCatalog 연동 실패
- Checkout 장애
- Frontend 에러
- Load Generator 크래시

**동작:**
1. 미완료 이슈 조회
2. 패턴별 중복 그룹 탐지
3. 가장 최신 이슈 1개만 유지
4. 나머지 이슈에 코멘트 추가 후 "완료" 전환

**의존성:**
- jq
- jira-control 스킬
- 환경변수: JIRA_SITE, JIRA_EMAIL, JIRA_API_TOKEN, JIRA_PROJECT_KEY

---

## 환경 설정

### 필수 환경변수
```bash
export JIRA_SITE="your-domain.atlassian.net"
export JIRA_EMAIL="your-email@example.com"
export JIRA_API_TOKEN="***"
export JIRA_PROJECT_KEY="KAN"
```

### Docker 서비스
- signoz-clickhouse: ClickHouse 데이터베이스
- product-catalog, checkout, frontend 등: OpenTelemetry Demo 서비스

---

## 크론잡 통합

### 5분 주기: 실시간 모니터링 및 자동 조치
```bash
~/.openclaw/skills/jira-resolver/scripts/check-signoz-metrics.sh 5 10.0
# 에러율 10% 이상 시 자동 재시작
```

### 30분 주기: 중복 이슈 정리
```bash
~/.openclaw/skills/jira-resolver/scripts/cleanup-duplicate-issues.sh false
```

### 일일 23:00: 일일 리포트
```bash
~/.openclaw/skills/jira-resolver/scripts/check-signoz-metrics.sh 1440 1.0
# 24시간 에러율 1% 이상 서비스 리포트
```

---

## 작동 원리

### SigNoz 메트릭 조회
```sql
SELECT serviceName, count() as total_calls,
       countIf(statusCode=2) as error_calls,
       round(error_calls * 100.0 / total_calls, 2) as error_rate_pct
FROM signoz_traces.distributed_signoz_index_v3
WHERE timestamp >= now() - INTERVAL 10 MINUTE
GROUP BY serviceName
ORDER BY error_rate_pct DESC
```

- `statusCode=2`: 에러 상태
- `distributed_signoz_index_v3`: SigNoz v3 트레이스 인덱스

### Docker 서비스 재시작
```bash
docker restart <container-name>
```

### Jira 워크플로우
```
해야 할 일 → 진행 중 → 검토 중 → 완료
```

---

## 버전
- v1.0.0 (2026-02-10): 초기 버전
  - SigNoz 메트릭 조회
  - Docker 자동 재시작
  - 중복 이슈 정리
