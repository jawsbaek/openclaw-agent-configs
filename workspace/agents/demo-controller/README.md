# Demo Controller Agent

OpenTelemetry 데모의 부하를 자동으로 제어하여 ops-monitor 에이전트를 테스트하는 에이전트입니다.

## 🚀 Quick Start

### 1. OpenTelemetry Demo 실행 확인

```bash
cd /path/to/ops-demo
docker compose ps
```

**주요 엔드포인트:**
- Frontend: http://localhost:8080
- Locust (Load Generator): http://localhost:65215
- SigNoz: http://localhost:3301
- FlagD HTTP: http://localhost:65156
- FlagD gRPC: localhost:65157

### 2. 수동 테스트

#### Locust 부하 변경
```bash
cd ~/.openclaw/workspace/agents/demo-controller/scenarios

# 50 users, 5/s spawn
./change-load.sh 50 5

# 통계 확인
./get-stats.sh
```

#### FlagD 부하 생성
```bash
# 10 RPS, 60초
./flagd-load.sh 10 60

# 100 RPS, 10분 (600초)
./flagd-load.sh 100 600

# 백그라운드 실행
./flagd-load.sh 100 600 &
```

### 3. 자동 스케줄

**매시 정각 (Locust 시나리오)**:
- 09:00: moderate (20 users)
- 12:00: high (50 users) ← ops-monitor warning 트리거
- 15:00: moderate (20 users)
- 18:00: normal (5 users)
- 22:00: recovery (2 users)

**15분마다 (FlagD 시나리오)**:
- 00:00-09:00: 10 RPS
- 09:00-12:00: 10 RPS
- 12:00-15:00: 100 RPS ← flagd 부하 증가
- 15:00-22:00: 10 RPS

---

## 🎯 시나리오 목적

### Locust (전체 애플리케이션)

| 시나리오 | Users | Spawn Rate | 목적 |
|---------|-------|-----------|------|
| Normal | 5 | 1/s | Baseline |
| Moderate | 20 | 2/s | 일반 트래픽 |
| High ⚠️ | 50 | 5/s | ops-monitor warning |
| Stress 🚨 | 100 | 10/s | Critical + Jira |
| Spike ⚡ | 80 | 20/s | 순간 부하 감지 |
| Recovery ✅ | 2 | 1/s | 알림 해제 |

### FlagD (Feature Flag 서비스)

| 시나리오 | RPS | Duration | 목적 |
|---------|-----|----------|------|
| FlagD Normal | 10 | 10min | Baseline |
| FlagD High | 100 | 10min | 응답 시간 증가 |
| FlagD Stress | 500 | 5min | 임계값 초과 |

---

## 🔧 수동 제어

### Locust UI 접속

```bash
open http://localhost:65215
```

**Locust Web UI에서:**
1. Users: 50
2. Spawn rate: 5
3. Host: http://frontend-proxy:8080
4. Start Swarming

### FlagD 부하 스크립트

```bash
cd ~/.openclaw/workspace/agents/demo-controller/scenarios

# HTTP 방식 (기본)
./flagd-load.sh <RPS> <DURATION_SEC> <FLAGS>
./flagd-load.sh 100 600 "productCatalogFailure,recommendationCache"

# gRPC 방식 (grpcurl 필요)
./flagd-grpc-load.sh <RPS> <DURATION_SEC>
./flagd-grpc-load.sh 50 300
```

---

## 📊 검증 플로우

### 전체 애플리케이션 (Locust)

```
1. demo-controller: high 시나리오 (50 users)
   ↓
2. 5분 대기
   ↓
3. SigNoz에서 메트릭 확인
   ↓
4. ops-monitor: 임계값 초과 감지
   ↓
5. Discord #bug-reporting 알림 ✅
6. Jira KAN 이슈 생성 ✅
```

### FlagD 서비스

```
1. demo-controller: flagd_high (100 RPS)
   ↓
2. 5-10분 대기
   ↓
3. SigNoz에서 flagd 메트릭 확인
   - flagd_resolve_duration
   - flagd_request_total
   ↓
4. 응답 시간 증가 확인
```

---

## 🐛 Troubleshooting

### Locust 응답 없음

```bash
docker ps | grep load-generator
docker logs load-generator

# 포트 확인
lsof -i :65215
```

### FlagD 연결 실패

```bash
# HTTP 테스트
curl -v http://localhost:65156/health

# gRPC 테스트 (grpcurl 설치 필요)
grpcurl -plaintext localhost:65157 list
```

### ops-monitor 알림 없음

```bash
# ops-monitor cronjob 확인
openclaw cron list | grep ops-monitor

# 다음 실행 시간 확인
openclaw cron list --json | jq '.jobs[] | select(.agentId=="ops-monitor") | .state.nextRunAtMs'

# SigNoz 메트릭 확인
open http://localhost:3301
```

---

## 📁 파일 구조

```
agents/demo-controller/
├── config.json                      # 시나리오 정의 (Locust + FlagD)
├── memory/
│   └── demo-controller-state.json  # 현재 상태
└── scenarios/
    ├── change-load.sh              # Locust 부하 변경
    ├── get-stats.sh                # Locust 통계 조회
    ├── flagd-load.sh               # FlagD HTTP 부하 생성
    └── flagd-grpc-load.sh          # FlagD gRPC 부하 생성
```

---

## 🎯 다음 단계

### 1. 현재 상태 확인

```bash
cd ~/.openclaw/workspace/agents/demo-controller/scenarios
./get-stats.sh
```

### 2. FlagD 부하 테스트

```bash
# 백그라운드로 10분간 100 RPS
./flagd-load.sh 100 600 &

# SigNoz에서 확인
open http://localhost:3301
```

### 3. ops-monitor 검증

다음 정각(11:00)에 자동으로:
- demo-controller → Locust high 시나리오
- demo-controller → FlagD 100 RPS (15분마다)
- ops-monitor → 메트릭 체크 (5분마다)

---

## 📚 참고

- [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/)
- [Locust Documentation](https://docs.locust.io/)
- [FlagD API](https://github.com/open-feature/flagd)
- ops-monitor: `agents/ops-monitor/AGENT.md`
