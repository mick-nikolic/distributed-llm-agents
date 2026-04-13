# Physical Entity–Relationship Model (PostgreSQL)

This document defines the **Physical Entity–Relationship Model** for the Distributed LLM Agents system.

It refines the Logical Entity-Relationship model into a fully specified PostgreSQL database schema, including:

- Tables
- Column data types
- Primary keys (PK)
- Foreign keys (FK)
- Constraints
- Indexes
- CHECK conditions
- Status enumerations
- Physical Entity-Relationship diagram (`diagrams/physical-erd.mmd`)

---

## 1. Global Design Decisions

- **Database**: PostgreSQL 15+
- **Primary keys**: `UUID` with `DEFAULT gen_random_uuid()`
- **Timestamps**: `TIMESTAMPTZ` (timezone-aware) for all time values
- **Structured fields**: Stored as `JSONB` (capabilities, dependencies, input_context, payload, participating_agent_ids)
- **Text fields**: `TEXT` for unbounded strings (description, raw_output, evaluation_notes, final_output)
- **Score fields**: `NUMERIC(5,4)` for probability/confidence values (range 0.0000 – 1.0000)
- **Status fields**: Stored as `TEXT` with strict `CHECK` constraints (upgradeable to ENUM)
- **Naming conventions**:
  - Tables: snake_case plural (e.g. `agents`, `tasks`, `subtasks`)
  - Columns: snake_case
  - Indexes: `idx_<table>_<column>`
  - Foreign key constraints: `fk_<table>_<column>`

> NOTE: The schema is normalized (3NF). The `task_id` in `consensus_records` is a selective denormalization for efficient per-task analytics queries. The SubTask dependency graph is stored as `JSONB` and can be normalized into a `subtask_dependencies` junction table if graph traversal becomes a primary query pattern.

---

## 2. Required Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- required for gen_random_uuid()
```

---

## 3. Tables

### 3.1 `agents`

```sql
CREATE TABLE agents (
    agent_id        UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_role      TEXT        NOT NULL
                                CHECK (agent_role IN ('planner', 'executor', 'critic', 'synthesizer')),
    llm_model       TEXT        NOT NULL,
    llm_provider    TEXT        NOT NULL
                                CHECK (llm_provider IN ('openai', 'anthropic', 'local')),
    capabilities    JSONB       NOT NULL DEFAULT '[]',
    status          TEXT        NOT NULL DEFAULT 'idle'
                                CHECK (status IN ('idle', 'busy', 'quarantined', 'offline')),
    registered_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agents_role   ON agents (agent_role);
CREATE INDEX idx_agents_status ON agents (status);
```

---

### 3.2 `tasks`

```sql
CREATE TABLE tasks (
    task_id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    title            TEXT        NOT NULL,
    description      TEXT        NOT NULL,
    complexity_level TEXT        NOT NULL DEFAULT 'moderate'
                                 CHECK (complexity_level IN ('simple', 'moderate', 'complex')),
    status           TEXT        NOT NULL DEFAULT 'pending'
                                 CHECK (status IN ('pending', 'decomposing', 'in_progress', 'completed', 'failed')),
    submitted_by     TEXT        NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at     TIMESTAMPTZ
);

CREATE INDEX idx_tasks_status       ON tasks (status);
CREATE INDEX idx_tasks_submitted_by ON tasks (submitted_by);
CREATE INDEX idx_tasks_created_at   ON tasks (created_at DESC);
```

---

### 3.3 `subtasks`

```sql
CREATE TABLE subtasks (
    subtask_id      UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID        NOT NULL
                                REFERENCES tasks (task_id) ON DELETE CASCADE,
    description     TEXT        NOT NULL,
    required_role   TEXT        NOT NULL
                                CHECK (required_role IN ('planner', 'executor', 'critic', 'synthesizer')),
    dependencies    JSONB       NOT NULL DEFAULT '[]',
    sequence_order  INTEGER     NOT NULL DEFAULT 0,
    status          TEXT        NOT NULL DEFAULT 'pending'
                                CHECK (status IN ('pending', 'assigned', 'in_progress', 'completed', 'failed', 'retrying')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_subtasks_task FOREIGN KEY (task_id) REFERENCES tasks (task_id)
);

CREATE INDEX idx_subtasks_task_id ON subtasks (task_id);
CREATE INDEX idx_subtasks_status  ON subtasks (status);
```

---

### 3.4 `executions`

```sql
CREATE TABLE executions (
    execution_id    UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    subtask_id      UUID        NOT NULL
                                REFERENCES subtasks (subtask_id) ON DELETE CASCADE,
    agent_id        UUID        NOT NULL
                                REFERENCES agents (agent_id),
    input_context   JSONB       NOT NULL DEFAULT '{}',
    raw_output      TEXT,
    attempt_number  INTEGER     NOT NULL DEFAULT 1
                                CHECK (attempt_number >= 1),
    status          TEXT        NOT NULL DEFAULT 'running'
                                CHECK (status IN ('running', 'completed', 'failed', 'hallucination_detected')),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,

    CONSTRAINT fk_executions_subtask FOREIGN KEY (subtask_id) REFERENCES subtasks (subtask_id),
    CONSTRAINT fk_executions_agent   FOREIGN KEY (agent_id)   REFERENCES agents (agent_id)
);

CREATE INDEX idx_executions_subtask_id ON executions (subtask_id);
CREATE INDEX idx_executions_agent_id   ON executions (agent_id);
CREATE INDEX idx_executions_status     ON executions (status);
```

---

### 3.5 `agent_messages`

```sql
CREATE TABLE agent_messages (
    message_id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    sender_agent_id     UUID        NOT NULL
                                    REFERENCES agents (agent_id),
    receiver_agent_id   UUID
                                    REFERENCES agents (agent_id),   -- NULL = broadcast
    task_id             UUID        NOT NULL
                                    REFERENCES tasks (task_id) ON DELETE CASCADE,
    message_type        TEXT        NOT NULL
                                    CHECK (message_type IN ('request', 'response', 'broadcast', 'heartbeat', 'error')),
    payload             JSONB       NOT NULL DEFAULT '{}',
    sent_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    delivered_at        TIMESTAMPTZ,

    CONSTRAINT fk_messages_sender   FOREIGN KEY (sender_agent_id)   REFERENCES agents (agent_id),
    CONSTRAINT fk_messages_receiver FOREIGN KEY (receiver_agent_id) REFERENCES agents (agent_id),
    CONSTRAINT fk_messages_task     FOREIGN KEY (task_id)           REFERENCES tasks (task_id)
);

CREATE INDEX idx_agent_messages_task_id        ON agent_messages (task_id);
CREATE INDEX idx_agent_messages_sender         ON agent_messages (sender_agent_id);
CREATE INDEX idx_agent_messages_receiver       ON agent_messages (receiver_agent_id);
CREATE INDEX idx_agent_messages_message_type   ON agent_messages (message_type);
CREATE INDEX idx_agent_messages_sent_at        ON agent_messages (sent_at DESC);
```

---

### 3.6 `evaluation_results`

```sql
CREATE TABLE evaluation_results (
    evaluation_id          UUID           PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id           UUID           NOT NULL
                                          REFERENCES executions (execution_id) ON DELETE CASCADE,
    evaluator_agent_id     UUID
                                          REFERENCES agents (agent_id),           -- NULL = automated
    hallucination_score    NUMERIC(5,4)   NOT NULL
                                          CHECK (hallucination_score BETWEEN 0.0 AND 1.0),
    consistency_score      NUMERIC(5,4)   NOT NULL
                                          CHECK (consistency_score BETWEEN 0.0 AND 1.0),
    evaluation_notes       TEXT,
    verdict                TEXT           NOT NULL
                                          CHECK (verdict IN ('accepted', 'rejected', 'needs_revision')),
    evaluated_at           TIMESTAMPTZ    NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_evaluations_execution FOREIGN KEY (execution_id)       REFERENCES executions (execution_id),
    CONSTRAINT fk_evaluations_evaluator FOREIGN KEY (evaluator_agent_id) REFERENCES agents (agent_id)
);

CREATE INDEX idx_evaluations_execution_id ON evaluation_results (execution_id);
CREATE INDEX idx_evaluations_verdict      ON evaluation_results (verdict);
CREATE INDEX idx_evaluations_evaluator    ON evaluation_results (evaluator_agent_id);
```

---

### 3.7 `consensus_records`

```sql
CREATE TABLE consensus_records (
    consensus_id             UUID           PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id                  UUID           NOT NULL
                                            REFERENCES tasks (task_id) ON DELETE CASCADE,
    subtask_id               UUID           NOT NULL
                                            REFERENCES subtasks (subtask_id) ON DELETE CASCADE,
    strategy                 TEXT           NOT NULL
                                            CHECK (strategy IN ('majority_vote', 'weighted_average', 'critic_review')),
    final_output             TEXT           NOT NULL,
    participating_agent_ids  JSONB          NOT NULL DEFAULT '[]',
    confidence_score         NUMERIC(5,4)   NOT NULL
                                            CHECK (confidence_score BETWEEN 0.0 AND 1.0),
    resolved_at              TIMESTAMPTZ    NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_consensus_subtask     UNIQUE (subtask_id),
    CONSTRAINT fk_consensus_task        FOREIGN KEY (task_id)    REFERENCES tasks (task_id),
    CONSTRAINT fk_consensus_subtask     FOREIGN KEY (subtask_id) REFERENCES subtasks (subtask_id)
);

CREATE INDEX idx_consensus_task_id    ON consensus_records (task_id);
CREATE INDEX idx_consensus_strategy   ON consensus_records (strategy);
CREATE INDEX idx_consensus_confidence ON consensus_records (confidence_score DESC);
```

---

## 4. Summary of Tables

| Table                | PK           | Key FKs                                               | Notes                                        |
|---------------------|--------------|-------------------------------------------------------|----------------------------------------------|
| `agents`             | `agent_id`   | —                                                     | Registry of all LLM agent instances          |
| `tasks`              | `task_id`    | —                                                     | Top-level problems submitted by clients       |
| `subtasks`           | `subtask_id` | `task_id`                                             | Decomposed units; DAG via `dependencies` JSONB |
| `executions`         | `execution_id` | `subtask_id`, `agent_id`                            | Each agent attempt at a sub-task              |
| `agent_messages`     | `message_id` | `sender_agent_id`, `receiver_agent_id`, `task_id`     | Inter-agent communication log                 |
| `evaluation_results` | `evaluation_id` | `execution_id`, `evaluator_agent_id`              | Quality / hallucination assessments           |
| `consensus_records`  | `consensus_id` | `task_id`, `subtask_id`                             | Aggregated final outputs per sub-task         |
