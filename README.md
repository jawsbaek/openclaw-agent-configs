> **Language**: English | [한국어](README.ko.md)

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

## Configuration Separation

The configuration follows a clear separation between **general agent framework** and **project-specific settings**, using OpenClaw's native `$include` directive and `${VAR}` environment variable substitution.

### General (Reusable)
- `workspace/` — Agent personas (SOUL.md), behavioral rules (AGENTS.md), workflow patterns, guardrails, drift detection, quality scoring
- Agent `config.json` files contain framework-level settings that work across any project

### Project-Specific (Override)
- `projects/otel-demo/` — Connection info (SigNoz, Jira, Discord), service definitions, threshold tuning, load test scenarios
- `.env` — Secrets (API tokens, bot tokens) referenced via `${VAR}` substitution
- `config/` — Channel bindings and routing rules specific to the deployment

### Adding a New Project
1. Create `projects/<new-project>/` with 4 files:
   - `integrations.json5` — Observability, ticket system, notification connections
   - `services.json5` — Service list and dependency graph
   - `thresholds.json5` — Detection thresholds tuned for the environment
   - `scenarios.json5` — Load test scenarios (if using Demo Controller)
2. Update `$include` paths in agent `config.json` files
3. Add secrets to `.env`

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
- **Channel bindings**: Agents bound to specific Discord guilds (using `$include` from `config/`)
- **Session management**: Per-channel DM scope, daily reset at 04:00
- **Message handling**: 2-second debounce, collect mode (cap: 20)
- **Heartbeat**: Runs on cheapest model (Flash)

### Agent Configs (`config.json`)
Each agent has its own config with:
- Framework-level settings (personas, behavioral rules, workflow patterns)
- Project-specific settings loaded via `$include` from `projects/<project-name>/`
- Environment variables referenced via `${VAR}` substitution (e.g., `${JIRA_API_TOKEN}`)
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
Copy `.env.example` to `.env` and fill in your secrets:
```bash
cp .env.example .env
# Edit .env with your actual credentials
```

**Important**: All secrets are managed via the `.env` file and referenced in configs using `${VAR}` syntax. Never commit secrets directly to config files.

### Quick Start
1. Clone this repository
2. Copy `.env.example` to `.env` and fill in your credentials
3. Copy agent configs to your OpenClaw workspace
4. Configure Discord channels in `config/channels.json5`
5. Update project-specific settings in `projects/otel-demo/`
6. Start OpenTelemetry Demo
7. Register cron jobs for each agent

---

## File Structure

```
openclaw-agent-configs/
├── .env.example                           # Environment variable template
├── .gitignore                             # Git ignore rules
├── openclaw.json                          # Main config (uses $include)
├── README.md                              # English documentation
├── README.ko.md                           # Korean documentation
├── config/                                # ── Modular config (from openclaw.json) ──
│   ├── agents.json5                       # Agent definitions & model routing
│   ├── channels.json5                     # Discord/iMessage channel settings
│   └── bindings.json5                     # Agent-channel routing rules
├── projects/                              # ── Project-specific overrides ──
│   └── otel-demo/                         # Current project (OTel Demo)
│       ├── integrations.json5             # SigNoz, Jira, Discord connections
│       ├── services.json5                 # 14 services + dependency graph
│       ├── thresholds.json5               # Threshold tuning per agent
│       └── scenarios.json5                # Demo Controller load scenarios
├── workspace/                             # ── General agent framework ──
│   ├── AGENTS.md                          # Main agent behavioral rules
│   ├── SOUL.md                            # Main agent persona
│   ├── TOOLS.md                           # Local tool notes
│   ├── IDENTITY.md                        # Agent identity
│   ├── HEARTBEAT.md                       # Heartbeat system
│   ├── WORKSPACE.md                       # Workspace structure guide
│   └── agents/
│       ├── ops-monitor/
│       │   ├── AGENT.md                   # Full agent definition
│       │   ├── AGENTS.md                  # Session rules
│       │   ├── SOUL.md                    # Persona
│       │   ├── README.md                  # Setup guide
│       │   └── config.json                # Config ($include → projects/)
│       ├── demo-controller/
│       │   ├── AGENT.md / AGENTS.md / SOUL.md / README.md
│       │   └── config.json                # Config ($include → projects/)
│       └── jira-resolver/
│           ├── AGENTS.md / SOUL.md
│           └── config.json                # Config ($include → projects/)
├── workspace-jira-resolver/               # Jira resolver workspace
├── workspace-ops/                         # Ops workspace
└── skills/                                # Shared tools
    ├── github-trending/
    ├── jira-control/
    └── jira-resolver/
```

---

## License

This configuration is shared as a reference implementation for building autonomous AI agent systems. Feel free to adapt it for your own infrastructure monitoring needs.
