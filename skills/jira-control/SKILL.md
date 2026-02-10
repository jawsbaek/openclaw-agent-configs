---
name: jira-control
description: Jira 이슈 생성/조회/코멘트/상태 전환을 위한 로컬 스킬. Ops 모니터링 및 수동 이슈 제어에 사용.
---

# Jira Control Skill

Jira Cloud REST API를 사용해 이슈를 제어합니다.

## Prerequisites

환경변수 설정:

```bash
export JIRA_SITE="your-domain.atlassian.net"
export JIRA_EMAIL="your-email@example.com"
export JIRA_API_TOKEN="AT..."
# 기본 프로젝트 키(선택)
export JIRA_PROJECT_KEY="OPS"
```

## Scripts

- `scripts/jira-create-issue.sh` : 이슈 생성
- `scripts/jira-add-comment.sh` : 코멘트 추가
- `scripts/jira-transition-issue.sh` : 상태 전환(To Do/In Progress/Done 등)

## Quick Examples

```bash
# 이슈 생성
~/.openclaw/skills/jira-control/scripts/jira-create-issue.sh \
  "Checkout 5xx 급증" \
  "13:10 KST 이후 checkout 서비스 5xx 에러율 급증. trace 기준 productcatalog 연동 지연 의심" \
  Bug \
  High

# 코멘트 추가
~/.openclaw/skills/jira-control/scripts/jira-add-comment.sh OPS-123 \
  "SigNoz trace 확인 결과, root cause는 productcatalog timeout으로 추정됩니다."

# 상태 전환 (이름 기반)
~/.openclaw/skills/jira-control/scripts/jira-transition-issue.sh OPS-123 "In Progress"
~/.openclaw/skills/jira-control/scripts/jira-transition-issue.sh OPS-123 "Done"
```

## Notes

- Jira Cloud 기준입니다.
- 토큰/개인정보는 스크립트에 하드코딩하지 마세요.
- 이슈 생성/코멘트 내용은 한국어로 작성 권장(ops-monitor 규칙).
