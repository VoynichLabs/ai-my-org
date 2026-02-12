# Agent Execution System - Implementation Guide
**Author:** openai-codex/gpt-5.1-codex-mini  
**Model:** openai-codex/gpt-5.1-codex-mini  
**Generated on:** 11 February 2026  


## 1. Data Models & Database Schema

### Core Entities

#### PlanTask (extends PlanExe's existing plan)
```python
class PlanTask(Base):
    """Structured task within a plan that agents can execute"""
    __tablename__ = "plan_tasks"
    
    id: str = Column(String, primary_key=True)  # "task_1_1"
    plan_id: str = Column(String, ForeignKey("plans.id"), nullable=False)
    phase_id: str = Column(String)  # "phase_1"
    
    title: str = Column(String, nullable=False)
    description: str = Column(Text, nullable=False)
    assignee_role: str = Column(String)  # "Business Analyst", "Developer"
    
    estimated_hours: int = Column(Integer)
    priority: int = Column(Integer, default=1)  # 1=highest
    
    # Decomposition
    decomposed: bool = Column(Boolean, default=False)
    decomposition_confidence: float = Column(Float)  # 0-1
    decomposition_notes: str = Column(Text)
    
    # Sequencing
    prerequisite_task_ids: str = Column(JSON, default=list)  # ["task_1_0"]
    is_blocking: bool = Column(Boolean, default=False)  # Blocks downstream tasks?
    
    # Metadata
    created_at: datetime = Column(DateTime, default=datetime.utcnow)
    updated_at: datetime = Column(DateTime, onupdate=datetime.utcnow)
    
    # Relationships
    subtasks = relationship("PlanSubtask", back_populates="task")
    execution_tasks = relationship("ExecutionTask", back_populates="plan_task")
```

#### PlanSubtask (Atomic executable unit)
```python
class PlanSubtask(Base):
    """Atomic action within a task"""
    __tablename__ = "plan_subtasks"
    
    id: str = Column(String, primary_key=True)  # "task_1_1_a"
    task_id: str = Column(String, ForeignKey("plan_tasks.id"), nullable=False)
    
    title: str = Column(String, nullable=False)
    action: str = Column(String)  # "send_email", "make_phone_call", "search_google_maps"
    params: dict = Column(JSON)  # Action-specific parameters
    
    # Execution hints
    timeout_hours: int = Column(Integer, default=24)
    retry_policy: dict = Column(JSON)  # {"max_attempts": 3, "backoff": "exponential"}
    
    # Sequencing
    prerequisite_subtask_ids: str = Column(JSON, default=list)
    
    # Metadata
    task = relationship("PlanTask", back_populates="subtasks")
    created_at: datetime = Column(DateTime, default=datetime.utcnow)
```

#### Agent (Registry of available agents)
```python
class Agent(Base):
    """Registered agent in the system"""
    __tablename__ = "agents"
    
    id: str = Column(String, primary_key=True)  # "agent_larry_001"
    name: str = Column(String, nullable=False)
    role: str = Column(String)  # "Full-stack contractor", "Project Manager"
    
    # Connectivity
    poll_endpoint: str = Column(String)  # Where agent polls for work
    webhook_url: str = Column(String)  # Alternative: push updates to agent
    is_active: bool = Column(Boolean, default=True)
    last_heartbeat: datetime = Column(DateTime)
    
    # Capabilities (JSON list)
    capabilities: list = Column(JSON)  # [{"action": "send_email", "params": [...]}, ...]
    
    # Metadata
    created_at: datetime = Column(DateTime, default=datetime.utcnow)
    
    # Relationships
    executions = relationship("Execution", back_populates="agent")
    events = relationship("ExecutionEvent", back_populates="agent")
```

#### Execution (Instance of plan being executed)
```python
class Execution(Base):
    """Single execution of a plan by an agent"""
    __tablename__ = "executions"
    
    id: str = Column(String, primary_key=True)  # "exec_larry_prospect_20260211_001"
    plan_id: str = Column(String, ForeignKey("plans.id"), nullable=False)
    agent_id: str = Column(String, ForeignKey("agents.id"), nullable=False)
    
    # Status
    status: str = Column(String, default="CREATED")  # CREATED|QUEUED|RUNNING|BLOCKED|PAUSED|COMPLETED|FAILED|CANCELLED
    status_message: str = Column(String)
    
    # Progress
    tasks_total: int = Column(Integer)
    tasks_completed: int = Column(Integer, default=0)
    current_task_id: str = Column(String)  # Which task is executing now
    
    # Control signals
    pause_requested: bool = Column(Boolean, default=False)
    cancel_requested: bool = Column(Boolean, default=False)
    
    # Metadata
    created_at: datetime = Column(DateTime, default=datetime.utcnow)
    started_at: datetime = Column(DateTime)
    completed_at: datetime = Column(DateTime)
    updated_at: datetime = Column(DateTime, onupdate=datetime.utcnow)
    
    # Relationships
    agent = relationship("Agent", back_populates="executions")
    plan = relationship("Plan", back_populates="executions")
    tasks = relationship("ExecutionTask", back_populates="execution")
    events = relationship("ExecutionEvent", back_populates="execution")
```

#### ExecutionTask (Task within an execution)
```python
class ExecutionTask(Base):
    """Execution state for a single task"""
    __tablename__ = "execution_tasks"
    
    id: str = Column(String, primary_key=True)
    execution_id: str = Column(String, ForeignKey("executions.id"), nullable=False)
    plan_task_id: str = Column(String, ForeignKey("plan_tasks.id"), nullable=False)
    
    # Status
    status: str = Column(String, default="QUEUED")  # QUEUED|RUNNING|BLOCKED|COMPLETE|ERROR
    
    # Progress
    subtasks_total: int = Column(Integer)
    subtasks_completed: int = Column(Integer, default=0)
    progress_percent: int = Column(Integer, default=0)  # 0-100
    
    # Results
    result_data: dict = Column(JSON)  # Task-specific output
    error_message: str = Column(String)
    
    # Timing
    started_at: datetime = Column(DateTime)
    completed_at: datetime = Column(DateTime)
    
    # Relationships
    execution = relationship("Execution", back_populates="tasks")
    plan_task = relationship("PlanTask", back_populates="execution_tasks")
    subtask_executions = relationship("ExecutionSubtask", back_populates="execution_task")
```

#### ExecutionEvent (Event log for execution)
```python
class ExecutionEvent(Base):
    """Immutable event log for debugging and auditing"""
    __tablename__ = "execution_events"
    
    id: str = Column(String, primary_key=True)
    execution_id: str = Column(String, ForeignKey("executions.id"), nullable=False)
    agent_id: str = Column(String, ForeignKey("agents.id"), nullable=False)
    
    # Event classification
    event_type: str = Column(String)  # heartbeat|task_started|task_complete|task_error|task_blocked|subtask_complete
    task_id: str = Column(String)  # May be null for heartbeat
    subtask_id: str = Column(String)  # May be null for task events
    
    # Event data
    status: str = Column(String)  # running|success|error|blocked|paused
    message: str = Column(String)
    output_data: dict = Column(JSON)  # Event-specific data
    
    # Error details (if applicable)
    error_type: str = Column(String)  # external_dependency_blocked|timeout|validation_error
    error_message: str = Column(String)
    human_review_required: bool = Column(Boolean, default=False)
    
    # Metadata
    timestamp: datetime = Column(DateTime, default=datetime.utcnow, nullable=False)
    agent_timestamp: datetime = Column(DateTime)  # When agent reported the event
    
    # Relationships
    execution = relationship("Execution", back_populates="events")
    agent = relationship("Agent", back_populates="events")
```

#### ExecutionSubtask (Atomic execution state)
```python
class ExecutionSubtask(Base):
    """Execution state for a single subtask"""
    __tablename__ = "execution_subtasks"
    
    id: str = Column(String, primary_key=True)
    execution_task_id: str = Column(String, ForeignKey("execution_tasks.id"), nullable=False)
    plan_subtask_id: str = Column(String, ForeignKey("plan_subtasks.id"), nullable=False)
    
    # Execution state
    status: str = Column(String, default="QUEUED")  # QUEUED|RUNNING|COMPLETE|ERROR|SKIPPED
    attempt: int = Column(Integer, default=1)
    max_attempts: int = Column(Integer, default=3)
    
    # Results
    result_data: dict = Column(JSON)  # Execution output
    error: str = Column(String)
    
    # Timing
    started_at: datetime = Column(DateTime)
    completed_at: datetime = Column(DateTime)
    
    # Relationships
    execution_task = relationship("ExecutionTask", back_populates="subtask_executions")
```

---

## 2. API Contracts (OpenAPI Spec)

### Agent Polling Endpoint

**GET `/api/agents/{agent_id}/work`**

Polling endpoint for agents to request work.

Request:
```
GET /api/agents/agent_larry_001/work?supported_actions=send_email,make_phone_call,search_google_maps
```

Response (200 OK):
```json
{
  "work_available": true,
  "execution_id": "exec_carol_prospect_20260211_001",
  "plan_id": "plan_carol_breeder_sys_20260211",
  "next_task": {
    "task_id": "task_1_1_a",
    "title": "Send initial interview request to Carol",
    "action": "send_email",
    "params": {
      "to": "carol@bittersweet-farm.com",
      "subject": "Discovery Interview Request",
      "body": "Hi Carol...",
      "cc": [],
      "bcc": []
    },
    "timeout_hours": 24,
    "retry_policy": {
      "max_attempts": 3,
      "backoff_seconds": 300
    },
    "context": {
      "plan_goal": "Build custom breeder management system",
      "previous_tasks": ["Called Carol, high interest"],
      "upcoming_tasks": ["Wait for response", "Document requirements"]
    }
  },
  "force_pause": false,
  "force_cancel": false
}
```

Response (204 No Content) - No work available:
```
Agent is up-to-date; nothing to do
```

---

### Event Reporting Endpoint

**POST `/api/executions/{execution_id}/events`**

Agent reports progress by submitting events.

Request:
```json
{
  "agent_id": "agent_larry_001",
  "agent_timestamp": "2026-02-11T16:09:45Z",
  "event_type": "task_complete",
  "task_id": "task_1_1_a",
  "status": "success",
  "output_data": {
    "email_sent_to": "carol@bittersweet-farm.com",
    "email_timestamp": "2026-02-11T16:09:45Z",
    "recipient_response_received": true,
    "response_excerpt": "Sounds good, let's schedule for Thursday at 2 PM",
    "response_timestamp": "2026-02-11T16:45:00Z",
    "mailbox_used": "82deutschmark@gmail.com"
  },
  "message": "Email sent and response received. Ready for next step."
}
```

Response (201 Created):
```json
{
  "event_id": "evt_123456",
  "execution_id": "exec_carol_prospect_20260211_001",
  "recorded_at": "2026-02-11T16:09:50Z",
  "next_task_id": "task_1_1_b",
  "next_action": "document_requirements",
  "status_message": "Task complete. Proceeding to Task 1.1.2"
}
```

---

### Error Reporting

**POST `/api/executions/{execution_id}/events`**

Error event with human review request.

Request:
```json
{
  "agent_id": "agent_larry_001",
  "agent_timestamp": "2026-02-11T17:00:00Z",
  "event_type": "task_error",
  "task_id": "task_1_1_a",
  "status": "error",
  "error_type": "external_dependency_blocked",
  "error_message": "Carol's email address (carol@bittersweet-farm.com) bounced. Invalid or deactivated.",
  "message": "Unable to send email. Contact verification needed.",
  "human_review_required": true,
  "suggestions": [
    "Try alternative email from website footer (info@bittersweetfarm.com)",
    "Call Carol directly at 860-455-9416",
    "Find email via LinkedIn or Facebook"
  ],
  "blocking": true
}
```

Response (201 Created):
```json
{
  "event_id": "evt_123457",
  "execution_id": "exec_carol_prospect_20260211_001",
  "recorded_at": "2026-02-11T17:00:05Z",
  "status": "BLOCKED",
  "human_review_required": true,
  "escalated_to": "user_mark_12345",
  "next_step_options": [
    "approve_alternative_email",
    "approve_phone_call",
    "skip_task",
    "cancel_execution"
  ]
}
```

---

## 3. Agent Decomposition Request (for smart agents)

**POST `/api/agents/{agent_id}/decompose`**

For agents with LLM capability, request task decomposition.

Request:
```json
{
  "task": {
    "task_id": "task_1_1",
    "title": "Interview Carol about current workflow",
    "description": "Understand her breeding records, customer management, puppy tracking workflows",
    "assignee_role": "Business Analyst"
  },
  "plan_context": {
    "goal": "Build custom dog breeder management system",
    "business_type": "Dog breeding",
    "industry": "Animal husbandry",
    "target_customer": "Carol Campion (Bittersweet Farm, border collie breeder, 25+ years experience)"
  },
  "agent_capabilities": [
    "send_email",
    "make_phone_call",
    "schedule_calendar_event",
    "conduct_interview",
    "document_findings",
    "analyze_feedback"
  ]
}
```

Response (200 OK):
```json
{
  "task_id": "task_1_1",
  "decomposition_confidence": 0.92,
  "decomposed_subtasks": [
    {
      "subtask_id": "task_1_1_a",
      "title": "Send initial contact email",
      "action": "send_email",
      "params": {
        "to": "carol@bittersweet-farm.com",
        "subject": "Breeder Management System - Discovery Interview",
        "body": "Hi Carol,\n\nI'm Larry, a software contractor building custom management systems for small animal operations...",
        "tone": "professional_friendly"
      },
      "depends_on": [],
      "success_criteria": [
        "Email sent successfully",
        "Response received within 24 hours"
      ],
      "failure_handling": "If bounce: try phone call"
    },
    {
      "subtask_id": "task_1_1_b",
      "title": "Conduct 90-minute discovery interview",
      "action": "conduct_interview",
      "params": {
        "duration_minutes": 90,
        "format": "video_call_or_phone",
        "script_template": "breeder_discovery_interview.md",
        "required_topics": [
          "Current breeding records system",
          "Customer management process",
          "Puppy tracking workflow",
          "Pain points with current tools",
          "Desired features"
        ]
      },
      "depends_on": ["task_1_1_a"],
      "success_criteria": [
        "Interview conducted",
        "All 5 topics covered",
        "Transcript or notes recorded"
      ]
    },
    {
      "subtask_id": "task_1_1_c",
      "title": "Document findings in structured format",
      "action": "document_findings",
      "params": {
        "template": "interview_summary_template.md",
        "output_path": "interviews/carol_20260211_discovery.md",
        "sections": [
          "Current workflow overview",
          "Key pain points",
          "Feature wishlist",
          "Estimated priorities"
        ]
      },
      "depends_on": ["task_1_1_b"],
      "success_criteria": [
        "Document created",
        "All sections completed",
        "Ready for proposal generation"
      ]
    }
  ],
  "execution_sequence": ["task_1_1_a", "task_1_1_b", "task_1_1_c"],
  "notes": "Task should be completed within 1 week. Carol has high interest based on call feedback.",
  "warnings": []
}
```

---

## 4. Dashboard WebSocket (Real-time Updates)

**WS `/api/executions/{execution_id}/stream`**

Opens a persistent WebSocket connection for real-time dashboard updates.

Connection: Agent subscribes with authentication token
```
WS /api/executions/exec_carol_prospect_20260211_001/stream?token=user_token_xyz
```

Messages pushed from server:
```json
{
  "type": "execution_update",
  "execution_id": "exec_carol_prospect_20260211_001",
  "status": "RUNNING",
  "progress": {
    "tasks_completed": 3,
    "tasks_total": 12,
    "percent": 25,
    "current_task_id": "task_1_1_c",
    "current_task_title": "Document findings"
  },
  "timestamp": "2026-02-11T16:10:00Z"
}
```

Or on event:
```json
{
  "type": "event",
  "event_id": "evt_123456",
  "execution_id": "exec_carol_prospect_20260211_001",
  "event_type": "task_complete",
  "task_id": "task_1_1_a",
  "message": "Email sent and response received. Ready for next step.",
  "timestamp": "2026-02-11T16:09:50Z"
}
```

---

## 5. Execution State Transitions

```python
# Valid state transitions
VALID_TRANSITIONS = {
    "CREATED": ["QUEUED", "CANCELLED"],
    "QUEUED": ["RUNNING", "CANCELLED"],
    "RUNNING": ["BLOCKED", "PAUSED", "COMPLETED", "FAILED", "CANCELLED"],
    "BLOCKED": ["RUNNING", "PAUSED", "CANCELLED"],  # After blocker resolved
    "PAUSED": ["RUNNING", "CANCELLED"],
    "COMPLETED": [],  # Terminal
    "FAILED": ["CANCELLED"],  # Or retry (new execution)
    "CANCELLED": [],  # Terminal
}

# On event received, validate and update state
def process_event(execution_id: str, event: ExecutionEvent):
    execution = db.session.get(Execution, execution_id)
    
    # Determine new state from event type
    new_state = derive_state(execution.status, event)
    
    # Validate transition
    if new_state not in VALID_TRANSITIONS.get(execution.status, []):
        raise InvalidStateTransition(f"{execution.status} → {new_state}")
    
    # Update execution
    execution.status = new_state
    execution.updated_at = datetime.utcnow()
    
    # Side effects
    if event.event_type == "task_complete":
        execution.tasks_completed += 1
    elif event.event_type == "task_error" and event.human_review_required:
        notify_user(f"Execution {execution_id} blocked: {event.error_message}")
```

---

## 6. Implementation Milestones

### Milestone 1: Core Data Models (Week 1)
- [ ] Database schema (SQLAlchemy models)
- [ ] Migrations (Alembic)
- [ ] Test fixtures

### Milestone 2: Agent Polling Loop (Week 2)
- [ ] `GET /api/agents/{agent_id}/work` endpoint
- [ ] `POST /api/executions/{execution_id}/events` endpoint
- [ ] Execution state machine
- [ ] Task decomposition placeholder (static examples)

### Milestone 3: Larry Agent Integration (Week 3)
- [ ] Larry agent registration
- [ ] Decomposition logic for prospecting tasks
- [ ] Email, phone call, calendar actions
- [ ] CSV management for prospects
- [ ] Test with real prospecting scenario

### Milestone 4: Dashboard (Week 4)
- [ ] Execution list view
- [ ] Real-time progress tracking
- [ ] Task breakdown tree
- [ ] Event log viewer
- [ ] WebSocket stream implementation

---

## 7. Testing Strategy

### Unit Tests
- State transitions (valid/invalid)
- Event processing logic
- Decomposition confidence scoring
- CSV parsing

### Integration Tests
- Full polling loop (agent requests work → processes → reports)
- Event ordering and deduplication
- Dashboard real-time updates
- Error recovery (retry policies)

### End-to-End Tests
- Larry prospecting scenario (all 3 phases)
- Multiple agents executing in parallel
- Human pause/cancel during execution
- Webhook notifications

---

## 8. Performance & Scalability

| Component | Target | Approach |
|-----------|--------|----------|
| Event ingestion | 1000 events/sec | Batch writes, queue |
| Dashboard updates | <100ms latency | WebSocket push |
| Agent polling | 100 agents × 10min interval | Connection pooling |
| Event storage | 1GB/month (est.) | Archival after 90 days |

---

## 9. Security Considerations

### Agent Authentication
- API key + signature verification for webhook requests
- JWT tokens for WebSocket connections
- Rate limiting: 100 reqs/min per agent

### Data Privacy
- Execution logs don't store sensitive data (emails, phone numbers masked)
- Event data encrypted at rest
- User-scoped queries (can only see own executions)

### Error Handling
- Never expose internal error details to agents
- Log stack traces separately for debugging
- Return generic error messages to clients

---

## 10. Logging & Debugging

### Agent-Side Logging
```python
import logging

logger = logging.getLogger("agent_larry")
handler = logging.FileHandler("logs/larry_execution.log")
formatter = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")
handler.setFormatter(formatter)
logger.addHandler(handler)

# On event
logger.info(f"Reporting event: {event_type} for task {task_id}")
```

### Server-Side Logging
```python
logger.info(f"Execution {execution_id} transitioned: {old_state} → {new_state}")
logger.error(f"Task {task_id} failed: {error_message}", exc_info=True)
```

### Debugging Commands
```bash
# View execution state
curl http://localhost:8000/api/executions/exec_carol_prospect_20260211_001

# View recent events
curl http://localhost:8000/api/executions/exec_carol_prospect_20260211_001/events?limit=10

# Manually advance execution
curl -X PATCH http://localhost:8000/api/executions/exec_carol_prospect_20260211_001 \
  -d '{"status": "PAUSED"}'
```

---

This implementation guide should be enough to start development. Questions: What should we prioritize first?
