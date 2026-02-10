# CRON_PROMPTS.md - Jira Resolver용 권장 프롬프트

아래는 OpenClaw cron의 `payload.kind=agentTurn`에 넣기 좋은 프롬프트 템플릿입니다.

---

## 1) 5분 주기: Jira 큐 점검/처리 및 자동 복구

**Job name:** jira-resolver-realtime

**Prompt:**
```
Jira Resolver 정기 실행입니다. project=KAN 기준으로 statusCategory != Done 티켓을 조회하고, 상태별로 처리하세요.

1. SigNoz 메트릭 조회:
   ~/.openclaw/skills/jira-resolver/scripts/check-signoz-metrics.sh 5 10.0

2. 티켓 처리:
   - 해야 할 일: 진행 중으로 전환 후 진단 시작 코멘트
   - 진행 중: 자동 조치 가능 여부 판단
     * 에러율 10% 이상 서비스: auto-restart-service.sh 사용하여 재시작
     * 재시작 후 검토 중으로 전환
   - 검토 중: 5분 메트릭 재검증 후 완료/재진행 결정
     * 에러율 1% 미만: 완료
     * 에러율 여전히 높음: 진행 중으로 되돌림

3. 중복 이슈 정리 (5% 확률):
   ~/.openclaw/skills/jira-resolver/scripts/cleanup-duplicate-issues.sh false

모든 코멘트는 한국어로, 근거(메트릭/재시작 로그) 포함.
```

---

## 2) 30분 주기: 정체 티켓 에스컬레이션

**Job name:** jira-resolver-stuck-check

**Prompt:**
`Jira Resolver 정체 티켓 점검입니다. 진행 중/검토 중 상태에서 30분 이상 업데이트 없는 티켓을 찾아 코멘트로 현재 병목, 필요한 수동 조치, 담당자 액션 아이템을 한국어로 남기세요. 필요 시 우선순위 상향을 제안하세요.`

---

## 3) 일일 23:00: 운영 요약 및 일일 리포트

**Job name:** jira-resolver-daily-summary

**Prompt:**
```
Jira Resolver 일일 요약 리포트를 생성하세요.

1. SigNoz 24시간 메트릭 조회:
   ~/.openclaw/skills/jira-resolver/scripts/check-signoz-metrics.sh 1440 1.0

2. Jira 티켓 통계:
   - 오늘 생성된 티켓 수
   - 완료 처리 건수 (자동/수동 구분)
   - 평균 해결 시간
   - 현재 미완료 티켓 수 및 상태별 분포

3. 장애 패턴 분석:
   - 반복 장애 TOP 3 (서비스별)
   - 자동 재시작 성공률
   - 중복 이슈 정리 건수

4. 내일 개선 액션:
   - Circuit Breaker 필요 서비스
   - 모니터링 임계값 조정 필요 여부
   - 크론잡 일정 최적화 제안

결과는 한국어 markdown 형식으로 작성하고 memory/YYYY-MM-DD.md에 저장.
```

---

## 4) 원샷 리마인더(선택)

**Prompt:**
`[리마인더] KAN-123 티켓 검증 대기 20분 경과. 재검증(에러율/응답시간) 후 상태를 업데이트하세요.`

---

## 권장 스케줄 예시

- realtime: `*/5 * * * *`
- stuck-check: `*/30 * * * *`
- daily-summary: `0 23 * * *`

> 주의: 실제 cron 등록 시 중복 실행 방지를 위해 기존 동일 목적 job 존재 여부를 먼저 확인하세요.
