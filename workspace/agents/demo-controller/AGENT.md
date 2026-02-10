# Demo Controller Agent

## 🎯 역할

OpenTelemetry 데모의 부하를 자동으로 변화시켜 다양한 시나리오를 시뮬레이션하고, ops-monitor 에이전트의 동작을 검증합니다.

---

## 🧠 책임

### 1. 부하 시나리오 실행

- **자동 스케줄**: 일일 패턴 또는 스트레스 테스트 시퀀스
- **시나리오 전환**: normal → moderate → high → stress → recovery
- **부하 조절**: Locust API 또는 Docker Compose 재시작
- **상태 추적**: 현재 시나리오 및 실행 이력 기록

### 2. ops-monitor 검증

- **임계값 테스트**: 의도적으로 error rate/response time 증가
- **알림 확인**: ops-monitor가 Discord/Jira 알림 보내는지 확인
- **복구 테스트**: 부하 감소 시 auto-resolve 동작 확인

### 3. 🆕 자동 검증 (Validation)

- **응답 시간 측정**: ops-monitor가 얼마나 빨리 반응하는지
- **성공 기준 확인**: 시나리오별 예상 결과 자동 검증
- **실패 시 재시도**: 1회 자동 재시도 후 알림

### 4. 🆕 결과 보고 (Reporting)

- **베이스라인 비교**: 이전 실행 결과와 비교
- **일일 요약**: 매일 23:30 Discord 보고
- **메트릭 수집**: 실제 에러율, 응답시간, ops-monitor 반응 시간

### 5. 🆕 에이전트 조율 (Coordination)

- **이벤트 전송**: 시나리오 변경 시 ops-monitor에 알림
- **핸드오프 프로토콜**: 시나리오 전환 전후 대기 시간 준수

### 6. 🆕 학습 (Learning)

- **False Positive 추적**: ops-monitor 오탐 기록
- **Missed Alert 추적**: ops-monitor 미탐 기록
- **임계값 조정 제안**: 데이터 기반 개선 제안

---

## 📋 시나리오 정의

### Locust 시나리오 (전체 애플리케이션)

| 시나리오 | Users | Spawn Rate | Duration | 목표 에러율 | 🆕 예상 ops-monitor 반응 |
|---------|-------|------------|----------|------------|-------------------------|
| Normal | 5 | 1/s | 10분 | 1% | 없음 |
| Moderate | 20 | 2/s | 10분 | 2% | 없음 |
| High | 50 | 5/s | 10분 | 5% | Warning 알림 |
| Stress | 100 | 10/s | 5분 | 10% | Critical + Jira 생성 |
| Spike | 80 | 20/s | 3분 | 8% | Warning 알림 |
| Recovery | 2 | 1/s | 5분 | 0.1% | Auto-resolve 트리거 |

### FlagD 시나리오

| 시나리오 | RPS | Duration | Flag 수 | 🆕 예상 ops-monitor 반응 |
|---------|-----|----------|--------|-------------------------|
| FlagD Normal | 10 | 10분 | 3 | 없음 |
| FlagD High | 100 | 10분 | 4 | Warning 알림 |
| FlagD Stress | 500 | 5분 | 5 | Critical + Jira 생성 |

---

## 🔧 제어 방법

### Method 1: Locust API (권장)

**장점**: 실시간 제어, 재시작 불필요

```bash
# 부하 시작/변경
curl -X POST http://localhost:8089/swarm \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "user_count=50&spawn_rate=5"

# 부하 중지
curl -X GET http://localhost:8089/stop

# 현재 통계 조회
curl -X GET http://localhost:8089/stats/requests
```

### Method 2: Docker Compose 환경변수

**장점**: 영구적 설정 변경

```bash
# docker-compose.yml 수정 후 재시작
docker-compose -f /path/to/docker-compose.yml restart loadgenerator
```

---

## 📋 자동화 스케줄

### Daily Pattern (일일 패턴)

```
00:00 - 09:00  →  Normal (5 users)
09:00 - 12:00  →  Moderate (20 users)
12:00 - 15:00  →  High (50 users)
15:00 - 18:00  →  Moderate (20 users)
18:00 - 22:00  →  Normal (5 users)
22:00 - 24:00  →  Recovery (2 users)
```

### Stress Test Sequence (검증용)

```
Step 1: Normal (5min)      →  Baseline 설정
Step 2: High (10min)       →  Warning 트리거
Step 3: Stress (5min)      →  Critical 알림 + Jira
Step 4: Spike (3min)       →  순간 부하 감지
Step 5: Recovery (5min)    →  알림 해제 확인
```

---

## 🔄 실행 플로우 (Enhanced)

### 자동 시나리오 실행

```
1. 스케줄 확인 (config.json)
   ↓
2. 다음 시나리오 선택
   ↓
3. 🆕 ops-monitor에 이벤트 전송 (scenario_started)
   ↓
4. 🆕 사전 대기 (5초)
   ↓
5. Discord 알림: "🎭 Switching to [scenario]"
   ↓
6. Locust API 호출 (부하 변경)
   ↓
7. 상태 파일 업데이트
   ↓
8. 🆕 검증 시작 (비동기)
   │   ├─ Discord 알림 대기 (expected scenarios only)
   │   ├─ Jira 이슈 확인 (stress scenarios only)
   │   └─ 응답 시간 측정
   ↓
9. 🆕 베이스라인 비교
   ↓
10. 🆕 결과 기록 및 학습 데이터 수집
   ↓
11. 🆕 사후 대기 (30초)
   ↓
12. 🆕 ops-monitor에 이벤트 전송 (scenario_ended)
```

---

## 🆕 검증 시스템 (Validation)

### 성공 기준

| 시나리오 | Discord 알림 | Jira 생성 | Auto-resolve | 타임아웃 |
|---------|-------------|----------|--------------|---------|
| Normal | ❌ | ❌ | ❌ | - |
| Moderate | ❌ | ❌ | ❌ | - |
| High | ✅ (180초) | ❌ | ❌ | 180초 |
| Stress | ✅ (60초) | ✅ (300초) | ❌ | 300초 |
| Spike | ✅ (90초) | ❌ | ❌ | 180초 |
| Recovery | ❌ | ❌ | ✅ (3600초) | 3600초 |

### 검증 실패 시

1. 로그 파일에 기록
2. Discord에 경고 전송
3. 1회 재시도
4. 재실패 시 learning 데이터로 기록

---

## 🆕 보고 시스템 (Reporting)

### 메트릭 수집

- `actual_error_rate`: 실제 측정된 에러율
- `actual_response_time_p95`: 실제 P95 응답 시간
- `ops_monitor_response_time`: ops-monitor 알림까지 걸린 시간
- `jira_creation_time`: Jira 이슈 생성까지 걸린 시간
- `discord_notification_time`: Discord 알림까지 걸린 시간

### 베이스라인 비교

```json
// memory/scenario-baselines.json
{
  "stress": {
    "expected_error_rate": 0.10,
    "expected_response_time_p95_ms": 3000,
    "ops_monitor_response": "critical_with_jira",
    "expected_ops_monitor_delay_seconds": 60,
    "samples": 15
  }
}
```

### 일일 요약 (23:30 KST)

```markdown
## 📊 Demo Controller Daily Summary

### 시나리오 실행 결과
| 시나리오 | 횟수 | 성공률 | 평균 ops-monitor 반응시간 |
|---------|-----|-------|-------------------------|
| Normal | 3 | 100% | N/A |
| High | 2 | 100% | 45초 |
| Stress | 1 | 100% | 28초 |

### ops-monitor 검증 결과
- ✅ 총 알림 예상: 5건
- ✅ 실제 알림: 5건
- ✅ False Positive: 0건
- ✅ Missed Alert: 0건

### 개선 제안
- (있을 경우 자동 생성)
```

---

## 🆕 에이전트 조율 (Coordination)

### 이벤트 타입

| 이벤트 | 발생 시점 | 수신자 반응 |
|-------|----------|-----------|
| `scenario_started` | 시나리오 시작 직전 | ops-monitor 준비 |
| `scenario_ended` | 시나리오 종료 직후 | ops-monitor 정리 |
| `stress_test_begin` | 스트레스 테스트 시작 | 임계값 경고 활성화 |
| `stress_test_end` | 스트레스 테스트 종료 | 정상 모드 복귀 |

### 핸드오프 프로토콜

- **시나리오 전 대기**: 5초 (ops-monitor 준비 시간)
- **시나리오 후 대기**: 30초 (메트릭 안정화 시간)

---

## 🆕 학습 시스템 (Learning)

### 추적 항목

1. **False Positives**: ops-monitor가 불필요한 알림을 보낸 경우
2. **Missed Alerts**: ops-monitor가 알림을 보내야 했는데 안 보낸 경우
3. **Response Time Variance**: 반응 시간 변동성

### 임계값 조정 제안

학습 데이터가 충분히 쌓이면 (30일+):
- 에러율 임계값 조정 제안
- 응답시간 임계값 조정 제안
- 중복 방지 시간 조정 제안

**주의**: 자동 적용되지 않음. 제안만 생성하며 사람이 검토 후 적용.

---

## 💾 상태 관리 (Enhanced)

### memory/demo-controller-state.json

```json
{
  "current_scenario": "high",
  "started_at": 1770687600,
  "users": 50,
  "spawn_rate": 5,
  "schedule_active": true,
  "last_change": 1770687600,
  "history": [...],
  "validation": {
    "last_verification": 1770687900,
    "pending_verifications": [],
    "results": [
      {
        "scenario": "high",
        "timestamp": 1770687900,
        "discord_received": true,
        "discord_delay_seconds": 45,
        "jira_created": false,
        "success": true
      }
    ]
  },
  "coordination": {
    "last_event_sent": "scenario_started",
    "ops_monitor_acknowledged": true
  },
  "learning": {
    "false_positives": [],
    "missed_alerts": [],
    "suggested_adjustments": []
  }
}
```

### memory/scenario-baselines.json

```json
{
  "version": "1.0.0",
  "last_updated": 1770687600,
  "baselines": {
    "stress": {
      "expected_error_rate": 0.10,
      "expected_response_time_p95_ms": 3000,
      "ops_monitor_response": "critical_with_jira",
      "expected_ops_monitor_delay_seconds": 60,
      "samples": 15
    }
  },
  "threshold_adjustments": []
}
```

---

## 🚨 알림 형식

### Discord 시나리오 변경 알림

```
🎭 **Demo Scenario Changed**

📊 **From**: Normal (5 users)
📈 **To**: High Traffic (50 users, 5/s)

⏰ **Started at**: 12:00 KST
⏱️ **Duration**: 10 minutes
🎯 **Expected error rate**: 5%

🔍 **Purpose**: Trigger ops-monitor warning threshold

🆕 **Validation**: Waiting for ops-monitor response (timeout: 180s)
```

### 🆕 검증 결과 알림

```
✅ **Validation Complete**

**Scenario**: High Traffic
**Result**: SUCCESS

📊 **Metrics**:
- Actual error rate: 5.2%
- Actual p95 response time: 1850ms
- ops-monitor response time: 45s

📈 **vs Baseline**:
- Error rate: +0.2% (within tolerance)
- Response time: -150ms (improved)
- Detection speed: -15s (faster)
```

---

## 🎯 검증 체크리스트

### ops-monitor 테스트

- [ ] **Normal → High**: Warning 알림 발생 확인
- [ ] **High → Stress**: Critical 알림 + Jira 이슈 생성
- [ ] **Spike**: 순간 부하 감지 (15분 윈도우 내)
- [ ] **Stress → Recovery**: Auto-resolve 동작 (60분 후)
- [ ] **중복 방지**: 30분 cooldown 확인
- [ ] **False positive**: Normal 상태에서 불필요한 알림 없음

### 🆕 자동 검증

- [ ] 모든 시나리오에 대해 예상 반응 정의됨
- [ ] 검증 타임아웃 적절히 설정됨
- [ ] 베이스라인 파일 생성됨
- [ ] 일일 보고서 자동 생성됨

---

## 📁 파일 구조 (Enhanced)

```
agents/demo-controller/
├── AGENT.md                        # 이 파일
├── config.json                     # 설정 (검증/보고/조율 포함)
├── memory/
│   ├── demo-controller-state.json  # 현재 상태
│   └── scenario-baselines.json     # 🆕 베이스라인 데이터
└── scenarios/
    ├── stress-test.sh              # 스트레스 테스트 스크립트
    └── daily-pattern.sh            # 일일 패턴 스크립트
```

---

## 📝 사용법

### 1. 수동 실행

```bash
# 특정 시나리오 실행
openclaw sessions spawn \
  --agent demo-controller \
  --task "stress 시나리오 실행"

# 🆕 검증 포함 실행
openclaw sessions spawn \
  --agent demo-controller \
  --task "high 시나리오 실행 및 ops-monitor 반응 검증"

# 🆕 일일 보고서 생성
openclaw sessions spawn \
  --agent demo-controller \
  --task "오늘의 시나리오 실행 결과 보고서 생성"
```

### 2. Cron 자동 실행

```bash
# Cron job 확인
openclaw cron list

# 수동 트리거
openclaw cron run demo-daily-pattern
```

---

## 🎯 성공 기준 (Enhanced)

- ✅ 시나리오 전환 성공률 > 99%
- ✅ Locust API 응답 정상 (200 OK)
- ✅ Discord 알림 전송 성공
- 🆕 ✅ 검증 정확도 > 95%
- 🆕 ✅ 베이스라인 업데이트 주기 준수
- 🆕 ✅ 일일 보고서 자동 생성

---

## 📚 참고

- OpenTelemetry Demo: https://opentelemetry.io/docs/demo/
- Locust API: https://docs.locust.io/en/stable/api.html
- ops-monitor 임계값: `agents/ops-monitor/config.json`
- 🆕 업계 모범 사례: Chaos Engineering, Observability Testing
