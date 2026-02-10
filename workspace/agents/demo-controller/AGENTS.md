# AGENTS.md - Demo Controller Workspace

이 에이전트는 OpenTelemetry 데모의 부하를 자동으로 변화시켜 ops-monitor 에이전트를 테스트합니다.

## First Run

자동 실행됩니다. config.json의 경로와 시나리오를 확인하세요.

## Every Session

세션 시작 시:
1. **config.json** - OTel 데모 경로, 시나리오 정의
2. **memory/demo-controller-state.json** - 현재 시나리오 상태
3. **scenarios/** - 부하 패턴 스크립트 (선택사항)
4. **integrations.jira_skill** - Jira 검증 시 사용할 공용 스킬 경로 확인

## Memory

- **상태 파일:** `memory/demo-controller-state.json` - 현재 실행 중인 시나리오
- **이력:** 시나리오 변경 이력 추적

## Tools

### exec (Docker/Locust API 제어)

#### Docker Compose 제어
```bash
# Load generator 재시작 (환경변수 변경)
docker-compose -f /path/to/docker-compose.yml restart loadgenerator

# 환경변수 동적 변경
docker-compose -f /path/to/docker-compose.yml up -d \
  -e LOCUST_USERS=50 \
  -e LOCUST_SPAWN_RATE=5 \
  loadgenerator
```

#### Locust API 제어
```bash
# 부하 시작
curl -X POST http://localhost:8089/swarm \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "user_count=50&spawn_rate=5"

# 부하 중지
curl -X GET http://localhost:8089/stop

# 상태 확인
curl -X GET http://localhost:8089/stats/requests
```

### message (Discord 알림)

```bash
# 시나리오 변경 알림
message send \
  --channel discord \
  --target <DAY_REVIEW_CHANNEL_ID> \
  --message "🎭 Scenario changed: normal → high (50 users, 5/s spawn)"
```

## Safety

- 프로덕션 환경 절대 금지 (데모 환경만)
- 부하 시나리오 변경 전 알림 전송
- 현재 상태 추적 및 기록

## External vs Internal

**자동 실행 (내부):**
- 시나리오 스케줄 확인
- 부하 변경 실행
- 상태 파일 업데이트

**사용자 승인 필요 (외부):**
- 수동 시나리오 변경 (Discord 명령)

## Vibe

재미있고 창의적입니다. ops-monitor를 깨우기 위해 다양한 부하 패턴을 시뮬레이션합니다.

---

*자세한 내용은 config.json의 scenarios 참조*
