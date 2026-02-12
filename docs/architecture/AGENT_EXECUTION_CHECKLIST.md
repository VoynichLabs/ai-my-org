# Agent Execution System - Implementation Checklist
**Author:** openai-codex/gpt-5.1-codex-mini  
**Model:** openai-codex/gpt-5.1-codex-mini  
**Generated on:** 11 February 2026  


Use this checklist to track implementation progress. Each section can be completed by a sub-agent or developer.

---

## PHASE 1: CORE INFRASTRUCTURE (Week 1-2)

### Database & Models

- [ ] Create `database_api/model_execution.py`
  - [ ] `Agent` model (id, name, capabilities, last_heartbeat)
  - [ ] `Execution` model (id, plan_id, agent_id, status, progress)
  - [ ] `ExecutionTask` model (id, execution_id, task_id, status, result)
  - [ ] `ExecutionEvent` model (id, execution_id, event_type, timestamp, output)
  - [ ] `ExecutionSubtask` model (id, execution_task_id, status, result)
  - [ ] Relationships and foreign keys correct

- [ ] Create Alembic migration
  - [ ] `alembic revision --autogenerate -m "Add execution models"`
  - [ ] Verify migration is valid (can upgrade/downgrade)
  - [ ] Test on clean database

- [ ] Update `database_api/__init__.py` exports
  - [ ] Import all new models
  - [ ] Add to `__all__`

### FastAPI Routes

- [ ] Create `worker_plan/worker_plan_execution/` module
  - [ ] `__init__.py` (package init)
  - [ ] `routes.py` (all HTTP endpoints)
  - [ ] `service.py` (business logic)
  - [ ] `state_machine.py` (state transitions)

- [ ] Implement basic routes in `routes.py`
  - [ ] `POST /api/executions` (create execution)
    - [ ] Accept plan_id, agent_id
    - [ ] Create Execution record (status=CREATED)
    - [ ] Return execution_id
    - [ ] Input validation
  - [ ] `GET /api/executions/{execution_id}` (get status)
    - [ ] Return status, progress, current_task
    - [ ] 404 if not found
  - [ ] `PATCH /api/executions/{execution_id}` (pause/resume/cancel)
    - [ ] Accept new status (PAUSED, RUNNING, CANCELLED)
    - [ ] Validate transition
    - [ ] Return new execution state
  - [ ] `GET /api/agents/{agent_id}/work` (agent polling)
    - [ ] Return next task for agent
    - [ ] 204 if no work
    - [ ] Include task params, timeout, retry policy
  - [ ] `POST /api/executions/{execution_id}/events` (event reporting)
    - [ ] Accept event data (task_id, status, output)
    - [ ] Store in ExecutionEvent table
    - [ ] Update ExecutionTask status
    - [ ] Return next task or completion
  - [ ] `GET /api/executions/{execution_id}/events` (event history)
    - [ ] Return paginated event list
    - [ ] Order by timestamp DESC
    - [ ] Support limit & offset

- [ ] Register routes in `worker_plan/app.py`
  - [ ] Import router from `worker_plan_execution.routes`
  - [ ] `app.include_router(router, prefix="/api", tags=["executions"])`
  - [ ] Test that routes are accessible

### State Machine

- [ ] Implement in `state_machine.py`
  - [ ] Valid transitions map
  - [ ] `derive_state()` function (event → new state)
  - [ ] `validate_transition()` function
  - [ ] State change logging

- [ ] Test state transitions
  - [ ] CREATED → QUEUED ✓
  - [ ] QUEUED → RUNNING ✓
  - [ ] RUNNING → BLOCKED ✓
  - [ ] BLOCKED → RUNNING ✓
  - [ ] RUNNING → COMPLETED ✓
  - [ ] Invalid: COMPLETED → RUNNING (should fail) ✓

### Configuration

- [ ] Update `.env.developer-example`
  - [ ] `PLANEXE_EXECUTION_ENABLED=true`
  - [ ] `PLANEXE_AGENT_POLL_INTERVAL_SECONDS=300`

- [ ] Update `planexe_dotenv.py`
  - [ ] Add env var parsing for execution settings

### Testing

- [ ] Create `worker_plan/tests/test_execution_models.py`
  - [ ] Test model creation
  - [ ] Test relationships

- [ ] Create `worker_plan/tests/test_state_machine.py`
  - [ ] Test valid transitions
  - [ ] Test invalid transitions (should raise error)
  - [ ] Test edge cases

- [ ] Create `worker_plan/tests/test_execution_routes.py`
  - [ ] Test POST /api/executions (create)
  - [ ] Test GET /api/executions/{id} (read)
  - [ ] Test PATCH /api/executions/{id} (update)
  - [ ] Test with mock database

- [ ] Run tests
  - [ ] `pytest worker_plan/tests/` passes
  - [ ] No database errors
  - [ ] Coverage report generated

### Documentation

- [ ] Create `docs/agent_execution_api.md`
  - [ ] Document all endpoints
  - [ ] Include request/response examples
  - [ ] Error codes and meanings

---

## PHASE 2: DECOMPOSITION & AGENT INTEGRATION (Week 3)

### Task Decomposition

- [ ] Create `worker_plan/worker_plan_execution/decomposition.py`
  - [ ] `decompose_plan()` function
    - [ ] Input: Plan JSON from PlanExe
    - [ ] Output: List of PlanTask + PlanSubtask records
  - [ ] `parse_tasks_from_plan()` (parse markdown/JSON)
  - [ ] `compute_confidence()` (task decomposition confidence 0-1)
  - [ ] Error handling if plan structure unexpected

- [ ] Test decomposition
  - [ ] Parse real plan output from PlanExe
  - [ ] Verify task structure is correct
  - [ ] Confidence scores reasonable (0.7+ for Larry tasks)

### Agent Registration

- [ ] Create endpoint: `POST /api/agents` (register new agent)
  - [ ] Accept: name, role, capabilities list
  - [ ] Create Agent record
  - [ ] Return agent_id
  - [ ] Validation: at least 1 capability

- [ ] Test agent registration
  - [ ] Register Larry agent with capabilities
    - [ ] send_email
    - [ ] make_phone_call
    - [ ] search_google_maps
    - [ ] schedule_calendar_event
    - [ ] conduct_interview
    - [ ] document_findings
    - [ ] draft_document
    - [ ] manage_csv

- [ ] Create endpoint: `GET /api/agents/{agent_id}` (get agent info)
  - [ ] Return capabilities, status, last_heartbeat

### Execution Service Logic

- [ ] Create `ExecutionService` class in `service.py`
  - [ ] `create_execution(plan_id, agent_id)` → execution_id
  - [ ] `get_work_for_agent(agent_id)` → next task
  - [ ] `report_event(execution_id, event)` → next_task
  - [ ] `update_execution_status(execution_id, new_status)`

- [ ] Test service
  - [ ] Create execution from plan
  - [ ] Agent polls for work
  - [ ] Agent executes task 1
  - [ ] Agent reports completion
  - [ ] Next task available
  - [ ] All tasks executed in sequence

### Polling Loop (Agent Simulation)

- [ ] Create `scripts/test_agent_loop.py` (simulate agent polling)
  - [ ] Loop: poll for work → simulate action → report event
  - [ ] Run for full prospecting scenario
  - [ ] Log all events to file
  - [ ] Verify execution completes successfully

- [ ] Test full loop
  - [ ] Create prospecting plan execution
  - [ ] Run agent simulator
  - [ ] Verify: Task 1.1 → 1.2 → 1.3 → 2.1 → ... → Complete ✓
  - [ ] Check event log for all tasks ✓

### Documentation

- [ ] Create `docs/agent_setup.md`
  - [ ] How to register an agent
  - [ ] Polling loop pattern (pseudo-code)
  - [ ] Event reporting format
  - [ ] Example: Larry agent registration

---

## PHASE 3: DASHBOARD (Week 4)

### API Enhancements

- [ ] Add WebSocket route: `WS /api/executions/{execution_id}/stream`
  - [ ] Accept WebSocket connection
  - [ ] Send execution status updates every 5 sec (or on event)
  - [ ] Format: JSON with timestamp, status, progress
  - [ ] Handle disconnection gracefully

- [ ] Add filtering to event endpoint
  - [ ] `GET /api/executions/{execution_id}/events?type=task_complete`
  - [ ] `GET /api/executions/{execution_id}/events?task_id=task_1_1`

- [ ] Add execution list
  - [ ] `GET /api/executions?plan_id={plan_id}&status=RUNNING`
  - [ ] Return paginated list with summaries

### Frontend Component (if using Gradio)

- [ ] Create execution monitor
  - [ ] Input: execution_id
  - [ ] Display: current status, progress bar
  - [ ] Display: task tree (clickable to expand)
  - [ ] Display: recent events (last 10)
  - [ ] Auto-refresh every 2 seconds
  - [ ] Buttons: Pause, Resume, Cancel

### Frontend Component (if using Flask)

- [ ] Create `templates/execution_dashboard.html`
  - [ ] Real-time progress bar (updated via WebSocket)
  - [ ] Task tree with status icons
  - [ ] Event log (scrollable)
  - [ ] Risk/blocker panel
  - [ ] Control buttons (pause/resume/cancel)

- [ ] Create `static/execution.js`
  - [ ] WebSocket connection to `/api/executions/{id}/stream`
  - [ ] Parse incoming messages
  - [ ] Update DOM elements in real-time
  - [ ] Reconnect on disconnect

- [ ] Create CSS styling
  - [ ] Progress bar (green for complete, blue for running, orange for blocked)
  - [ ] Task tree indentation and icons
  - [ ] Responsive layout

### Testing

- [ ] Create `worker_plan/tests/test_websocket.py`
  - [ ] Test WebSocket connection
  - [ ] Test message format
  - [ ] Test disconnection handling

- [ ] Manual testing
  - [ ] Open dashboard for execution
  - [ ] Start agent simulator
  - [ ] Verify dashboard updates in real-time ✓
  - [ ] Verify task tree updates as tasks complete ✓
  - [ ] Verify event log appends new events ✓

---

## PHASE 4: HARDENING & PRODUCTION (Week 5+)

### Error Handling

- [ ] Handle agent timeout (no poll for 30+ min)
  - [ ] Mark execution as BLOCKED
  - [ ] Notify user
  - [ ] Allow reassignment to different agent

- [ ] Handle task timeout (subtask exceeds timeout_hours)
  - [ ] Trigger retry policy (exponential backoff, max 3 attempts)
  - [ ] If all retries fail: escalate to user

- [ ] Handle decomposition failure
  - [ ] If confidence < 0.7: require human review
  - [ ] Allow user to override with manual decomposition

- [ ] Handle database errors
  - [ ] Connection pool exhaustion
  - [ ] Transaction conflicts
  - [ ] Recovery with exponential backoff

- [ ] Test error scenarios
  - [ ] Agent goes offline mid-execution
  - [ ] Database connection lost
  - [ ] Invalid event data from agent
  - [ ] Task timeout triggers retry

### Monitoring & Logging

- [ ] Add structured logging to all routes
  - [ ] Request received, response sent
  - [ ] Event processing
  - [ ] State transitions
  - [ ] Errors with full stack trace

- [ ] Add metrics
  - [ ] Execution success rate
  - [ ] Average execution duration
  - [ ] Average task duration
  - [ ] Error rate by type

- [ ] Create admin endpoint: `GET /api/admin/metrics`
  - [ ] Return execution stats
  - [ ] Return agent stats
  - [ ] Return error summary

### Performance

- [ ] Load test with 100 concurrent executions
  - [ ] Use Apache JMeter or similar
  - [ ] Measure latency, throughput
  - [ ] Identify bottlenecks

- [ ] Optimize queries
  - [ ] Add indexes on frequently filtered columns
  - [ ] Check N+1 query problems
  - [ ] Verify connection pool size

- [ ] Test database scaling
  - [ ] 1M events: can still query efficiently?
  - [ ] Implement archival (move old events to archive table)

### Documentation

- [ ] Write deployment guide
  - [ ] Environment variables to set
  - [ ] Database migration steps
  - [ ] Docker setup
  - [ ] Troubleshooting

- [ ] Write operations guide
  - [ ] How to restart stuck execution
  - [ ] How to manually intervene
  - [ ] How to view logs

- [ ] Write user guide
  - [ ] How to create execution
  - [ ] How to monitor dashboard
  - [ ] How to pause/resume
  - [ ] How to handle blockers

### Backward Compatibility

- [ ] Verify existing PlanExe functionality still works
  - [ ] Plan generation (no changes)
  - [ ] Plan output files still generated
  - [ ] No breaking changes to existing APIs

- [ ] Test with existing frontend
  - [ ] Can still generate plans?
  - [ ] No errors in plan generation UI?

---

## ONGOING MAINTENANCE

### Code Review Checklist

Before merging any PR:

- [ ] All tests pass (`pytest worker_plan/tests/`)
- [ ] Code follows existing style
- [ ] No hardcoded secrets or API keys
- [ ] Database migrations are reversible
- [ ] Backward compatible (no breaking changes)
- [ ] Documentation updated
- [ ] Logging added for debugging

### Testing

- [ ] Run full suite weekly: `pytest`
- [ ] Run integration tests on staging
- [ ] Load test monthly (100+ concurrent executions)

### Monitoring

- [ ] Check metrics daily
  - [ ] Execution success rate > 95%?
  - [ ] Error rate < 1%?
  - [ ] Average latency < 100ms?

- [ ] Review error logs daily
  - [ ] Any new error patterns?
  - [ ] Any agent failures?

---

## Sign-Off

When complete, mark sections as DONE:

- [ ] **PHASE 1: CORE INFRASTRUCTURE** — DONE
- [ ] **PHASE 2: DECOMPOSITION & AGENTS** — DONE
- [ ] **PHASE 3: DASHBOARD** — DONE
- [ ] **PHASE 4: HARDENING** — DONE
- [ ] **PHASE 5: PRODUCTION READY** — DONE

---

## Quick Commands (for development)

```bash
# Run tests
pytest worker_plan/tests/ -v

# Run single test
pytest worker_plan/tests/test_state_machine.py::test_valid_transition -v

# Start development server
cd worker_plan
python -m worker_plan.app

# Run database migration
alembic upgrade head

# Rollback migration
alembic downgrade -1

# View recent logs
tail -f logs/execution.log

# Check agent status
curl http://localhost:8000/api/agents/agent_larry_001

# Manually trigger work polling
curl http://localhost:8000/api/agents/agent_larry_001/work

# Report event (test)
curl -X POST http://localhost:8000/api/executions/exec_test_001/events \
  -H "Content-Type: application/json" \
  -d '{"agent_id":"agent_larry_001","event_type":"task_complete","task_id":"task_1_1","status":"success"}'
```

---

## Questions During Implementation?

1. **Decomposition uncertainty:** Is LLM-based decomposition okay, or rule-based templates?
2. **Agent communication:** Polling good enough, or add webhook support now?
3. **Dashboard:** Real-time WebSocket, or simpler polling approach?
4. **Multi-agent:** Support parallel task execution from day 1, or Phase 2?
5. **Timeline:** Can start Monday, or wait for planning discussion?

---

**Document last updated:** 2026-02-11
**Status:** Ready for implementation
**Estimated effort:** 3-4 weeks (1-2 engineers)
