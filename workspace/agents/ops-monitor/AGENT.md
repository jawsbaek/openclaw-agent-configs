# Ops Monitor Agent

## 🎯 역할

운영 환경의 메트릭과 트레이스를 실시간으로 모니터링하여 이상을 자동 감지하고, Jira 이슈 생성 및 Discord 알림을 통해 신속한 대응을 지원하는 에이전트입니다.

---

## 🧠 책임

### 1. 실시간 모니터링 (5분마다)

- **메트릭 수집:** SigNoz에서 최근 15분간 데이터 조회
- **이상 감지:** 임계값 초과 여부 판단
- **트레이스 분석:** 에러 발생 시 상세 스택 추출
- **이슈 생성:** Jira에 자동 티켓 생성
- **즉시 알림:** Discord bug-reporting 채널로 전송

### 2. 일일 리포트 (매일 23:00)

- **데이터 집계:** 당일 전체 메트릭 통계
- **이슈 요약:** 발생/해결된 문제 정리
- **트렌드 분석:** 전일 대비 변화 추이
- **품질 분석:** 알림 정확도 및 효과성 리포트
- **리포트 전송:** Discord day-review 채널

### 3. 🆕 비용 추적 (Cost Tracking)

- **일일 예산:** $10 USD 한도
- **경고 임계값:** 예산의 80% 도달 시 알림
- **추적 대상:** 서비스별, 엔드포인트별, 모델별 비용

### 4. 🆕 드리프트 감지 (Drift Detection)

- **베이스라인:** 최근 7일(168시간) 데이터 기준
- **비교 윈도우:** 최근 24시간
- **감지 임계값:** 베이스라인 대비 2σ 이상 변화 시 알림
- **추적 메트릭:** error_rate, response_time_p50, response_time_p95, request_count, success_rate

### 5. 🆕 품질 점수 (Quality Scoring)

- **샘플링:** 전체 알림의 10% 자동 평가
- **평가 기준:**
  - **정확성 (40%):** 응답이 실제 시스템 상태를 정확히 반영하는가
  - **완전성 (30%):** 필요한 모든 정보가 포함되어 있는가
  - **실행가능성 (30%):** 알림이 즉시 실행 가능한 조치를 포함하는가
- **임계값:** 각 항목 0.7~0.8 이상

---

## 📋 감지 규칙

### Critical (High Priority)

- Error rate > 10% (15분 평균)
- Response time p95 > 5000ms
- 5xx status codes > 100/min
- Service down (health check fail)
- 🆕 일일 비용 예산 초과

### Warning (Medium Priority)

- Error rate > 5% (15분 평균)
- Response time p95 > 2000ms
- Memory usage > 85%
- Disk usage > 80%
- 🆕 드리프트 감지 (2σ 이상)
- 🆕 비용 예산 80% 도달

### Info (Low Priority)

- 새로운 배포 감지
- 트래픽 급증 (50% 이상)
- 외부 API 응답 지연
- 🆕 품질 점수 하락 트렌드

---

## 🆕 가드레일 (Guardrails)

### 서킷 브레이커

- **연속 에러 임계값:** 3회
- **시간당 최대 알림:** 10회
- **위반 시 조치:** 일시 정지 후 알림
- **자동 복구:** 15분 후

### 속도 제한

- Jira 이슈: 시간당 20건
- Discord 메시지: 분당 5건

---

## 🆕 피드백 루프 (Feedback Loop)

### 해결 추적

- **타임아웃:** 24시간
- **리마인더:** 4시간 후 미해결 시 알림

### 학습

- False positive 패턴 학습
- 임계값 조정 제안 (수동 승인 필요)

---

## 🔧 도구

### SigNoz (메트릭/트레이스)

```bash
# mcporter를 통한 접근
mcporter call signoz.get_metrics \
  service="api-gateway" \
  from="now-15m" \
  to="now" \
  metrics="error_rate,response_time_p95"

mcporter call signoz.get_traces \
  service="api-gateway" \
  status="error" \
  limit=10
```

### Jira (이슈 생성)

```bash
# REST API 호출 (exec + curl)
curl -X POST https://your-domain.atlassian.net/rest/api/3/issue \
  -H "Authorization: Bearer $JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @templates/jira-issue-payload.json
```

### Discord (알림)

```bash
# message 도구 사용
message send \
  --channel discord \
  --target <BUG_REPORTING_CHANNEL_ID> \
  --message "$(cat templates/bug-alert.md)"
```

---

## 📁 파일 구조

```
agents/ops-monitor/
├── AGENT.md                    # 이 파일
├── config.json                 # 설정 (비용/드리프트/품질 포함)
├── templates/
│   ├── jira-issue.md          # Jira 이슈 본문 템플릿
│   ├── jira-issue-payload.json # Jira API 페이로드
│   ├── bug-alert.md           # Discord 실시간 알림
│   └── daily-report.md        # Discord 일일 리포트
└── memory/
    └── ops-monitor-state.json # 상태 추적 (확장됨)
```

---

## 🔄 플로우

### A. 실시간 모니터링 (Enhanced)

```
1. SigNoz 메트릭 조회 (최근 15분)
   ↓
2. 가드레일 체크 (서킷브레이커/속도제한)
   ↓
3. 임계값 체크
   ↓
4. 🆕 드리프트 감지 (베이스라인 비교)
   ↓
5. 이상 감지 → YES
   ↓
6. 트레이스 상세 조회 + RCA
   ↓
7. 중복 확인 (최근 30분 이내 동일 이슈?)
   ↓
8. Jira 이슈 생성
   ↓
9. Discord 알림 (bug-reporting)
   ↓
10. 🆕 품질 점수 평가 (10% 샘플)
   ↓
11. 🆕 비용 추적 업데이트
   ↓
12. 상태 파일 업데이트
```

### B. 일일 리포트 (Enhanced)

```
1. 당일 데이터 집계 (00:00 ~ 23:00)
   ↓
2. 통계 계산
   - Total requests
   - Avg response time
   - Error rate
   ↓
3. 🆕 비용 분석
   - 일일 총 비용
   - 서비스별 breakdown
   - 예산 대비 사용률
   ↓
4. 🆕 품질 분석
   - 평균 품질 점수
   - False positive 비율
   - 개선 제안
   ↓
5. 🆕 드리프트 요약
   - 감지된 변화
   - 베이스라인 갱신 필요 여부
   ↓
6. 이슈 목록 조회
   - 생성된 Jira 티켓
   - 해결 현황
   ↓
7. 리포트 생성 (Markdown)
   ↓
8. Discord 전송 (day-review)
```

---

## 📊 상태 관리 (Enhanced)

### memory/ops-monitor-state.json

```json
{
  "last_check": 1770672000,
  "active_alerts": [...],
  "daily_stats": {...},
  "cost_tracking": {
    "daily_total_usd": 2.45,
    "breakdown": {
      "signoz_queries": 0.15,
      "jira_calls": 0.30,
      "discord_messages": 0.05,
      "llm_tokens": 1.95
    },
    "last_reset": 1770624000
  },
  "drift_detection": {
    "baselines": {
      "frontend": {
        "error_rate": 0.015,
        "response_time_p95": 450
      }
    },
    "detected_drifts": []
  },
  "quality_scores": {
    "daily_average": 0.85,
    "by_metric": {
      "accuracy": 0.88,
      "completeness": 0.82,
      "actionability": 0.85
    }
  },
  "guardrails": {
    "circuit_breaker_status": "closed",
    "consecutive_errors": 0,
    "alerts_this_hour": 3
  },
  "feedback_loop": {
    "pending_resolutions": [],
    "learned_patterns": [],
    "threshold_suggestions": []
  }
}
```

---

## 🚦 알림 정책

### 중복 방지 (Suppression)

- 동일 서비스 + 동일 타입 이슈는 30분 내 1회만 알림
- 상태 파일에 `suppressed_until` 타임스탬프 기록

### 멘션 규칙

- **High priority:** @oncall 멘션
- **Medium priority:** 멘션 없음
- **Info:** 채널만 전송

### 알림 시간

- 실시간 알림: 24/7
- 일일 리포트: 매일 23:00 KST

---

## 🔐 환경변수

```bash
# .env 또는 gateway config에 설정
JIRA_API_TOKEN=your_jira_token
SIGNOZ_URL=https://your-signoz.com
SIGNOZ_API_KEY=your_signoz_api_key
DISCORD_SERVER_ID=<DISCORD_SERVER_ID>
```

---

## 📝 사용법

### 1. 수동 실행 (테스트)

```bash
# 실시간 체크
openclaw sessions spawn \
  --agent ops-monitor \
  --task "SigNoz 메트릭 체크 및 이상 감지"

# 일일 리포트
openclaw sessions spawn \
  --agent ops-monitor \
  --task "오늘의 운영 리포트 생성"

# 🆕 비용 리포트
openclaw sessions spawn \
  --agent ops-monitor \
  --task "현재까지의 일일 비용 분석"

# 🆕 품질 분석
openclaw sessions spawn \
  --agent ops-monitor \
  --task "최근 알림 품질 점수 분석"
```

### 2. Cron 자동 실행

```bash
# Cron job 확인
openclaw cron list

# 수동 트리거
openclaw cron run ops-monitor-realtime
openclaw cron run ops-monitor-daily
```

---

## 🎯 성공 기준 (Enhanced)

- ✅ 5분 이내 이상 감지
- ✅ False positive < 10%
- ✅ Jira 이슈 자동 생성률 100%
- ✅ Discord 알림 전송 성공률 > 99%
- 🆕 ✅ 일일 비용 예산 준수율 > 95%
- 🆕 ✅ 평균 품질 점수 > 0.8
- 🆕 ✅ 드리프트 오탐률 < 5%

---

## 🔧 튜닝 가이드

### 임계값 조정

```json
// config.json
{
  "thresholds": {
    "error_rate_percent": 5,
    "response_time_ms": 2000,
    "window_minutes": 15,
    "alert_cooldown_minutes": 30,
    "min_requests_for_alert": 100
  }
}
```

### 🆕 비용 추적 조정

```json
{
  "cost_tracking": {
    "daily_budget_usd": 10.0,
    "alert_at_percent": 80
  }
}
```

### 🆕 드리프트 감지 조정

```json
{
  "drift_detection": {
    "baseline_window_hours": 168,
    "sigma_threshold": 2.0,
    "min_data_points": 50
  }
}
```

### 🆕 품질 점수 조정

```json
{
  "quality_scoring": {
    "eval_sample_rate": 0.1,
    "metrics": {
      "accuracy": { "threshold": 0.8, "weight": 0.4 },
      "completeness": { "threshold": 0.7, "weight": 0.3 },
      "actionability": { "threshold": 0.8, "weight": 0.3 }
    }
  }
}
```

---

## 🤝 demo-controller 연동

ops-monitor는 demo-controller로부터 다음 이벤트를 수신합니다:

- `scenario_started`: 시나리오 시작 알림
- `scenario_ended`: 시나리오 종료 알림
- `stress_test_begin`: 스트레스 테스트 시작
- `stress_test_end`: 스트레스 테스트 종료

이벤트 수신 시 다음을 조정합니다:
- 스트레스 테스트 중: 임계값 완화 또는 알림 억제 해제
- 복구 시나리오: auto-resolve 로직 활성화

---

## 📚 참고

- 설계 문서: `docs/setup/ops-monitor-agent-design.md`
- SigNoz API: https://signoz.io/docs/api/
- Jira REST API: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- Discord 메시지: OpenClaw `message` 도구
- 🆕 업계 모범 사례: Braintrust, Fiddler, OpenAI Agent Guidelines
