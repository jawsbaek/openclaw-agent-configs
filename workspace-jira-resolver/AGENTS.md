# AGENTS.md - Jira Resolver Workspace

이 워크스페이스는 **Jira 티켓 기반 장애 대응**에 최적화되어 있습니다.

## First Run Checklist

1. `SOUL.md` 읽기
2. `USER.md` 읽기
3. `IDENTITY.md` 확인
4. `MEMORY.md` 없으면 생성 (현재 생성 완료)
5. `memory/YYYY-MM-DD.md` 생성
6. `BOOTSTRAP.md`는 초기 설정이 끝났으면 보관만 하고 재실행하지 않기

---

## Every Session

세션 시작 즉시:

1. `SOUL.md`
2. `USER.md`
3. `memory/오늘.md`, `memory/어제.md`
4. 메인 세션이면 `MEMORY.md`
5. `HEARTBEAT.md` (heartbeat poll 수신 시)

---

## Memory Rules

- 단기: `memory/YYYY-MM-DD.md`
- 장기: `MEMORY.md`
- “기억해줘” 요청은 반드시 파일에 기록
- Jira 운영에서 재발 패턴/실패 원인은 장기 메모리로 승격

---

## Jira 운영 규칙

- 상태 전환: 해야 할 일 → 진행 중 → 검토 중 → 완료
- 코멘트: **한국어 + 근거(메트릭/로그/트레이스)**
- 자동 조치 실패 시: 원인/다음 액션/수동 담당 명시
- 파괴적 명령 금지 (`rm`, `down`, 데이터 삭제 계열)

공용 스킬 경로:

**Jira 제어 (jira-control):**
- `~/.openclaw/skills/jira-control/scripts/jira-create-issue.sh`
- `~/.openclaw/skills/jira-control/scripts/jira-add-comment.sh`
- `~/.openclaw/skills/jira-control/scripts/jira-transition-issue.sh`

**자동 복구 (jira-resolver):**
- `~/.openclaw/skills/jira-resolver/scripts/check-signoz-metrics.sh` - SigNoz 메트릭 조회
- `~/.openclaw/skills/jira-resolver/scripts/auto-restart-service.sh` - Docker 서비스 재시작
- `~/.openclaw/skills/jira-resolver/scripts/cleanup-duplicate-issues.sh` - 중복 이슈 정리

---

## 인프라 환경

### Docker 기반 서비스
- **플랫폼**: Docker Compose (Kubernetes 아님)
- **주요 서비스**: product-catalog, checkout, frontend, frontend-proxy, flagd, load-generator, ad, recommendation, fraud-detection, product-reviews
- **모니터링**: signoz, signoz-clickhouse, signoz-otel-collector, signoz-mcp-server

### SigNoz 연결
- **Web UI**: http://localhost:3301 (로그인 필요)
- **ClickHouse**: signoz-clickhouse 컨테이너 (docker exec 사용)
- **MCP Server**: http://localhost:8001/mcp (세션 관리 복잡)
- **권장 방법**: ClickHouse 직접 쿼리 (`signoz_traces.distributed_signoz_index_v3`)

### 메트릭 조회 예시
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
- `statusCode=0`: 정상 상태

---

## Heartbeat vs Cron

- Heartbeat: 가벼운 상태 확인(큐 급증/정체 감지)
- Cron: 정기 처리/일일 리포트/정확한 시간 작업

`CRON_PROMPTS.md`의 템플릿을 우선 사용.

---

## Recommended Cron Jobs

1. 5분 주기 처리 (`jira-resolver-realtime`)
2. 30분 정체 점검 (`jira-resolver-stuck-check`)
3. 23:00 일일 요약 (`jira-resolver-daily-summary`)

---

## 자동 조치 워크플로우

### 1. 메트릭 기반 장애 탐지
```bash
~/.openclaw/skills/jira-resolver/scripts/check-signoz-metrics.sh [interval] [threshold]
```
- 임계값 초과 서비스 감지
- JSON Lines 형식 출력

### 2. 자동 재시작 (안전한 조치)
```bash
~/.openclaw/skills/jira-resolver/scripts/auto-restart-service.sh <service> [jira_issue]
```
- Docker 컨테이너 재시작
- Jira 이슈에 자동 코멘트 추가
- 재시작 후 상태 확인

### 3. 검증 및 이슈 종료
```bash
# 5-10분 후 메트릭 재확인
check-signoz-metrics.sh 10 1.0

# 에러율 정상 범위(1% 미만) 확인되면
jira-transition-issue.sh KAN-XXX "완료"
```

### 4. 중복 이슈 정리 (주기적)
```bash
~/.openclaw/skills/jira-resolver/scripts/cleanup-duplicate-issues.sh false
```

### 안전 기준
- **자동 재시작 허용**: 상태 비저장 서비스 (product-catalog, flagd, load-generator 등)
- **수동 조치 필요**: 데이터베이스, 메시지 큐, 영구 저장소
- **파괴적 명령 금지**: `rm`, `docker-compose down`, 데이터 삭제

---

## Safety

- 외부 전송(메시지/공개 채널)은 목적/수신처 확인 후 실행
- 불명확하면 실행보다 질문
- 실수/장애는 숨기지 말고 메모리와 코멘트에 기록
