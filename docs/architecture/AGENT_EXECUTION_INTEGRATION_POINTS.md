# Agent Execution System - PlanExe Integration Points

## Overview

This document maps where the Agent Execution System (PlanExe-AX) integrates with the existing PlanExe codebase.

**Current PlanExe Architecture:**
- `worker_plan/`: FastAPI service that runs the plan generation pipeline
- `database_api/`: SQLAlchemy models for users, credit, payments, tasks
- `frontend_*`: UI frontends (single-user, multi-user)
- `mcp_local`/`mcp_cloud`: MCP interfaces for tool access

**Where to add PlanExe-AX:**
- New module: `worker_plan/worker_plan_execution/` (execution engine)
- New DB models in `database_api/` (execution, agent, event tables)
- New FastAPI routes in `worker_plan/app.py` (execution API)
- New frontend component: execution dashboard (within existing frontend)

---

## 1. Database Layer Integration

### Location: `database_api/`

**New file: `model_execution.py`**

Add SQLAlchemy models for:
- Agent
- Execution
- ExecutionTask
- ExecutionEvent
- ExecutionSubtask

These should follow the same pattern as existing models in `database_api/`:
- Use SQLAlchemy `declarative_base()`
- Include timestamp columns (`created_at`, `updated_at`)
- Define relationships clearly
- No business logic in models (only schema)

**Integration with existing `database_api/`:**
```python
# database_api/__init__.py
from .model_execution import Agent, Execution, ExecutionTask, ExecutionEvent

__all__ = [
    "Agent",
    "Execution",
    "ExecutionTask",
    "ExecutionEvent",
    # ... existing exports
]
```

**Database migration:**
```bash
cd worker_plan
alembic revision --autogenerate -m "Add agent execution models"
alembic upgrade head
```

---

## 2. FastAPI Routes Integration

### Location: `worker_plan/app.py`

**Where to add routes:**

The existing `worker_plan/app.py` has the `/api/plans/start` endpoint for plan generation. Add execution routes alongside:

```python
# worker_plan/app.py (existing code)
@app.post("/api/plans/start")
async def start_plan(request: StartRunRequest) -> StartRunResponse:
    """Existing: Generate a plan"""
    # ... existing implementation

# NEW ROUTES (add after existing endpoints)

@app.get("/api/agents/{agent_id}/work")
async def agent_get_work(agent_id: str) -> AgentWorkResponse:
    """Agent polls for work"""
    # Implementation in execution module

@app.post("/api/executions")
async def create_execution(request: CreateExecutionRequest) -> ExecutionResponse:
    """Create execution from existing plan"""
    # Implementation in execution module

@app.get("/api/executions/{execution_id}")
async def get_execution(execution_id: str) -> ExecutionResponse:
    """Get execution status"""
    # Implementation in execution module

@app.patch("/api/executions/{execution_id}")
async def update_execution(execution_id: str, request: UpdateExecutionRequest) -> ExecutionResponse:
    """Pause/resume/cancel execution"""
    # Implementation in execution module

@app.post("/api/executions/{execution_id}/events")
async def report_event(execution_id: str, request: ExecutionEventRequest) -> EventResponse:
    """Agent reports progress"""
    # Implementation in execution module

@app.get("/api/executions/{execution_id}/events")
async def get_execution_events(execution_id: str, limit: int = 100) -> list[ExecutionEventResponse]:
    """Get event history"""
    # Implementation in execution module

@app.websocket("/api/executions/{execution_id}/stream")
async def execution_stream(execution_id: str, websocket: WebSocket):
    """Real-time execution updates"""
    # Implementation in execution module
```

**Code organization:**

Create module structure:
```
worker_plan/
├── app.py (main FastAPI app)
├── worker_plan_execution/
│   ├── __init__.py
│   ├── routes.py (all execution endpoints)
│   ├── service.py (business logic)
│   ├── state_machine.py (execution state transitions)
│   └── decomposition.py (task decomposition logic)
├── worker_plan_api/ (existing)
└── worker_plan_internal/ (existing)
```

**In `app.py`, import and register routes:**
```python
from worker_plan_execution.routes import router as execution_router

app.include_router(execution_router, prefix="/api", tags=["executions"])
```

---

## 3. Existing Model Integration

### `model_taskitem.py` (existing)

The existing `PlanTask` in `model_taskitem.py` might already have task structure. Check if it needs extension for agent execution:

**Existing fields to preserve:**
```python
# From model_taskitem.py
- id
- title
- description
- parent_task_id
- estimated_duration
```

**New fields to add (extend, don't modify):**
```python
# Add to PlanTask if not already present
- agent_executable: bool = False  # Can agents execute this?
- decomposition_confidence: float = 0.0
- assignee_role: str = None  # "Business Analyst", "Developer"
- prerequisite_task_ids: list = []
```

**Approach:** If PlanExe tasks aren't structured for agents yet, create a `PlanTaskExecution` view that wraps `PlanTask` with execution-specific fields.

---

## 4. Worker Service Integration

### Location: `worker_plan/worker_plan_internal/`

The existing `plan.py` runs the plan generation pipeline. Don't modify it. Instead:

**New parallel system for execution:**

1. **Decomposition engine** (`worker_plan_execution/decomposition.py`)
   - Takes generated plan (markdown/JSON output)
   - Parses tasks and converts to `PlanTask` + `PlanSubtask` records
   - Computes decomposition confidence (task can be executed autonomously?)
   - Stores in database

2. **Execution scheduler** (`worker_plan_execution/service.py`)
   - Polls database for ready executions
   - Matches agents to tasks
   - Triggers agent notifications
   - Manages state transitions

3. **State machine** (`worker_plan_execution/state_machine.py`)
   - Validates execution state transitions
   - Emits events on transitions
   - Handles retry logic

**These should be lightweight, independent of the pipeline logic.**

---

## 5. Frontend Integration

### Location: `frontend_single_user/` or `frontend_multi_user/`

**No changes needed to existing plan generation UI.** Instead, add a new section:

**In single-user mode (Gradio):**
```python
# frontend_single_user/app.py (hypothetical)

import gradio as gr

def create_ui():
    with gr.Blocks() as app:
        # Existing plan generation UI
        with gr.Tab("Generate Plan"):
            # ... existing code ...
        
        # NEW: Execution management UI
        with gr.Tab("Execute Plan"):
            execution_id = gr.Textbox(label="Execution ID")
            get_status_btn = gr.Button("Get Status")
            status_output = gr.JSON(label="Execution Status")
            
            get_status_btn.click(
                get_execution_status,
                inputs=[execution_id],
                outputs=[status_output]
            )
            
            # Event log
            event_log = gr.Dataframe(label="Event Log")
            
            # Real-time updates (requires WebSocket)
            # Can be implemented with JavaScript + gr.HTML()
```

**In multi-user mode (Flask):**
```python
# frontend_multi_user/app.py (hypothetical)

@app.route("/executions/<execution_id>")
def show_execution(execution_id):
    execution = db.session.get(Execution, execution_id)
    events = db.session.query(ExecutionEvent).filter_by(execution_id=execution_id).order_by(ExecutionEvent.timestamp.desc())
    
    return render_template("execution_dashboard.html", 
                          execution=execution, 
                          events=events)
```

**Dashboard template (`templates/execution_dashboard.html`):**
- Progress bar (% complete)
- Task tree (hierarchical)
- Real-time event feed
- Control buttons (pause, resume, cancel)
- Risk/blocker panel

---

## 6. Configuration Integration

### Location: `.env` and `llm_config.json`

**New environment variables to add:**

```bash
# .env or .env.developer-example

# Execution settings
PLANEXE_EXECUTION_ENABLED=true
PLANEXE_AGENT_POLL_INTERVAL_SECONDS=300  # 5 minutes
PLANEXE_EXECUTION_TIMEOUT_HOURS=168  # 1 week default timeout
PLANEXE_DECOMPOSITION_MODEL=claude-3-sonnet  # LLM for task decomposition
PLANEXE_WEBSOCKET_ENABLED=true
PLANEXE_EVENT_LOG_RETENTION_DAYS=90
```

**In `planexe_dotenv.py`:**
```python
class PlanExeDotEnv(BaseSetting):
    # Existing fields...
    
    execution_enabled: bool = Field(default=False, env="PLANEXE_EXECUTION_ENABLED")
    agent_poll_interval_seconds: int = Field(default=300, env="PLANEXE_AGENT_POLL_INTERVAL_SECONDS")
    execution_timeout_hours: int = Field(default=168, env="PLANEXE_EXECUTION_TIMEOUT_HOURS")
    websocket_enabled: bool = Field(default=False, env="PLANEXE_WEBSOCKET_ENABLED")
```

---

## 7. Dependency Updates

### Location: `worker_plan/pyproject.toml`

Add dependencies:
```toml
[project]
dependencies = [
    # Existing...
    
    # New for execution
    "python-socketio>=5.10.0",  # WebSocket support
    "python-engineio>=4.8.0",   # WebSocket transport
    "pydantic>=2.0",  # Already present, but confirm
    "sqlalchemy>=2.0",  # Already present, but confirm
]
```

---

## 8. Testing Integration

### Location: `worker_plan/tests/` and `database_api/tests/`

**New test files:**

1. **`tests/test_execution_state_machine.py`**
   - Test valid/invalid transitions
   - Test event processing

2. **`tests/test_decomposition.py`**
   - Test task parsing from plan JSON
   - Test confidence scoring
   - Test subtask generation

3. **`tests/test_execution_routes.py`**
   - Test `/api/agents/{agent_id}/work`
   - Test `/api/executions` CRUD
   - Test event reporting
   - Test WebSocket stream

4. **`tests/test_execution_integration.py`**
   - End-to-end: create plan → decompose → execute → report

---

## 9. Documentation Integration

### Location: `docs/`

**New documentation files:**

1. **`docs/agent_execution.md`**
   - Architecture overview
   - Agent capability definition
   - Execution lifecycle

2. **`docs/agent_api_reference.md`**
   - OpenAPI spec for agent endpoints
   - Example requests/responses
   - Error codes

3. **`docs/agent_setup.md`**
   - How to register an agent
   - How to implement polling loop
   - How to report events

4. **`docs/agent_examples.md`**
   - Example: Larry prospecting agent
   - Example: Generic task execution agent
   - Example: Human-in-the-loop agent

---

## 10. Docker Integration

### Location: `docker-compose.yml` and `Dockerfile`

**Update `docker-compose.yml`:**

```yaml
services:
  # Existing services...
  
  # Execution system (optional, runs inside worker_plan if not separated)
  # No changes needed if execution runs in worker_plan service
  
  # If separating execution scheduler as standalone:
  execution_scheduler:
    build:
      context: .
      dockerfile: worker_plan/Dockerfile
      target: execution_scheduler
    environment:
      PLANEXE_EXECUTION_ENABLED: "true"
      DATABASE_URL: ${DATABASE_URL}
    depends_on:
      - postgres  # If using multi-user mode
    ports:
      - "8001:8001"  # Different port if needed
```

---

## 11. Sample Integration Flow

### Step 1: Plan Generation (Existing)
```
User: "Build custom website for dog breeders"
↓
/api/plans/start (existing)
↓
worker_plan_internal generates 40-page plan
↓
Output: plan_id = "plan_carol_breeder_20260211"
```

### Step 2: Task Decomposition (New)
```
PlanExe extracts tasks from generated plan
↓
decomposition.py parses into PlanTask + PlanSubtask
↓
Stores in database (Execution = CREATED)
↓
User clicks "Start Execution"
↓
Execution state = QUEUED
```

### Step 3: Agent Polling (New)
```
Larry agent polls: GET /api/agents/agent_larry_001/work?every 5 min
↓
Server returns: "Here's task_1_1_a - send email to Carol"
↓
Agent executes (send email)
↓
Agent reports: POST /api/executions/{execution_id}/events (success)
↓
Server transitions: ExecutionTask = COMPLETE
↓
Next task unlocked: task_1_1_b
```

### Step 4: Real-time Dashboard (New)
```
User opens dashboard for execution_id
↓
Browser opens WebSocket: WS /api/executions/{execution_id}/stream
↓
As agent reports events, server pushes to WebSocket
↓
Dashboard updates in real-time:
   - Progress bar advances
   - Task tree shows status changes
   - Event log appends new events
↓
When execution completes, dashboard shows summary
```

---

## 12. Error Handling Integration

### Error Scenarios

1. **Agent unreachable** (doesn't poll for 30 min)
   - Execution status → BLOCKED
   - Server waits, retries
   - After 24h: escalate to user

2. **Task decomposition fails** (agent doesn't understand task)
   - Event reports: error_type = "decomposition_failed"
   - human_review_required = true
   - User can override with manual decomposition

3. **Subtask timeout** (email never sent, after 24h retries)
   - Event reports: error_type = "timeout"
   - Retry policy kicks in (up to 3 attempts)
   - After final failure: escalate

4. **External blocker** (waiting for Carol's email response)
   - Event reports: status = "BLOCKED"
   - Execution waits (polling external email)
   - Can set max wait time (e.g., 48h), then escalate

---

## 13. Rollout Strategy

### Phase 1 (Week 1-2): Core Infrastructure
- [ ] Add database models (`database_api/model_execution.py`)
- [ ] Add basic routes (`worker_plan_execution/routes.py`)
- [ ] Add state machine (`worker_plan_execution/state_machine.py`)
- [ ] Test with manual API calls

### Phase 2 (Week 3): Larry Integration
- [ ] Register Larry agent
- [ ] Implement decomposition for prospecting tasks
- [ ] Test full prospecting scenario (manual)

### Phase 3 (Week 4): Dashboard
- [ ] Add execution dashboard UI
- [ ] Add WebSocket stream
- [ ] Test real-time updates

### Phase 4 (Week 5+): Production Hardening
- [ ] Load testing (100 concurrent executions)
- [ ] Error handling edge cases
- [ ] Monitoring and alerting
- [ ] Documentation finalization

---

## 14. Backward Compatibility

**Key principle:** Agent execution is OPTIONAL. Existing PlanExe functionality must not break.

- Plans can be generated WITHOUT execution (existing behavior)
- Execution endpoints are new, don't conflict with existing `/api/plans/start`
- Database migrations are additive only
- No changes to existing models (only additions)

---

## Next Steps

1. **Confirm database schema** with team (any existing task models to reuse?)
2. **Choose WebSocket approach** (Socket.IO vs pure WebSocket vs Server-Sent Events)
3. **Prototype Phase 1** with mock decomposition (no LLM yet)
4. **Test Larry agent** with real prospecting scenario
5. **Gather feedback** before full dashboard implementation

---

This integration plan should align with Mark's existing PlanExe codebase. Questions about specific integration points?
