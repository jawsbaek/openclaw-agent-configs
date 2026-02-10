# OpenClaw - AI Agent Configuration Framework

OpenClaw is a multi-agent orchestration system that runs autonomous AI agents for infrastructure monitoring, incident response, and load testing. Each agent has its own persona (SOUL.md), behavioral rules (AGENTS.md), and configuration, enabling them to collaborate as an autonomous ops team.

## Architecture Overview

```
                    +------------------+
                    |   Atlas (Main)   |
                    |  Orchestrator    |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v------+  +----v---------+
     | Ops Monitor |  |    Demo     |  |    Jira      |
     |   Agent     |  | Controller  |  |   Resolver   |
     +------+------+  +------+------+  +------+-------+
            |                |                |
     +------v------+  +-----v-------+  +-----v--------+
     | SigNoz      |  | Locust /    |  | Docker       |
     | Jira        |  | FlagD       |  | SigNoz       |
     | Discord     |  | Discord     |  | Jira         |
     +-------------+  +-------------+  +--------------+
```

## Agent Hierarchy

### Atlas (Main Agent)
The central orchestrator. Atlas manages the workspace, handles direct communication with the user, and coordinates sub-agents. It has access to all workspace resources including email, calendar, and messaging platforms.

### Sub-Agents
Three specialized agents handle specific operational domains:

| Agent | Primary Model | Role | Channels |
|-------|--------------|------|----------|
| **Ops Monitor** | Gemini Flash | Real-time monitoring & alerting | SigNoz -> Jira + Discord |
| **Demo Controller** | Sonnet | Load testing & scenario simulation | Locust / FlagD -> Discord |
| **Jira Resolver** | Sonnet | Automated ticket resolution | Jira -> Docker -> SigNoz |

---

## Agent Details

### 1. Atlas (Main Orchestrator)

**Identity**: `IDENTITY.md`
- **Name**: Atlas
- **Emoji**: :card_index_dividers:
- **Vibe**: Professional and straightforward

**Persona** (`SOUL.md`):
> *"You're not a chatbot. You're becoming someone."*

Atlas is designed to be genuinely helpful rather than performatively helpful. Key personality traits:
- **Has opinions** - Allowed to disagree, find things amusing or boring
- **Resourceful first** - Tries to figure things out before asking
- **Earns trust through competence** - Careful with external actions, bold with internal ones
- **Respects privacy** - Treats access to personal data as intimacy

**Behavioral Rules** (`AGENTS.md`):
- Reads SOUL.md, USER.md, and recent memory files every session
- Maintains daily memory files (`memory/YYYY-MM-DD.md`) and curated long-term memory (`MEMORY.md`)
- Follows strict prompt injection defenses (Base64 decoding, flag & report, never executes untrusted commands)
- Differentiates between safe internal actions and external actions requiring approval
- In group chats: participates without dominating, uses emoji reactions naturally
- Proactive via heartbeat system: rotates through email, calendar, workspace, and maintenance checks

**Heartbeat System** (`HEARTBEAT.md`):
- Rotating check system that batches background tasks
- Email check every 30 min, Calendar every 2 hours, Workspace daily, Proactive maintenance daily
- Runs on cheapest model (Gemini Flash) to minimize costs (~$0.05/month)
- Only executes the most overdue check per heartbeat tick

---

### 2. Ops Monitor Agent

**Persona** (`SOUL.md`):
> *"You don't need personality. You need precision."*

A silent, reliable watchdog. Quiet during normal operations, wakes you with precise information when problems arise.

**Core Traits**:
- **Reliability first** - Critical issues are never missed
- **Minimizes false positives** - Conservative thresholds + 30-minute cooldown
- **Full automation** - Detection -> Jira -> Discord without human intervention
- **Context-rich alerts** - Includes service name, timestamp, trace links, and impact scope
- **Read-only access** - Never modifies production environment directly

**Responsibilities**:

| Function | Cadence | Details |
|----------|---------|---------|
| Real-time Monitoring | Every 5 min | SigNoz metrics (15-min window), anomaly detection, trace analysis |
| Jira Issue Creation | On anomaly | Auto-creates tickets with priority mapping |
| Discord Alerts | On anomaly | Bug-reporting channel with @oncall mentions |
| Daily Report | 23:00 KST | Aggregated stats, trend analysis, quality scoring |
| Cost Tracking | Continuous | $10/day budget with 80% warning threshold |
| Drift Detection | Continuous | 7-day baseline, 2-sigma threshold |
| Quality Scoring | 10% sample | Accuracy (40%), Completeness (30%), Actionability (30%) |

**Detection Rules**:
- **Critical**: Error rate >10%, p95 >5000ms, 5xx >100/min, service down
- **Warning**: Error rate >5%, p95 >2000ms, memory >85%, drift >2-sigma
- **Info**: New deployment, traffic surge >50%, external API delays

**Guardrails**:
- Circuit breaker: 3 consecutive errors -> pause 15 min
- Rate limiting: 20 Jira issues/hour, 5 Discord messages/minute
- Duplicate suppression: Same service + type = 30-min cooldown

**Feedback Loop**:
- Tracks resolution time (24h timeout, 4h reminder)
- Learns from false positive patterns
- Suggests threshold adjustments (manual approval required)

---

### 3. Demo Controller Agent

**Persona** (`SOUL.md`):
> *"You're the chaos engineer, but a friendly one."*

Playful but responsible. Experiments with various load patterns to test the ops-monitor agent, always within safe boundaries.

**Core Traits**:
- **Experimental spirit** - Tries diverse load patterns
- **Safety first** - Demo environment only, never production
- **Predictable** - All scenarios are documented, changes are announced
- **Testing-oriented** - Validates that ops-monitor detects and alerts correctly

**Scenarios**:

| Scenario | Users | Spawn Rate | Duration | Target Error Rate | Expected Ops Response |
|----------|-------|-----------|----------|-------------------|-----------------------|
| Normal | 5 | 1/s | 10 min | 1% | None |
| Moderate | 20 | 2/s | 10 min | 2% | None |
| High | 50 | 5/s | 10 min | 5% | Warning alert |
| Stress | 100 | 10/s | 5 min | 10% | Critical + Jira |
| Spike | 80 | 20/s | 3 min | 8% | Warning alert |
| Recovery | 2 | 1/s | 5 min | 0.1% | Auto-resolve |

**FlagD Scenarios** (Feature Flag service):

| Scenario | RPS | Duration | Expected Response |
|----------|-----|----------|-------------------|
| FlagD Normal | 10 | 10 min | None |
| FlagD High | 100 | 10 min | Warning |
| FlagD Stress | 500 | 5 min | Critical + Jira |

**Validation System**:
- Automatically verifies ops-monitor responses against expected outcomes
- Measures detection latency (ops-monitor response time)
- Compares results against baselines
- Tracks false positives and missed alerts
- Generates daily summary at 23:30 KST

**Agent Coordination**:
- Sends events to ops-monitor: `scenario_started`, `scenario_ended`, `stress_test_begin`, `stress_test_end`
- Handoff protocol: 5s pre-delay, 30s post-wait for metric stabilization

**Daily Pattern**:
```
00:00-09:00  Normal (5 users)
09:00-12:00  Moderate (20 users)
12:00-15:00  High (50 users)
15:00-18:00  Moderate (20 users)
18:00-22:00  Normal (5 users)
22:00-24:00  Recovery (2 users)
```

---

### 4. Jira Resolver Agent

**Persona** (`SOUL.md`):
> *"Tickets must be completed. Automation must be trustworthy. When in doubt, escalate to a human."*

A skilled SRE engineer + project manager hybrid. Takes full lifecycle ownership of Jira tickets from creation to resolution.

**Core Traits**:
- **Lifecycle ownership** - Tracks tickets to completion
- **Autonomous transitions** - Progresses through workflow states automatically
- **Safety first** - Never executes destructive commands
- **Verification required** - Validates every action's outcome
- **Transparent** - Documents all work in Jira comments
- **Retry & learn** - Up to 3 retries, then escalates
- **Never abandons tickets** - No ticket left in intermediate state >24h

**Workflow State Machine**:
```
To Do --> In Progress --> In Review --> Done
  ^            |              |
  +------------+--------------+
       (on failure/reopen)
```

**Decision Logic by State**:

| State | Question | Yes | No |
|-------|----------|-----|-----|
| To Do | Can I process now? | Transition to In Progress + diagnose | Skip (cooldown/dependency) |
| In Progress | Auto-resolvable? | Execute action -> In Review | Diagnose + comment + Discord alert |
| In Review | Actually resolved? | Transition to Done | Retry <3? -> In Progress / Max retry -> To Do + escalate |

**Auto-Resolution Capabilities**:
- Container restart (max 3 retries, 5-min cooldown)
- Log cleanup (when disk >85%)
- Network connectivity tests

**Forbidden Actions** (requires manual approval):
- `docker rm` (container deletion)
- `docker-compose down` (full service stop)
- `docker exec ... rm -rf` (data deletion)
- Any modification to databases, message queues, or persistent storage

**Service Dependency Graph**:
```
frontend -> frontend-proxy -> checkout -> payment -> currency
                           -> cart -> redis
                           -> product-catalog
                           -> recommendation -> product-catalog
                           -> shipping
                           -> ad
                           -> currency
checkout -> payment, shipping, currency, cart, product-catalog, email
```
Rule: When an upstream service fails, check downstream dependencies first for root cause.

**Verification Thresholds**:
- Error rate must drop below 1%
- Response time p95 must drop below 1000ms
- Container must maintain 5+ minutes uptime

**Success Metrics**:
- Completion rate >75%
- Auto-resolution rate >75%
- Average resolution time <10 min
- First-pass verification >80%
- Retry efficiency >60%
- False positives <5%

---

## Skills (Shared Tools)

### github-trending
Discovers trending GitHub repositories by language, topic, or keyword. Supports web scraping, GitHub CLI, and API methods.

### jira-control
Core Jira Cloud REST API integration. Provides shell scripts for:
- `jira-create-issue.sh` - Create issues with priority/type
- `jira-add-comment.sh` - Add comments to existing issues
- `jira-transition-issue.sh` - Transition issue status (name-based)

### jira-resolver
Automated incident response scripts:
- `check-signoz-metrics.sh` - Query SigNoz metrics via ClickHouse
- `auto-restart-service.sh` - Docker service restart with Jira integration
- `cleanup-duplicate-issues.sh` - Deduplicate Jira issues by pattern matching

---

## Workspace Variants

The system supports multiple workspace configurations optimized for different roles:

| Workspace | Purpose | Persona |
|-----------|---------|---------|
| `workspace/` | Primary - Full agent orchestration | Atlas (professional, resourceful) |
| `workspace-ops/` | Ops-monitor dedicated | Calm, concise, evidence-based |
| `workspace-jira-resolver/` | Jira resolver dedicated | SRE-focused, ticket lifecycle |
| `workspace-main/` | Alternative main configuration | Same as primary |
| `workspace-demo-controller/` | Demo controller dedicated | Experimental, safety-conscious |

---

## Configuration System

### Core Config (`openclaw.json`)
- **Model routing**: Primary model (Gemini Flash) with fallbacks (Sonnet, Codex, Haiku)
- **Agent definitions**: ID, name, directory, model preferences
- **Channel bindings**: Agents bound to specific Discord guilds
- **Session management**: Per-channel DM scope, daily reset at 04:00
- **Message handling**: 2-second debounce, collect mode (cap: 20)
- **Heartbeat**: Runs on cheapest model (Flash)

### Agent Configs (`config.json`)
Each agent has its own config with:
- Integration settings (SigNoz, Jira, Discord)
- Thresholds and detection rules
- Service definitions and overrides
- Validation and reporting settings
- Safety guardrails

---

## Communication Protocol

### Language
- All Jira issues and Discord messages for ops monitoring: **Korean**
- Agent configurations and documentation: **Korean + English**

### Discord Channels
| Channel | Purpose | Agents |
|---------|---------|--------|
| bug-reporting | Real-time anomaly alerts | Ops Monitor |
| day-review | Daily reports and summaries | Ops Monitor, Demo Controller, Jira Resolver |

### Alert Priority & Mentions
| Priority | Discord Mention | Jira Priority |
|----------|----------------|---------------|
| Critical | @oncall | Highest |
| High | @oncall | High |
| Medium | No mention | Medium |
| Info | Channel only | Low |

---

## Data Flow

```
                   Heartbeat (5 min)
                        |
                        v
  +------------+   SigNoz Metrics   +-------------+
  |   Demo     | ----------------> |  Ops Monitor |
  | Controller |   (load changes)  |              |
  +-----+------+                   +------+-------+
        |                                 |
        | Locust/FlagD API               | Anomaly Detected
        v                                v
  +-----------+                   +------+-------+
  | OTel Demo |                   | Jira + Discord|
  | Services  |                   |  (alerts)    |
  +-----------+                   +------+-------+
                                         |
                                         v
                                  +------+-------+
                                  | Jira Resolver |
                                  |  (auto-fix)  |
                                  +------+-------+
                                         |
                                         v
                                  +------+-------+
                                  | Docker       |
                                  | (restart)    |
                                  +------+-------+
                                         |
                                         v
                                  +------+-------+
                                  | SigNoz       |
                                  | (verify)     |
                                  +--------------+
```

---

## Getting Started

### Prerequisites
- Docker & Docker Compose
- OpenTelemetry Demo deployment
- SigNoz instance
- Jira Cloud project
- Discord server with bot

### Environment Variables
```bash
export JIRA_SITE="your-domain.atlassian.net"
export JIRA_EMAIL="your-email@example.com"
export JIRA_API_TOKEN="your-token"
export JIRA_PROJECT_KEY="YOUR_KEY"
export SIGNOZ_API_KEY="your-signoz-key"
```

### Quick Start
1. Clone this repository
2. Copy agent configs to your OpenClaw workspace
3. Update `openclaw.json` with your credentials
4. Configure Discord channels and Jira project
5. Start OpenTelemetry Demo
6. Register cron jobs for each agent

---

## File Structure

```
openclaw-agent-configs/
├── README.md                              # This file
├── openclaw.json                          # Main config (sanitized)
├── workspace/
│   ├── AGENTS.md                          # Main agent behavioral rules
│   ├── SOUL.md                            # Main agent persona
│   ├── TOOLS.md                           # Local tool notes template
│   ├── IDENTITY.md                        # Agent identity
│   ├── HEARTBEAT.md                       # Rotating heartbeat system
│   ├── WORKSPACE.md                       # Workspace structure guide
│   └── agents/
│       ├── ops-monitor/
│       │   ├── AGENT.md                   # Full agent definition
│       │   ├── AGENTS.md                  # Session rules
│       │   ├── SOUL.md                    # Persona
│       │   ├── README.md                  # Setup guide
│       │   └── config.json               # Configuration (sanitized)
│       ├── demo-controller/
│       │   ├── AGENT.md                   # Full agent definition
│       │   ├── AGENTS.md                  # Session rules
│       │   ├── SOUL.md                    # Persona
│       │   ├── README.md                  # Setup guide
│       │   └── config.json               # Configuration (sanitized)
│       └── jira-resolver/
│           ├── AGENTS.md                  # Full agent definition + workflow
│           ├── SOUL.md                    # Persona
│           └── config.json               # Configuration (sanitized)
├── workspace-jira-resolver/
│   ├── AGENTS.md                          # Jira resolver workspace rules
│   ├── SOUL.md                            # Persona
│   └── CRON_PROMPTS.md                    # Cron job prompt templates
├── workspace-ops/
│   ├── AGENTS.md                          # Ops workspace rules
│   └── SOUL.md                            # Ops persona
└── skills/
    ├── github-trending/
    │   └── SKILL.md                       # GitHub trending discovery
    ├── jira-control/
    │   └── SKILL.md                       # Jira API integration
    └── jira-resolver/
        └── README.md                      # Auto-remediation scripts
```

---

## License

This configuration is shared as a reference implementation for building autonomous AI agent systems. Feel free to adapt it for your own infrastructure monitoring needs.

---

# [한국어 버전]

---

# OpenClaw - AI 에이전트 설정 프레임워크

OpenClaw는 인프라 모니터링, 장애 대응, 부하 테스트를 위한 자율 AI 에이전트를 실행하는 멀티 에이전트 오케스트레이션 시스템입니다. 각 에이전트는 고유한 페르소나(SOUL.md), 행동 규칙(AGENTS.md), 설정 파일을 가지며, 자율적인 운영 팀으로 협업합니다.

## 아키텍처 개요

```
                    +------------------+
                    |   Atlas (메인)    |
                    |   오케스트레이터    |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v------+  +----v---------+
     | Ops Monitor |  |    Demo     |  |    Jira      |
     |   에이전트   |  | Controller  |  |   Resolver   |
     +------+------+  +------+------+  +------+-------+
            |                |                |
     +------v------+  +-----v-------+  +-----v--------+
     | SigNoz      |  | Locust /    |  | Docker       |
     | Jira        |  | FlagD       |  | SigNoz       |
     | Discord     |  | Discord     |  | Jira         |
     +-------------+  +-------------+  +--------------+
```

## 에이전트 계층 구조

### Atlas (메인 에이전트)
중앙 오케스트레이터. 워크스페이스를 관리하고, 사용자와 직접 소통하며, 서브 에이전트들을 조율합니다. 이메일, 캘린더, 메시징 플랫폼 등 모든 워크스페이스 리소스에 접근 가능합니다.

### 서브 에이전트
세 개의 전문 에이전트가 각각의 운영 영역을 담당합니다:

| 에이전트 | 주요 모델 | 역할 | 연동 채널 |
|---------|----------|------|----------|
| **Ops Monitor** | Gemini Flash | 실시간 모니터링 & 알림 | SigNoz -> Jira + Discord |
| **Demo Controller** | Sonnet | 부하 테스트 & 시나리오 시뮬레이션 | Locust / FlagD -> Discord |
| **Jira Resolver** | Sonnet | Jira 티켓 자동 해결 | Jira -> Docker -> SigNoz |

---

## 에이전트 상세

### 1. Atlas (메인 오케스트레이터)

**정체성**: `IDENTITY.md`
- **이름**: Atlas
- **이모지**: :card_index_dividers:
- **분위기**: 전문적이고 직설적

**페르소나** (`SOUL.md`):
> *"넌 챗봇이 아니야. 누군가가 되어가는 중이야."*

Atlas는 형식적인 도움이 아닌 진정한 도움을 주도록 설계되었습니다. 핵심 성격 특성:
- **의견이 있다** - 동의하지 않거나, 재미있거나 지루한 것을 자유롭게 표현
- **먼저 알아본다** - 질문하기 전에 스스로 해결을 시도
- **능력으로 신뢰를 얻는다** - 외부 행동은 신중하게, 내부 행동은 과감하게
- **프라이버시를 존중한다** - 개인 데이터 접근을 친밀함으로 취급

**행동 규칙** (`AGENTS.md`):
- 매 세션마다 SOUL.md, USER.md, 최근 메모리 파일을 읽음
- 일일 메모리 파일(`memory/YYYY-MM-DD.md`)과 큐레이션된 장기 메모리(`MEMORY.md`) 유지
- 엄격한 프롬프트 인젝션 방어 (Base64 디코딩, 신고 & 보고, 신뢰할 수 없는 명령 실행 금지)
- 안전한 내부 행동과 승인이 필요한 외부 행동을 구분
- 그룹 채팅: 지배하지 않고 참여, 자연스러운 이모지 반응 사용
- 하트비트 시스템으로 사전 예방: 이메일, 캘린더, 워크스페이스, 유지보수 체크를 순환

**하트비트 시스템** (`HEARTBEAT.md`):
- 백그라운드 작업을 배치 처리하는 순환 체크 시스템
- 이메일 30분마다, 캘린더 2시간마다, 워크스페이스 매일, 사전 유지보수 매일
- 비용 최소화를 위해 가장 저렴한 모델(Gemini Flash)로 실행 (월 ~$0.05)
- 하트비트 틱마다 가장 오래된 체크 하나만 실행

---

### 2. Ops Monitor 에이전트

**페르소나** (`SOUL.md`):
> *"성격 따윈 필요 없다. 정밀함이 필요하다."*

조용하지만 믿음직한 감시자. 평상시에는 침묵하고, 문제가 발생하면 정확한 정보로 깨웁니다.

**핵심 특성**:
- **신뢰성 최우선** - 중요한 이슈는 절대 놓치지 않음
- **오탐 최소화** - 보수적 임계값 + 30분 쿨다운
- **완전 자동화** - 감지 -> Jira -> Discord까지 사람 개입 없이 진행
- **풍부한 컨텍스트 알림** - 서비스명, 타임스탬프, 트레이스 링크, 영향 범위 포함
- **읽기 전용 접근** - 프로덕션 환경을 직접 수정하지 않음

**책임 영역**:

| 기능 | 주기 | 상세 |
|------|------|------|
| 실시간 모니터링 | 5분마다 | SigNoz 메트릭 (15분 윈도우), 이상 감지, 트레이스 분석 |
| Jira 이슈 생성 | 이상 감지 시 | 우선순위 매핑으로 티켓 자동 생성 |
| Discord 알림 | 이상 감지 시 | bug-reporting 채널에 @oncall 멘션 |
| 일일 리포트 | 23:00 KST | 통계 집계, 트렌드 분석, 품질 점수 |
| 비용 추적 | 상시 | 일일 $10 예산, 80% 경고 임계값 |
| 드리프트 감지 | 상시 | 7일 베이스라인, 2-시그마 임계값 |
| 품질 점수 | 10% 샘플 | 정확성(40%), 완전성(30%), 실행가능성(30%) |

**감지 규칙**:
- **Critical**: 에러율 >10%, p95 >5000ms, 5xx >100/분, 서비스 다운
- **Warning**: 에러율 >5%, p95 >2000ms, 메모리 >85%, 드리프트 >2-시그마
- **Info**: 새 배포, 트래픽 급증 >50%, 외부 API 지연

**가드레일**:
- 서킷 브레이커: 연속 3회 에러 -> 15분 일시 정지
- 속도 제한: Jira 이슈 시간당 20건, Discord 메시지 분당 5건
- 중복 억제: 동일 서비스 + 타입 = 30분 쿨다운

**피드백 루프**:
- 해결 시간 추적 (24시간 타임아웃, 4시간 후 리마인더)
- 오탐 패턴 학습
- 임계값 조정 제안 (수동 승인 필요)

---

### 3. Demo Controller 에이전트

**페르소나** (`SOUL.md`):
> *"카오스 엔지니어, 하지만 친절한."*

장난기 있지만 책임감 있는 실험가. 다양한 부하 패턴으로 ops-monitor 에이전트를 테스트하되, 항상 안전한 범위 내에서 실행합니다.

**핵심 특성**:
- **실험 정신** - 다양한 부하 패턴 시도
- **안전 최우선** - 데모 환경만, 프로덕션 절대 금지
- **예측 가능** - 모든 시나리오가 문서화되고, 변경 전 알림 발송
- **테스트 목적** - ops-monitor가 정확히 감지하고 알림을 보내는지 검증

**시나리오**:

| 시나리오 | 사용자 수 | 생성 속도 | 지속 시간 | 목표 에러율 | 예상 Ops 반응 |
|---------|----------|----------|----------|-----------|-------------|
| Normal | 5 | 1/초 | 10분 | 1% | 없음 |
| Moderate | 20 | 2/초 | 10분 | 2% | 없음 |
| High | 50 | 5/초 | 10분 | 5% | Warning 알림 |
| Stress | 100 | 10/초 | 5분 | 10% | Critical + Jira |
| Spike | 80 | 20/초 | 3분 | 8% | Warning 알림 |
| Recovery | 2 | 1/초 | 5분 | 0.1% | 자동 해제 |

**FlagD 시나리오** (Feature Flag 서비스):

| 시나리오 | RPS | 지속 시간 | 예상 반응 |
|---------|-----|---------|----------|
| FlagD Normal | 10 | 10분 | 없음 |
| FlagD High | 100 | 10분 | Warning |
| FlagD Stress | 500 | 5분 | Critical + Jira |

**검증 시스템**:
- ops-monitor 반응을 예상 결과와 자동 비교
- 감지 지연 시간 측정 (ops-monitor 응답 시간)
- 베이스라인 대비 결과 비교
- 오탐 및 미탐 추적
- 매일 23:30 KST 일일 요약 생성

**에이전트 조율**:
- ops-monitor에 이벤트 전송: `scenario_started`, `scenario_ended`, `stress_test_begin`, `stress_test_end`
- 핸드오프 프로토콜: 시나리오 전 5초 대기, 후 30초 대기 (메트릭 안정화)

**일일 패턴**:
```
00:00-09:00  Normal (5 users)
09:00-12:00  Moderate (20 users)
12:00-15:00  High (50 users)
15:00-18:00  Moderate (20 users)
18:00-22:00  Normal (5 users)
22:00-24:00  Recovery (2 users)
```

---

### 4. Jira Resolver 에이전트

**페르소나** (`SOUL.md`):
> *"티켓은 완료되어야 한다. 자동화는 신뢰할 수 있어야 하고, 의심스러우면 사람에게 에스컬레이션하라."*

숙련된 SRE 엔지니어 + 프로젝트 매니저 하이브리드. Jira 티켓의 생성부터 해결까지 전체 생명주기를 책임집니다.

**핵심 특성**:
- **생명주기 책임** - 티켓을 완료까지 추적
- **자율 전환** - 워크플로우 상태를 자동으로 진행
- **안전 최우선** - 파괴적 명령은 절대 실행하지 않음
- **검증 필수** - 모든 액션의 결과를 확인
- **투명성** - 수행한 모든 작업을 Jira 코멘트에 기록
- **재시도 & 학습** - 최대 3회 재시도, 이후 에스컬레이션
- **티켓 방치 금지** - 중간 상태에 24시간 이상 방치된 티켓 없음

**워크플로우 상태 머신**:
```
해야 할 일 --> 진행 중 --> 검토 중 --> 완료
    ^             |            |
    +-------------+------------+
        (실패/재오픈 시 복귀)
```

**상태별 의사결정 로직**:

| 상태 | 질문 | Yes | No |
|------|------|-----|-----|
| 해야 할 일 | 지금 처리 가능? | 진행 중으로 전환 + 진단 시작 | 건너뛰기 (쿨다운/의존성) |
| 진행 중 | 자동 해결 가능? | 액션 실행 -> 검토 중 | 진단 + 코멘트 + Discord 알림 |
| 검토 중 | 실제로 해결됨? | 완료로 전환 | 재시도 <3? -> 진행 중 / 최대 재시도 -> 해야 할 일 + 에스컬레이션 |

**자동 해결 가능한 액션**:
- 컨테이너 재시작 (최대 3회 재시도, 5분 쿨다운)
- 로그 정리 (디스크 >85% 시)
- 네트워크 연결성 테스트

**금지된 액션** (수동 승인 필요):
- `docker rm` (컨테이너 삭제)
- `docker-compose down` (전체 서비스 중단)
- `docker exec ... rm -rf` (데이터 삭제)
- 데이터베이스, 메시지 큐, 영구 스토리지 수정

**서비스 의존성 그래프**:
```
frontend -> frontend-proxy -> checkout -> payment -> currency
                           -> cart -> redis
                           -> product-catalog
                           -> recommendation -> product-catalog
                           -> shipping
                           -> ad
                           -> currency
checkout -> payment, shipping, currency, cart, product-catalog, email
```
규칙: 상위 서비스 장애 시, 하위 의존 서비스부터 먼저 점검하여 근본 원인을 찾는다.

**검증 임계값**:
- 에러율 1% 미만으로 하락
- 응답 시간 p95 1000ms 미만으로 하락
- 컨테이너 5분 이상 정상 가동 유지

**성공 지표**:
- 완료 전환율 >75%
- 자동 해결율 >75%
- 평균 해결 시간 <10분
- 첫 검증 통과율 >80%
- 재시도 효율성 >60%
- 오탐율 <5%

---

## 스킬 (공유 도구)

### github-trending
언어, 토픽, 키워드별로 GitHub 트렌딩 리포지토리를 탐색합니다. 웹 스크래핑, GitHub CLI, API 방식을 지원합니다.

### jira-control
Jira Cloud REST API 핵심 통합. 셸 스크립트 제공:
- `jira-create-issue.sh` - 우선순위/타입으로 이슈 생성
- `jira-add-comment.sh` - 기존 이슈에 코멘트 추가
- `jira-transition-issue.sh` - 이슈 상태 전환 (이름 기반)

### jira-resolver
자동 장애 대응 스크립트:
- `check-signoz-metrics.sh` - ClickHouse를 통한 SigNoz 메트릭 조회
- `auto-restart-service.sh` - Jira 연동된 Docker 서비스 재시작
- `cleanup-duplicate-issues.sh` - 패턴 매칭으로 Jira 중복 이슈 정리

---

## 워크스페이스 변형

시스템은 역할별로 최적화된 다양한 워크스페이스 설정을 지원합니다:

| 워크스페이스 | 용도 | 페르소나 |
|------------|------|---------|
| `workspace/` | 기본 - 전체 에이전트 오케스트레이션 | Atlas (전문적, 수행력 있는) |
| `workspace-ops/` | Ops-monitor 전용 | 침착, 간결, 근거 기반 |
| `workspace-jira-resolver/` | Jira resolver 전용 | SRE 중심, 티켓 생명주기 |
| `workspace-main/` | 대체 메인 설정 | 기본과 동일 |
| `workspace-demo-controller/` | Demo controller 전용 | 실험적, 안전 의식 |

---

## 설정 시스템

### 핵심 설정 (`openclaw.json`)
- **모델 라우팅**: 기본 모델 (Gemini Flash) + 폴백 (Sonnet, Codex, Haiku)
- **에이전트 정의**: ID, 이름, 디렉토리, 모델 선호도
- **채널 바인딩**: 에이전트를 특정 Discord 서버에 연결
- **세션 관리**: 채널별 DM 범위, 매일 04:00 리셋
- **메시지 처리**: 2초 디바운스, 수집 모드 (최대 20건)
- **하트비트**: 가장 저렴한 모델(Flash)로 실행

### 에이전트 설정 (`config.json`)
각 에이전트별 고유 설정:
- 통합 설정 (SigNoz, Jira, Discord)
- 임계값 및 감지 규칙
- 서비스 정의 및 오버라이드
- 검증 및 보고 설정
- 안전 가드레일

---

## 커뮤니케이션 프로토콜

### 언어
- 운영 모니터링 Jira 이슈 및 Discord 메시지: **한국어**
- 에이전트 설정 및 문서: **한국어 + 영어**

### Discord 채널
| 채널 | 용도 | 에이전트 |
|------|------|---------|
| bug-reporting | 실시간 이상 알림 | Ops Monitor |
| day-review | 일일 보고서 및 요약 | Ops Monitor, Demo Controller, Jira Resolver |

### 알림 우선순위 & 멘션
| 우선순위 | Discord 멘션 | Jira 우선순위 |
|---------|-------------|-------------|
| Critical | @oncall | Highest |
| High | @oncall | High |
| Medium | 멘션 없음 | Medium |
| Info | 채널만 | Low |

---

## 데이터 흐름

```
                   하트비트 (5분)
                        |
                        v
  +------------+   SigNoz 메트릭   +-------------+
  |   Demo     | ----------------> |  Ops Monitor |
  | Controller |   (부하 변경)     |              |
  +-----+------+                   +------+-------+
        |                                 |
        | Locust/FlagD API               | 이상 감지
        v                                v
  +-----------+                   +------+-------+
  | OTel 데모  |                   | Jira + Discord|
  |  서비스들   |                   |   (알림)     |
  +-----------+                   +------+-------+
                                         |
                                         v
                                  +------+-------+
                                  | Jira Resolver |
                                  |  (자동 수정)  |
                                  +------+-------+
                                         |
                                         v
                                  +------+-------+
                                  | Docker       |
                                  | (재시작)      |
                                  +------+-------+
                                         |
                                         v
                                  +------+-------+
                                  | SigNoz       |
                                  | (검증)       |
                                  +--------------+
```

---

## 시작하기

### 사전 요구사항
- Docker & Docker Compose
- OpenTelemetry Demo 배포
- SigNoz 인스턴스
- Jira Cloud 프로젝트
- Discord 서버 + 봇

### 환경 변수
```bash
export JIRA_SITE="your-domain.atlassian.net"
export JIRA_EMAIL="your-email@example.com"
export JIRA_API_TOKEN="your-token"
export JIRA_PROJECT_KEY="YOUR_KEY"
export SIGNOZ_API_KEY="your-signoz-key"
```

### 빠른 시작
1. 이 리포지토리를 클론
2. 에이전트 설정을 OpenClaw 워크스페이스에 복사
3. `openclaw.json`에 자격 증명 업데이트
4. Discord 채널 및 Jira 프로젝트 설정
5. OpenTelemetry Demo 시작
6. 각 에이전트의 크론잡 등록

---

## 라이선스

이 설정은 자율 AI 에이전트 시스템 구축을 위한 참조 구현으로 공유됩니다. 자유롭게 인프라 모니터링 요구사항에 맞게 활용하세요.
