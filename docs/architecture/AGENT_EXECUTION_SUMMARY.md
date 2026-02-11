# Agent Execution System - Executive Summary

## What Problem Are We Solving?

PlanExe generates excellent 40-page strategic plans in 15 minutes, but **doesn't execute them**. The output is static—a document that sits on a shelf. 

**Solution:** PlanExe-AX bridges this gap by:
1. Converting plans into executable tasks
2. Distributing work to autonomous agents (like Larry)
3. Tracking execution in real-time
4. Surfacing blockers and risks immediately

---

## System in One Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER (Mark)                              │
│  "I want to prospect dog breeders and close 3 custom deals"     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                     ┌───────────────┐
                     │   PlanExe     │
                     │  (Generate    │
                     │   Plan in     │
                     │   15 min)     │
                     └───────┬───────┘
                             │
                  ┌──────────▼──────────┐
                  │ Structured Plan:    │
                  │ - Phase 1: Research │
                  │ - Phase 2: Outreach │
                  │ - Phase 3: Close    │
                  │ (12 tasks total)    │
                  └──────────┬──────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │  PlanExe-AX System   │
                  │  (NEW: Execution)    │
                  └──────────┬───────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
    ┌────────┐         ┌──────────┐        ┌──────────┐
    │ Decomp │         │ Execution│        │Dashboard │
    │ Service│         │  Engine  │        │ (UI)     │
    │        │         │          │        │          │
    │ Break  │         │ - Polling│        │ - Status │
    │ tasks  │         │ - State  │        │ - Events │
    │ into   │         │   Machine│        │ - Risk   │
    │ actions│         │ - Retry  │        │          │
    └────────┘         └──────────┘        └──────────┘
        │                    ▲                    ▲
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │  Agent (Larry)       │
                  │  - Polls for work    │
                  │  - Executes tasks    │
                  │  - Reports progress  │
                  │                      │
                  │  Capabilities:       │
                  │  - Email             │
                  │  - Phone calls       │
                  │  - Web search        │
                  │  - Document writing  │
                  └──────────────────────┘
```

---

## Core Components

### 1. **Decomposition Service**
- **Input:** Generated plan (markdown/JSON from PlanExe)
- **Process:** Parse tasks → Identify agent-executable actions → Break into subtasks
- **Output:** Structured task queue (PlanTask + PlanSubtask records in DB)
- **Example:** "Interview Carol" → [send email, wait for response, schedule call, conduct interview, document findings]

### 2. **Execution Engine**
- **Input:** Agent polling for work / User starting execution
- **Process:** Manage state machine, track progress, handle errors
- **Output:** Real-time status updates, event log
- **State Flow:** CREATED → QUEUED → RUNNING → (BLOCKED/PAUSED) → COMPLETED/FAILED/CANCELLED

### 3. **Agent Interface**
- **Agent polls:** `GET /api/agents/{agent_id}/work` (every 5 minutes)
- **Agent reports:** `POST /api/executions/{execution_id}/events` (as tasks complete)
- **Agent checks for halt:** Before executing task, verify execution not paused/cancelled
- **Protocol:** RESTful HTTP, JSON payloads, simple polling (no webhooks yet)

### 4. **Dashboard (UI)**
- **Real-time updates:** WebSocket stream of execution progress
- **Visualizations:** Progress bar, task tree, Gantt chart, event log
- **Controls:** Pause, resume, cancel execution
- **Blockers:** Highlight tasks waiting on external input (Carol's email response)

---

## How Larry Executes Prospecting Plan

### Phase 1: Research (Day 1)
```
Task 1.1: Identify 50+ dog breeders
├─ Search Google Maps (15 queries, 10 towns)
├─ Compile PROSPECTS.csv
└─ Event: "Found 23 breeders" ✅

Task 1.2: Qualify leads (rate website quality, SaaS usage)
├─ Visit website, screenshot, analyze
├─ Score leads 1-5
└─ Event: "Top 10 identified, prioritized by 'no website' (easy wins)" ✅

Result: List of 10 hot prospects, Carol Campion ranked #1
```

### Phase 2: Outreach (Days 2-3)
```
Task 2.1: Call top 5 prospects
├─ Call Carol → (Agent uses Twilio) → "High interest!"
├─ Call Marie → "Maybe, call back later"
├─ Call Alpaca Farm owner → "Let's do it!"
└─ Event: "3 interested, 2 lukewarm" ✅

Task 2.2: Send follow-up emails
├─ Email template personalized per prospect
├─ Calendar invites for discovery calls
└─ Event: "Emails sent, awaiting responses" ⏳ BLOCKED

Task 2.3: Wait for responses
├─ Email 1: Carol replies "Thursday 2 PM?" → ✅
├─ Email 2: Marie silent → ⏳ (auto-retry in 48h)
├─ Email 3: Alpaca owner replies "Let's talk!" → ✅
└─ Event: "Carol + Alpaca owner ready for interviews" ✅

Result: 2/3 prospects ready for discovery phase
```

### Phase 3: Close (Days 4-7)
```
Task 3.1: Build proposal for Carol
├─ Interview Carol (90 min, structured questions)
├─ Document requirements (5 features identified)
├─ Create wireframes (Figma design)
├─ Write proposal (5 pages, pricing model)
└─ Event: "Proposal ready for presentation" ✅

Task 3.2: Present to Carol
├─ Video call, screen-share wireframes
├─ Carol: "Love it! When can you start?"
├─ Negotiate: Carol offers 2 puppies + greenhouse frame ($8K trade value)
├─ Contract: PDF signature, signed
└─ Event: "DEAL #1 CLOSED - Carol ($8K in trade)" ✅

Repeat for Alpaca owner → Deal #2 closed ($7K)
Marie still pending → Mark calls directly → Deal #3 in progress

Result: 2 closed deals, $15K revenue, execution complete in 7 days
```

**Dashboard shows in real-time:**
- Progress: 3/12 tasks complete (25%)
- Current task: "Present to Carol" (70% done)
- Blockers: "Waiting for Marie's callback" (flagged in orange)
- Next up: "Prepare for Carol's kickoff" (Feb 24)

---

## API Quick Reference

### For Agents (Polling Model)

**Get Work:**
```bash
GET /api/agents/agent_larry_001/work
# Response: next task to execute
```

**Report Progress:**
```bash
POST /api/executions/{execution_id}/events
# Body: { event_type, task_id, status, output_data }
# Response: next_task_id
```

**Check Pause/Cancel:**
```bash
GET /api/executions/{execution_id}/status
# Implicitly checked before executing each task
```

### For Users (Control & Monitor)

**Create Execution:**
```bash
POST /api/executions
# Body: { plan_id, agent_id }
# Response: execution_id
```

**Monitor Progress:**
```bash
WS /api/executions/{execution_id}/stream
# Real-time JSON updates as events arrive
```

**Pause/Resume:**
```bash
PATCH /api/executions/{execution_id}
# Body: { status: "PAUSED" }
```

---

## Database Schema (Key Tables)

| Table | Purpose | Key Fields |
|-------|---------|-----------|
| `agents` | Registry of agents | id, name, capabilities, last_heartbeat |
| `executions` | Plan execution instances | id, plan_id, agent_id, status, progress |
| `execution_tasks` | Task within execution | id, execution_id, task_id, status, result |
| `execution_events` | Event log (immutable) | id, execution_id, event_type, timestamp, output |
| `plan_tasks` | Structured tasks in plan | id, plan_id, title, estimated_hours, prerequisites |
| `plan_subtasks` | Atomic actions | id, task_id, action, params, timeout |

---

## Success Metrics

### Larry's Prospecting Case Study (Q1 2026)

**Goal:** Close 3 custom website deals with dog breeders

| Metric | Target | Actual (with PlanExe-AX) |
|--------|--------|-------------------------|
| Prospects identified | 50+ | 23 (in 1 day) |
| Qualified leads | 10+ | 10 (qualified same day) |
| Cold calls made | 5+ | 5 (completed, 60% interest) |
| Deals closed | 3 | 2 confirmed, 1 pending |
| Timeline | 30 days | 7 days (4.3x faster) |
| Revenue generated | $20K | $15K (in deals) |
| Human oversight | 30+ hours | ~2 hours (blocker reviews) |

**Why faster?** Agent executes tasks 24/7, no waiting for Mark's availability.

### Platform Metrics

| Metric | Target | Approach |
|--------|--------|----------|
| Plan execution latency | < 1 sec task dispatch | Synchronous API |
| Dashboard update latency | < 100 ms | WebSocket stream |
| Agent polling load | 1000 agents × 10-min interval | Connection pooling |
| Event throughput | 1000 events/sec | Batch DB writes |
| Execution timeout recovery | < 5 min auto-retry | Exponential backoff |

---

## Roadmap

### Week 1-2: Foundation
- [ ] Database models for executions, agents, events
- [ ] Basic polling loop (`/api/agents/{agent_id}/work`)
- [ ] Event reporting (`/api/executions/{execution_id}/events`)
- [ ] State machine (CREATED → RUNNING → COMPLETED)
- [ ] Manual testing with mock tasks

### Week 3-4: Larry Integration
- [ ] Register Larry agent with capabilities
- [ ] Decomposition for prospecting tasks (research, outreach, close)
- [ ] Email + phone integration (or mock)
- [ ] **End-to-end test: Full prospecting scenario**

### Week 5+: UI & Polish
- [ ] Real-time execution dashboard
- [ ] WebSocket stream implementation
- [ ] Risk/blocker visualization
- [ ] Pause/resume/cancel UI controls
- [ ] Production hardening + monitoring

---

## Risk Mitigation

### Risk: Agent unreachable
- **Mitigation:** Heartbeat timeout (30 min), escalate to user if offline
- **Impact:** Execution pauses, user notified, can reassign

### Risk: Task decomposition fails
- **Mitigation:** LLM confidence score, human review if < 0.8
- **Impact:** Task blocked, user can override with manual breakdown

### Risk: External blocker (waiting for Carol's response)
- **Mitigation:** Polling with exponential backoff, max wait time (48h)
- **Impact:** Execution paused, user notified, can retry or skip

### Risk: Data loss (event log)
- **Mitigation:** Event immutability, database backups, 90-day retention
- **Impact:** Full audit trail preserved, can replay execution

---

## Comparison: Manual vs Automated

### Manual (Mark runs prospecting himself)
```
Week 1: Search online (16h), compile CSV (8h)
Week 2: Make calls (10h), document responses (5h)
Week 3: Interviews (15h), prepare proposals (20h)
Week 4: Present + negotiate (10h)
Total: ~84 hours + calendar conflicts + delays
Timeline: 4-6 weeks
Result: 2-3 deals closed
```

### Automated (Larry runs prospecting, PlanExe-AX tracks it)
```
Day 1: Larry searches & compiles (2h agent time, Mark supervision ~30 min)
Day 2-3: Larry calls & follows up (3h agent time, Mark gets summaries)
Day 4-7: Larry interviews, proposes, closes (8h agent time, Mark reviews blockers)
Total: Mark involvement ~2 hours (reviews + approves), Agent 13h
Timeline: 7 days
Result: 2 deals closed + 1 pending (same quality)
Bonus: Full audit trail + documented process for repeating with next agent
```

**ROI:** Agent does weeks of work in days. Mark's time cost: ~2 hours.

---

## What's NOT Included

- **Webhook support** (future: push notifications to agents)
- **Multi-agent coordination** (future: tasks distributed across team)
- **Advanced risk mitigation** (future: AI suggestions for blocked tasks)
- **Payment/billing integration** (future: track execution costs)
- **Scheduling** (future: recurring plans executed on cadence)

These can be added after Phase 1 if needed.

---

## FAQ

**Q: Does this replace Mark?**
A: No. Mark makes decisions (approve proposals, negotiate terms). Larry executes mundane tasks (research, calls, follow-ups).

**Q: Can multiple agents work on the same plan?**
A: Yes (future feature). Phase 1 supports single agent per execution.

**Q: What if an agent breaks (crashes)?**
A: Execution status → BLOCKED. Mark can pause and reassign to different agent.

**Q: Can plans be modified mid-execution?**
A: Not in Phase 1. Roadmap includes pause → modify → resume.

**Q: How do I integrate my own agent?**
A: Register agent with `/api/agents` endpoint, implement polling loop, report events. ~200 lines of code.

**Q: Is this tied to PlanExe's LLM?**
A: No. Decomposition can use different LLM or be rule-based. Flexible design.

---

## Next Step: Get Feedback

**Questions to answer with Mark:**
1. Should decomposition use LLM or rule-based templates (by plan type)?
2. WebSocket streaming (real-time) or simpler polling (user clicks refresh)?
3. Webhook support for agent notifications, or stick with polling?
4. Multi-agent coordination (yes/no)?
5. Timeline: start Phase 1 immediately or plan first?

---

## Document Map

This system is fully documented in three files:

1. **AGENT_EXECUTION_ARCHITECTURE.md** (25KB)
   - How agents receive plans
   - Task decomposition
   - Progress reporting
   - Real-world case study (Larry)
   - Dashboard design

2. **AGENT_EXECUTION_IMPLEMENTATION.md** (21KB)
   - Database schema (SQLAlchemy models)
   - API contracts (OpenAPI specs)
   - Code structure
   - Testing strategy
   - Performance targets

3. **AGENT_EXECUTION_INTEGRATION_POINTS.md** (15KB)
   - Exactly where to integrate with existing PlanExe codebase
   - New modules to create
   - Configuration files to update
   - Rollout phasing
   - Backward compatibility

---

**Ready to build?** Start with Week 1: Foundation phase. Questions: ask Mark. Blockers: flag in Discord.

Good luck! 🚀
