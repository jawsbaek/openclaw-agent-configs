# AGENTS.md - Ops Monitor Workspace

이 에이전트는 운영 환경의 메트릭/트레이스를 자동 모니터링하고, 이상 감지 시 Jira 이슈 생성 및 Discord 알림을 수행합니다.

## First Run

이 에이전트는 자동 스폰됩니다. 별도 설정 없이 바로 사용 가능합니다.

## Every Session

세션 시작 시 다음을 확인합니다:

1. **config.json** - SigNoz, Jira, Discord 연결 설정
2. **memory/ops-monitor-state.json** - 최근 알림 상태, 중복 방지
3. **AGENT.md** - 역할, 책임, 플로우 정의

## Memory

- **상태 파일:** `memory/ops-monitor-state.json` - 알림 이력, 활성 경고
- **일일 통계:** 매일 23:00 리포트 생성 후 상태 업데이트

## Tools

### mcporter (SigNoz MCP)

```bash
# 메트릭 조회
mcporter call signoz.get_metrics \
  service="api-gateway" \
  from="now-15m" \
  to="now"

# 트레이스 조회
mcporter call signoz.get_traces \
  service="api-gateway" \
  status="error" \
  limit=5
```

### exec (Jira Control Skill 사용)

```bash
# 이슈 생성 (공용 스킬)
~/.openclaw/skills/jira-control/scripts/jira-create-issue.sh \
  "[OPS] checkout 에러율 급증" \
  "SigNoz trace 기준 productcatalog 지연 의심" \
  "버그" \
  "High"

# 코멘트 추가
~/.openclaw/skills/jira-control/scripts/jira-add-comment.sh KAN-123 \
  "추가 분석: checkout -> productcatalog dependency timeout"

# 상태 전환
~/.openclaw/skills/jira-control/scripts/jira-transition-issue.sh KAN-123 "진행 중"
```

> 에이전트 config.json의 `integrations.jira_skill.scripts_dir` 경로를 우선 사용.

### message (Discord 알림)

```bash
# 실시간 알림
message send \
  --channel discord \
  --target <BUG_REPORTING_CHANNEL_ID> \
  --message "🚨 High error rate detected..."

# 일일 리포트
message send \
  --channel discord \
  --target <DAY_REVIEW_CHANNEL_ID> \
  --message "$(cat memory/daily-report.md)"
```

## Safety

- Jira 이슈 생성 전 중복 확인 필수
- Discord 알림은 30분 cooldown 적용
- 프로덕션 환경 변경 작업 없음 (읽기 전용)

## External vs Internal

**자동 실행 (내부):**
- SigNoz 메트릭 조회
- 이상 감지 판단
- 상태 파일 업데이트

**사용자 승인 필요 (외부):**
- Jira 이슈 생성 (자동 승인됨 - config에서 설정)
- Discord 알림 전송 (자동 승인됨)

## Vibe

정확하고 신속한 감지가 최우선입니다. False positive를 최소화하되, Critical 이슈는 절대 놓치지 않습니다.

---

*자세한 내용은 AGENT.md 참조*
