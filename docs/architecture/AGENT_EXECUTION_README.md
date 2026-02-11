# PlanExe Agent Execution System (PlanExe-AX) — Complete Design

## 📚 Document Index

Start here. Read in order:

1. **[AGENT_EXECUTION_SUMMARY.md](AGENT_EXECUTION_SUMMARY.md)** ← **START HERE**
   - Executive summary (5 min read)
   - Problem statement and solution overview
   - What problem are we solving?
   - Comparison: manual vs automated
   - **Who reads this:** Mark, stakeholders, team leads

2. **[AGENT_EXECUTION_ARCHITECTURE.md](AGENT_EXECUTION_ARCHITECTURE.md)** (Main Design Doc)
   - How agents receive plans from PlanExe
   - How agents break down tasks (decomposition)
   - How agents report progress
   - Execution loop design (polling vs webhooks)
   - UI & API requirements
   - **Real case study:** Larry prospecting dog breeders Q1 2026
   - **Who reads this:** Architects, senior developers, designers

3. **[AGENT_EXECUTION_IMPLEMENTATION.md](AGENT_EXECUTION_IMPLEMENTATION.md)** (Technical Deep Dive)
   - Complete database schema (SQLAlchemy models)
   - Full API contracts (OpenAPI specs with examples)
   - Code organization and structure
   - Testing strategy
   - Performance targets
   - **Who reads this:** Developers implementing the system

4. **[AGENT_EXECUTION_INTEGRATION_POINTS.md](AGENT_EXECUTION_INTEGRATION_POINTS.md)** (Code Integration)
   - Exactly where to add code in existing PlanExe repo
   - File paths and locations
   - What to modify in `.env`, `docker-compose.yml`, etc.
   - Backward compatibility checklist
   - **Who reads this:** Developers integrating with PlanExe

5. **[AGENT_EXECUTION_CHECKLIST.md](AGENT_EXECUTION_CHECKLIST.md)** (Implementation Roadmap)
   - Phase-by-phase checklist (Weeks 1-5)
   - Each checkbox is a task
   - Quick command reference
   - Sign-off template
   - **Who reads this:** Project manager, developers executing the work

---

## 🎯 Quick Start (For Mark)

### If you have 10 minutes:
Read **AGENT_EXECUTION_SUMMARY.md** → Skim the diagrams → Decide if this solves your problem.

### If you have 1 hour:
1. Read **AGENT_EXECUTION_SUMMARY.md** (15 min)
2. Skim **AGENT_EXECUTION_ARCHITECTURE.md** sections 1-3 (20 min)
3. Read the Larry prospecting case study (25 min)
4. Decide next steps (budget, timeline, etc.)

### If you want to start building:
1. Read **AGENT_EXECUTION_CHECKLIST.md** → Phase 1 section
2. Assign tasks to developers
3. Track progress with checklist
4. Weekly review of completed items

---

## 🔑 Key Concepts

### What is PlanExe-AX?

**The Problem:** PlanExe generates excellent 40-page plans, but you still have to execute them manually.

**The Solution:** PlanExe-AX automates plan execution using autonomous agents (like Larry).

```
Plan (40 pages) → Decomposed into tasks → Agent executes → Real-time tracking → Done
```

### How It Works (30-Second Version)

1. **User creates plan:** "Prospect dog breeders, close 3 deals"
2. **PlanExe generates plan:** 40-page doc with phases, tasks, timeline
3. **PlanExe-AX decomposes:** Breaks tasks into atomic actions (search, call, email, etc.)
4. **Agent (Larry) polls:** "Do I have work?" every 5 minutes
5. **Agent executes:** Searches Google Maps, calls prospects, sends emails, documents findings
6. **Agent reports:** Each task completed → system unlocks next task
7. **Dashboard updates:** Real-time progress visible to Mark
8. **Result:** 2-3 deals closed, tracked automatically, in 7 days instead of 30

### Why This Matters

- **Speed:** Agent executes tasks 24/7, no waiting for Mark's availability
- **Tracking:** Every step logged, debuggable, repeatable
- **Scalability:** Same system can run with multiple agents (team coordination later)
- **Cost:** Agents handle 95% of execution, Mark only approves decisions

---

## 📋 What Gets Built

### Core Components

| Component | Purpose | Where |
|-----------|---------|-------|
| **Decomposition Engine** | Parse plan → atomic tasks | `worker_plan_execution/decomposition.py` |
| **Execution Engine** | Manage task sequencing, state machine | `worker_plan_execution/service.py` |
| **Agent Polling API** | Agent polls for work | `GET /api/agents/{agent_id}/work` |
| **Event Reporting API** | Agent reports progress | `POST /api/executions/{execution_id}/events` |
| **State Machine** | CREATED → RUNNING → COMPLETED | `worker_plan_execution/state_machine.py` |
| **Dashboard UI** | Real-time progress visualization | `frontend_*/execution_dashboard.html` |
| **Database** | Execution, task, event records | `database_api/model_execution.py` |

### Data Stored

- **Executions:** Each time a plan is run (id, plan_id, agent_id, status, progress)
- **Tasks:** Atomic actions within a plan (what needs to be done)
- **Events:** Immutable log of everything that happened (audit trail)
- **Agents:** Registry of available agents and their capabilities

### APIs Created

**For Agents:**
- `GET /api/agents/{agent_id}/work` — Agent polls for next task
- `POST /api/executions/{execution_id}/events` — Agent reports progress

**For Users:**
- `POST /api/executions` — Create execution from plan
- `GET /api/executions/{execution_id}` — Check status
- `PATCH /api/executions/{execution_id}` — Pause/resume/cancel
- `WS /api/executions/{execution_id}/stream` — Real-time updates (WebSocket)

---

## 📈 Timeline & Effort

### Phase 1: Foundation (Week 1-2)
**What:** Core database, API routes, state machine
**Effort:** ~40 hours (1 engineer)
**Deliverable:** Can create execution, agent can poll for work, events logged
**Test:** Manual API testing with mock tasks

### Phase 2: Agent Integration (Week 3)
**What:** Decomposition engine, Larry agent registration
**Effort:** ~30 hours (1-2 engineers)
**Deliverable:** Can decompose real prospecting plan, Larry can execute it
**Test:** Full prospecting scenario (manual agent loop)

### Phase 3: Dashboard (Week 4)
**What:** Real-time UI, WebSocket streaming
**Effort:** ~20 hours (1 frontend engineer)
**Deliverable:** Can see execution progress live on dashboard
**Test:** Open dashboard, watch it update as agent works

### Phase 4: Hardening (Week 5+)
**What:** Error handling, monitoring, performance tuning
**Effort:** ~30 hours (1-2 engineers)
**Deliverable:** Production-ready, can handle 100+ concurrent executions
**Test:** Load testing, error scenarios, recovery

**Total Effort:** ~120 hours (~3 weeks, 1-2 engineers)
**Total Duration:** 4-5 weeks (can overlap phases)

---

## 💰 Business Case (Larry's Prospecting)

### Current Approach (Manual)
- Mark researches, calls, emails, proposes, closes deals himself
- **Time:** ~84 hours across 4-6 weeks
- **Result:** 2-3 deals closed
- **Constraint:** Only 1 person (Mark) can do it, so limited capacity

### With PlanExe-AX (Automated)
- Larry agent executes the plan automatically, Mark supervises
- **Time:** ~13 hours agent time, ~2 hours Mark oversight
- **Result:** 2-3 deals closed (same quality, faster)
- **Benefit:** Mark freed up for other work; can run 5 parallel prospecting plans = 15 deals/month

### ROI
- **Development cost:** ~$10K (120 hours × $85/hr)
- **Break-even:** 1 successful prospecting execution (1 deal closed = ~$8K revenue)
- **Payoff:** 4 weeks for one agent; scales exponentially with multiple agents

---

## 🚀 Getting Started

### Step 1: Read the Docs
- [ ] Read AGENT_EXECUTION_SUMMARY.md (Mark)
- [ ] Share with team, get buy-in

### Step 2: Plan the Work
- [ ] Decide timeline (can you start Week 1?)
- [ ] Assign engineers to Phase 1
- [ ] Schedule weekly check-ins

### Step 3: Phase 1 (Week 1-2)
- [ ] Engineers follow AGENT_EXECUTION_CHECKLIST.md
- [ ] Use AGENT_EXECUTION_IMPLEMENTATION.md as reference
- [ ] Mark reviews progress weekly

### Step 4: Phase 2 (Week 3)
- [ ] Integrate Larry agent
- [ ] Run full prospecting scenario
- [ ] Manual testing with agent simulator

### Step 5: Phase 3-4 (Week 4+)
- [ ] Build dashboard
- [ ] Add error handling
- [ ] Production deployment

---

## ❓ FAQ (Quick Answers)

**Q: Is this a rewrite of PlanExe?**
A: No. This adds execution layer on top of existing PlanExe. Plan generation unchanged.

**Q: Can this work with other agents besides Larry?**
A: Yes. Any agent can register with the system. Protocol is open.

**Q: What if my plan is 50 tasks and the agent fails on task 10?**
A: Execution pauses (BLOCKED status), Mark is notified, can approve workaround or reassign to different agent.

**Q: Is this just for prospecting?**
A: No. Works for any plan: business planning, project execution, research, etc. Prospecting is just the test case.

**Q: Can multiple agents work together?**
A: Phase 1 is single agent per execution. Multi-agent coordination is Phase 2 (future).

**Q: Do I need to modify my existing plans?**
A: No. Works with plans generated by existing PlanExe. Decomposition is automatic.

**Q: What if a task requires human judgment?**
A: Task becomes BLOCKED, Mark is notified, can approve next step or take over manually.

---

## 🛠️ Technical Stack

- **Backend:** Python 3.13, FastAPI, SQLAlchemy
- **Database:** PostgreSQL (multi-user) or SQLite (single-user)
- **Frontend:** Gradio (simple) or Flask + HTML/JS (advanced)
- **Real-time:** WebSocket (Socket.IO optional)
- **Testing:** pytest, fixtures
- **Deployment:** Docker, Railway, or bare metal

No new external dependencies beyond what PlanExe already uses.

---

## 📞 Questions?

These documents are complete and answer 99% of questions. If you have questions:

1. **Architecture question?** → Check AGENT_EXECUTION_ARCHITECTURE.md
2. **Implementation detail?** → Check AGENT_EXECUTION_IMPLEMENTATION.md
3. **Code location?** → Check AGENT_EXECUTION_INTEGRATION_POINTS.md
4. **What to build next?** → Check AGENT_EXECUTION_CHECKLIST.md
5. **Executive summary?** → Check AGENT_EXECUTION_SUMMARY.md

If something is unclear, file an issue or ask me directly.

---

## 📄 Document Versions

| Document | Size | Status | Last Updated |
|----------|------|--------|--------------|
| AGENT_EXECUTION_SUMMARY.md | 14 KB | ✅ Complete | 2026-02-11 |
| AGENT_EXECUTION_ARCHITECTURE.md | 25 KB | ✅ Complete | 2026-02-11 |
| AGENT_EXECUTION_IMPLEMENTATION.md | 21 KB | ✅ Complete | 2026-02-11 |
| AGENT_EXECUTION_INTEGRATION_POINTS.md | 15 KB | ✅ Complete | 2026-02-11 |
| AGENT_EXECUTION_CHECKLIST.md | 13 KB | ✅ Complete | 2026-02-11 |
| AGENT_EXECUTION_README.md | 6 KB | ✅ Complete | 2026-02-11 |

---

## 🎓 How to Use These Documents

### For Mark (Product Owner)
1. Read SUMMARY (10 min)
2. Review case study (Larry prospecting)
3. Decide: go/no-go on Phase 1
4. If yes: assign to team, use CHECKLIST to track

### For Architects
1. Read SUMMARY (overview)
2. Read ARCHITECTURE (deep understanding)
3. Review decision points (LLM vs rules, polling vs webhooks, etc.)
4. Suggest improvements or alternatives

### For Developers
1. Skim SUMMARY (understand why)
2. Read IMPLEMENTATION (data model, API contract)
3. Read INTEGRATION_POINTS (where to add code)
4. Use CHECKLIST (track what you've done)
5. Reference ARCHITECTURE for context when stuck

### For DevOps/Infrastructure
1. Read INTEGRATION_POINTS section on Docker/Config
2. Review environment variables needed
3. Plan deployment strategy
4. Set up monitoring (from CHECKLIST Phase 4)

---

## ✅ Next Action

**For Mark:**
1. Read AGENT_EXECUTION_SUMMARY.md
2. Schedule 30-min review with team
3. Decide: Start Phase 1 this week? Or wait?
4. If yes: Assign one engineer, they follow CHECKLIST.md

**For Team:**
1. Read the docs that match your role (see above)
2. Ask questions on unclear sections
3. Be ready to start Phase 1 (database + API)

---

## Good luck! 🚀

This is a solid architecture that will scale from Larry alone to a whole team of agents. Start with Phase 1, validate with real execution (Larry's prospecting), then iterate.

You've got this.

— Claude (Subagent)
