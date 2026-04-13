# Logical Entity–Relationship Model

This document defines the **Logical Entity-Relationship model** for the Distributed LLM Agents system.

At this level we:

- Define **entities, attributes, primary keys (PK), and foreign keys (FK)**.
- Keep the model **technology-agnostic** (no vendor-specific types).
- Make relationships **explicit** (1–N, N–1, N-M).
- Build directly on the conceptual model (`conceptual-erd.md`) and prepare for the Physical ERD.

Core entities:

- `Agent`
- `Task`
- `SubTask`
- `Execution`
- `AgentMessage`
- `EvaluationResult`
- `ConsensusRecord`

> Note: Roles (Planner, Executor, Critic, Synthesizer) are treated as *roles of `Agent`* at this stage and are not modeled as separate entities.

---

## 1. Entities and Attributes

### 1.1 Agent

Represents a registered LLM-backed agent instance in the system.

**Primary key**

- `agent_id` – unique identifier for the agent.

**Attributes**

| Attribute        | Type     | Description                                                                  |
|-----------------|----------|------------------------------------------------------------------------------|
| `agent_id`      | string   | Unique agent identifier (PK).                                                |
| `agent_role`    | string   | Role of the agent: "planner", "executor", "critic", "synthesizer".           |
| `llm_model`     | string   | Underlying LLM model identifier (e.g. "gpt-4o", "claude-3-5-sonnet").       |
| `llm_provider`  | string   | LLM provider name (e.g. "openai", "anthropic", "local").                     |
| `capabilities`  | json     | List of capability tags this agent can handle (e.g. ["code", "reasoning"]). |
| `status`        | string   | Current agent status: "idle", "busy", "quarantined", "offline".              |
| `registered_at` | datetime | When this agent was registered with the system.                              |

---

### 1.2 Task

Represents a high-level problem submitted by an external client.

**Primary key**

- `task_id` – surrogate identifier for the task.

**Attributes**

| Attribute           | Type     | Description                                                              |
|--------------------|----------|--------------------------------------------------------------------------|
| `task_id`          | string   | Unique task identifier (PK).                                             |
| `title`            | string   | Short human-readable name for the task.                                  |
| `description`      | text     | Full natural-language description of the task.                           |
| `complexity_level` | string   | Estimated complexity: "simple", "moderate", "complex".                   |
| `status`           | string   | Lifecycle state: "pending", "decomposing", "in_progress", "completed", "failed". |
| `submitted_by`     | string   | Identifier of the external user or system that submitted the task.       |
| `created_at`       | datetime | When the task was submitted.                                             |
| `completed_at`     | datetime | When the task reached a terminal state (nullable).                       |

---

### 1.3 SubTask

Represents a decomposed unit of work derived from a Task.

**Primary key**

- `subtask_id` – surrogate identifier for the sub-task.

**Foreign keys**

- `task_id` → `Task.task_id`.

**Attributes**

| Attribute         | Type     | Description                                                                     |
|------------------|----------|---------------------------------------------------------------------------------|
| `subtask_id`     | string   | Unique sub-task identifier (PK).                                                |
| `task_id`        | string   | FK to `Task` (the parent task this sub-task belongs to).                        |
| `description`    | text     | Natural-language description of the sub-task's scope and expected output.       |
| `required_role`  | string   | The agent role required to process this sub-task (e.g. "executor").             |
| `dependencies`   | json     | Ordered list of `subtask_id` values this sub-task depends on.                   |
| `sequence_order` | integer  | Position of this sub-task within the DAG's topological ordering.                |
| `status`         | string   | State: "pending", "assigned", "in_progress", "completed", "failed", "retrying". |
| `created_at`     | datetime | When the sub-task was created by the TaskDecomposer.                            |

---

### 1.4 Execution

Represents a single agent's attempt to process a SubTask.

**Primary key**

- `execution_id` – surrogate identifier for the execution attempt.

**Foreign keys**

- `subtask_id` → `SubTask.subtask_id`.
- `agent_id` → `Agent.agent_id`.

**Attributes**

| Attribute        | Type     | Description                                                                           |
|-----------------|----------|---------------------------------------------------------------------------------------|
| `execution_id`  | string   | Unique execution identifier (PK).                                                     |
| `subtask_id`    | string   | FK to `SubTask` being processed.                                                      |
| `agent_id`      | string   | FK to `Agent` that performed this execution.                                          |
| `input_context` | json     | Structured context provided to the agent (prior sub-task outputs, task description). |
| `raw_output`    | text     | The raw text output produced by the LLM agent.                                        |
| `attempt_number`| integer  | Which attempt this represents for this SubTask (1 = first attempt, 2 = first retry). |
| `status`        | string   | "running", "completed", "failed", "hallucination_detected".                           |
| `started_at`    | datetime | When the execution began.                                                             |
| `completed_at`  | datetime | When the execution ended (nullable if still running or failed mid-execution).         |

---

### 1.5 AgentMessage

Represents a communication event between agents, scoped to a Task.

**Primary key**

- `message_id` – surrogate identifier for the message.

**Foreign keys**

- `sender_agent_id` → `Agent.agent_id`.
- `receiver_agent_id` → `Agent.agent_id` (nullable for broadcast messages).
- `task_id` → `Task.task_id`.

**Attributes**

| Attribute            | Type     | Description                                                               |
|---------------------|----------|---------------------------------------------------------------------------|
| `message_id`        | string   | Unique message identifier (PK).                                           |
| `sender_agent_id`   | string   | FK to `Agent` (the sender).                                               |
| `receiver_agent_id` | string   | FK to `Agent` (the receiver); nullable for broadcast.                     |
| `task_id`           | string   | FK to `Task` (the context in which this message was sent).                |
| `message_type`      | string   | "request", "response", "broadcast", "heartbeat", "error".                 |
| `payload`           | json     | Structured message content (sub-task reference, output fragment, signal). |
| `sent_at`           | datetime | When the message was published to the Message Bus.                        |
| `delivered_at`      | datetime | When the receiver acknowledged the message (nullable).                    |

---

### 1.6 EvaluationResult

Represents the quality assessment of a single Execution.

**Primary key**

- `evaluation_id` – surrogate identifier for the evaluation.

**Foreign keys**

- `execution_id` → `Execution.execution_id`.
- `evaluator_agent_id` → `Agent.agent_id` (nullable if evaluated by an automated strategy).

**Attributes**

| Attribute              | Type     | Description                                                               |
|-----------------------|----------|---------------------------------------------------------------------------|
| `evaluation_id`       | string   | Unique evaluation identifier (PK).                                        |
| `execution_id`        | string   | FK to `Execution` being evaluated.                                        |
| `evaluator_agent_id`  | string   | FK to `Agent` that performed the evaluation (nullable for automated).     |
| `hallucination_score` | float    | Probability that the output contains hallucinated content (0.0 – 1.0).   |
| `consistency_score`   | float    | Degree to which the output is self-consistent and coherent (0.0 – 1.0).  |
| `evaluation_notes`    | text     | Free-text explanation of the verdict (e.g. detected contradiction).       |
| `verdict`             | string   | "accepted", "rejected", "needs_revision".                                 |
| `evaluated_at`        | datetime | When the evaluation was completed.                                        |

---

### 1.7 ConsensusRecord

Represents the authoritative resolved output for a SubTask, aggregated from multiple Executions.

**Primary key**

- `consensus_id` – surrogate identifier for the consensus record.

**Foreign keys**

- `task_id` → `Task.task_id`.
- `subtask_id` → `SubTask.subtask_id`.

**Attributes**

| Attribute                | Type     | Description                                                                        |
|-------------------------|----------|------------------------------------------------------------------------------------|
| `consensus_id`          | string   | Unique consensus record identifier (PK).                                           |
| `task_id`               | string   | FK to `Task` (denormalized for easier querying and analytics).                     |
| `subtask_id`            | string   | FK to `SubTask` whose Executions are being aggregated.                             |
| `strategy`              | string   | Aggregation strategy applied: "majority_vote", "weighted_average", "critic_review".|
| `final_output`          | text     | The resolved, authoritative output for this SubTask.                               |
| `participating_agent_ids` | json   | List of `agent_id` values whose Executions contributed to this consensus.          |
| `confidence_score`      | float    | Aggregate confidence in the final output (0.0 – 1.0).                             |
| `resolved_at`           | datetime | When the ConsensusService produced this record.                                    |

---

## 2. Relationships (Logical)

Cardinalities here are refined from the conceptual model.

1. **Agent–Execution**
   - `Agent` **1 → N** `Execution`
   - Implemented via `Execution.agent_id` → `Agent.agent_id`.

2. **Task–SubTask**
   - `Task` **1 → N** `SubTask`
   - Implemented via `SubTask.task_id` → `Task.task_id`.

3. **SubTask–SubTask (DAG dependency)**
   - `SubTask` **N → M** `SubTask` (self-referential dependency graph)
   - Implemented via `SubTask.dependencies` (JSON list of `subtask_id` values); may be normalized into a `subtask_dependencies` junction table at the physical level.

4. **SubTask–Execution**
   - `SubTask` **1 → N** `Execution`
   - Implemented via `Execution.subtask_id` → `SubTask.subtask_id`.

5. **Execution–EvaluationResult**
   - `Execution` **1 → N** `EvaluationResult`
   - Implemented via `EvaluationResult.execution_id` → `Execution.execution_id`.

6. **SubTask–ConsensusRecord**
   - `SubTask` **1 → 0..1** `ConsensusRecord`
   - Implemented via `ConsensusRecord.subtask_id` → `SubTask.subtask_id` (unique constraint).

7. **Task–ConsensusRecord**
   - `Task` **1 → N** `ConsensusRecord`
   - Implemented via `ConsensusRecord.task_id` → `Task.task_id` (denormalized FK).

8. **Agent–AgentMessage (Sender)**
   - `Agent` **1 → N** `AgentMessage`
   - Implemented via `AgentMessage.sender_agent_id` → `Agent.agent_id`.

9. **Agent–AgentMessage (Receiver)**
   - `Agent` **0..1 → N** `AgentMessage` (nullable receiver for broadcast)
   - Implemented via `AgentMessage.receiver_agent_id` → `Agent.agent_id` (nullable FK).

10. **Task–AgentMessage**
    - `Task` **1 → N** `AgentMessage`
    - Implemented via `AgentMessage.task_id` → `Task.task_id`.

11. **Agent–EvaluationResult (Evaluator)**
    - `Agent` **0..1 → N** `EvaluationResult` (nullable for automated evaluation)
    - Implemented via `EvaluationResult.evaluator_agent_id` → `Agent.agent_id` (nullable FK).

---
