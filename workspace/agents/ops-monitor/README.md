# Ops Monitor Agent 🚨

SigNoz 메트릭 기반 자동 이슈 생성 및 리포팅 시스템

---

## 📋 개요

이 에이전트는:
- ✅ **SigNoz 메트릭/트레이스** 실시간 모니터링 (5분마다)
- ✅ **Jira 이슈** 자동 생성 (이상 감지 시)
- ✅ **Discord 실시간 알림** (bug-reporting 채널)
- ✅ **일일 운영 리포트** 생성 (매일 23:00 KST)

---

## 🚀 빠른 시작

### 1. 의존성 체크

```bash
./scripts/ops-monitor.sh check-deps
```

필요한 것들:
- ✅ `mcporter` - SigNoz MCP 통신
- ✅ `jq` - JSON 처리 (선택사항)
- ✅ `JIRA_API_TOKEN` 환경변수

---

### 2. SigNoz MCP 설정

```bash
# MCP 서버 추가
mcporter config add signoz \
  --url https://your-signoz-instance.com/api/mcp

# 인증
mcporter auth signoz

# 테스트
mcporter call signoz.health

# 연결 확인
./scripts/ops-monitor.sh check-signoz
```

---

### 3. Jira API 토큰 발급

1. Jira 로그인
2. https://id.atlassian.com/manage-profile/security/api-tokens
3. "Create API token" 클릭
4. 환경변수 설정:

```bash
# ~/.zshrc 또는 ~/.bashrc에 추가
export JIRA_API_TOKEN="your_token_here"

# 또는 OpenClaw gateway config에 추가
openclaw gateway config.patch \
  --raw '{"env": {"JIRA_API_TOKEN": "your_token_here"}}'
```

---

### 4. 설정 파일 수정

`agents/ops-monitor/config.json` 편집:

```json
{
  "signoz": {
    "url": "https://your-signoz-instance.com",  // ← 실제 URL로 변경
    "dashboard_base_url": "https://your-signoz-instance.com/dashboard"
  },
  "jira": {
    "url": "https://your-domain.atlassian.net",  // ← 실제 도메인으로 변경
    "project_key": "OPS"  // ← 프로젝트 키 확인
  },
  "discord": {
    "mentions": {
      "oncall_role_id": "ACTUAL_ROLE_ID"  // ← Discord oncall role ID
    }
  }
}
```

---

### 5. Discord 채널 확인

```bash
./scripts/ops-monitor.sh check-discord
```

채널 ID 확인:
- **Bug reporting:** `<BUG_REPORTING_CHANNEL_ID>`
- **Day review:** `<DAY_REVIEW_CHANNEL_ID>`

---

### 6. 테스트 실행 (Dry Run)

```bash
# 실시간 모니터링 테스트
./scripts/ops-monitor.sh test-realtime

# 일일 리포트 테스트
./scripts/ops-monitor.sh test-daily
```

결과 확인:
```bash
openclaw sessions list --kinds isolated
```

---

### 7. Cron Job 설정

```bash
# Cron job 생성
./scripts/ops-monitor.sh setup-cron

# 확인
openclaw cron list

# 수동 실행 (테스트)
openclaw cron run ops-monitor-realtime
openclaw cron run ops-monitor-daily
```

---

## 📊 사용법

### 현재 상태 확인

```bash
./scripts/ops-monitor.sh status
```

### 수동 실행

```bash
# 실시간 체크
openclaw sessions spawn \
  --agent ops-monitor \
  --task "SigNoz 메트릭 체크 및 이상 감지"

# 일일 리포트
openclaw sessions spawn \
  --agent ops-monitor \
  --task "오늘의 운영 리포트 생성"
```

### Cron Job 관리

```bash
# 목록
openclaw cron list

# 수동 트리거
openclaw cron run ops-monitor-realtime
openclaw cron run ops-monitor-daily

# 제거
./scripts/ops-monitor.sh remove-cron
```

---

## 🔧 튜닝

### 임계값 조정

`config.json` 편집:

```json
{
  "thresholds": {
    "error_rate_percent": 5,           // 에러율 임계값 (%)
    "response_time_p95_ms": 2000,      // 응답시간 p95 (ms)
    "window_minutes": 15,              // 감지 윈도우 (분)
    "alert_cooldown_minutes": 30       // 중복 알림 방지 (분)
  }
}
```

### False Positive 줄이기

- `window_minutes` 증가 (15분 → 30분)
- `error_rate_percent` 상향 조정 (5% → 10%)
- `alert_cooldown_minutes` 증가 (30분 → 60분)

---

## 📁 구조

```
agents/ops-monitor/
├── README.md                       # 이 파일
├── AGENT.md                        # 에이전트 정의
├── config.json                     # 설정
├── templates/
│   ├── jira-issue.md              # Jira 이슈 본문
│   ├── jira-issue-payload.json    # Jira API 페이로드
│   ├── bug-alert.md               # Discord 실시간 알림
│   └── daily-report.md            # Discord 일일 리포트
└── memory/
    └── ops-monitor-state.json     # 상태 추적
```

---

## 🐛 트러블슈팅

### SigNoz 연결 실패

```bash
# MCP 서버 목록 확인
mcporter list

# 재인증
mcporter auth signoz --reset

# 직접 호출 테스트
mcporter call signoz.get_metrics service=api-gateway
```

### Jira 이슈 생성 실패

```bash
# API 토큰 확인
echo $JIRA_API_TOKEN

# 프로젝트 키 확인
curl -H "Authorization: Bearer $JIRA_API_TOKEN" \
  https://your-domain.atlassian.net/rest/api/3/project

# 이슈 타입 확인
curl -H "Authorization: Bearer $JIRA_API_TOKEN" \
  https://your-domain.atlassian.net/rest/api/3/issuetype
```

### Discord 알림 전송 실패

```bash
# OpenClaw Discord 설정 확인
openclaw gateway config.get | jq '.channels.discord'

# 채널 권한 확인 (Discord 설정에서)
# - "Send Messages" 권한 필요
# - 봇이 채널에 접근 가능한지 확인
```

---

## 📚 참고 문서

- **설계 문서:** `docs/setup/ops-monitor-agent-design.md`
- **에이전트 정의:** `agents/ops-monitor/AGENT.md`
- **SigNoz API:** https://signoz.io/docs/api/
- **Jira REST API:** https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- **mcporter 가이드:** http://mcporter.dev

---

## 🎯 로드맵

### Phase 1: 기본 모니터링 ✅
- [x] SigNoz MCP 통합
- [x] Jira 이슈 자동 생성
- [x] Discord 실시간 알림
- [x] 일일 리포트

### Phase 2: 고도화 (TODO)
- [ ] 자동 복구 (Auto-remediation)
- [ ] 머신러닝 기반 이상 감지
- [ ] 다중 서비스 지원
- [ ] Grafana 대시보드 연동

### Phase 3: 확장 (TODO)
- [ ] PagerDuty 통합
- [ ] Slack 알림
- [ ] Datadog 메트릭
- [ ] Incident response automation

---

**질문이나 개선사항은 Discord로!** 🚀
