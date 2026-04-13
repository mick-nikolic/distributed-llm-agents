# Class Diagram – Distributed LLM Agents System

This document describes the **class-level architecture** of the Distributed LLM Agents system.

It complements:

- `design/system-architecture.md` (C4 + 4+1 views)
- `design/data-model/*` (Conceptual / Logical / Physical Entity-Relationship diagrams)

and focuses on how the **API layer**, **orchestration services**, **agent roles**, **evaluator**, **consensus**, **fault management**, **repositories**, and **domain models** are structured around the core entities:

- `Agent`
- `Task`
- `SubTask`
- `Execution`
- `AgentMessage`
- `EvaluationResult`
- `ConsensusRecord`

---

## 1. Architectural Style

The backend is implemented following a **layered, event-driven** architecture:

- **API Layer (Presentation)**
  - REST endpoints for task submission, status queries, and result retrieval.
  - Responsible for HTTP concerns: routing, authentication, request/response mapping.

- **Orchestration Layer (Application / Domain Logic)**
  - `OrchestratorService`: central coordinator; owns the task lifecycle.
  - `TaskDecomposer`: converts a raw Task into a dependency-ordered DAG of SubTasks.
  - `AgentRegistry`: maintains the catalog of registered agents and their availability.

- **Agent Layer**
  - `AgentProxy`: represents a remote LLM agent; handles dispatch, timeout, and result collection.
  - Concrete agent roles: `PlannerAgent`, `ExecutorAgent`, `CriticAgent`, `SynthesizerAgent`.
  - `LLMProviderAdapter`: abstracts calls to external LLM APIs (OpenAI, Anthropic, local).

- **Evaluation Layer**
  - `EvaluatorService`: scores each Execution for hallucination and consistency.
  - `HallucinationDetector`: pluggable strategy interface for detection algorithms.

- **Consensus Layer**
  - `ConsensusService`: aggregates multiple Executions for the same SubTask.
  - `ConsensusStrategy`: pluggable interface; concrete implementations: `MajorityVoteStrategy`, `WeightedAverageStrategy`, `CriticReviewStrategy`.

- **Fault Management Layer**
  - `FaultManager`: monitors heartbeats, tracks failure counts, decides retry or reassign.
  - `CircuitBreaker`: per-agent state machine (CLOSED → OPEN → HALF-OPEN).

- **Repository Layer (Persistence)**
  - Repositories encapsulate all SQLAlchemy database access.
  - One repository per aggregate root entity.

- **Domain Model Layer (Data / ORM)**
  - SQLAlchemy ORM models mapped to physical tables.

- **Schema Layer (DTOs)**
  - Pydantic models for request/response bodies; decouple API contracts from domain models.

The diagram in `diagrams/class-diagram.mmd` visualizes these layers and their dependencies.

---

## 2. Package Overview

```text
app/
  main.py
  api/
    v1/
      routers/
        tasks.py
        agents.py
        executions.py
        evaluations.py
        consensus.py
  orchestrator/
    orchestrator_service.py
    task_decomposer.py
    dag.py
  agents/
    agent_registry.py
    agent_proxy.py
    roles/
      planner_agent.py
      executor_agent.py
      critic_agent.py
      synthesizer_agent.py
    adapters/
      openai_adapter.py
      anthropic_adapter.py
      local_llm_adapter.py
  evaluator/
    evaluator_service.py
    strategies/
      hallucination_detector.py
      consistency_scorer.py
      cross_reference_detector.py
  consensus/
    consensus_service.py
    strategies/
      majority_vote_strategy.py
      weighted_average_strategy.py
      critic_review_strategy.py
  fault/
    fault_manager.py
    circuit_breaker.py
    retry_policy.py
  messaging/
    message_bus.py
    topics.py
    router.py
  repositories/
    task_repository.py
    subtask_repository.py
    execution_repository.py
    agent_repository.py
    agent_message_repository.py
    evaluation_repository.py
    consensus_repository.py
  models/
    agent.py
    task.py
    subtask.py
    execution.py
    agent_message.py
    evaluation_result.py
    consensus_record.py
  schemas/
    task.py
    subtask.py
    execution.py
    evaluation.py
    consensus.py
    agent.py
  core/
    config.py
    database.py
```

---

## 3. Key Classes and Responsibilities

### 3.1 OrchestratorService

Central coordinator of the task lifecycle.

**Responsibilities:**
- Receive a `Task` from the API layer.
- Invoke `TaskDecomposer` to produce a DAG of `SubTask` objects.
- Persist the Task and SubTasks via their repositories.
- Schedule SubTasks onto the Message Bus as their dependencies are resolved.
- Collect Execution results and trigger Consensus when multiple results exist.
- Mark the Task as `completed` or `failed` when all SubTasks are resolved.

**Key methods:**
- `submit_task(task_create: TaskCreate) -> Task`
- `on_execution_received(execution: Execution) -> None`
- `resolve_task(task_id: str) -> Task`

---

### 3.2 TaskDecomposer

Converts a high-level Task description into an ordered set of SubTasks.

**Responsibilities:**
- Call the Planner LLM Agent to generate a decomposition plan.
- Parse the plan into `SubTask` objects with explicit dependency references.
- Construct and validate the DAG (detect cycles, validate dependencies).

**Key methods:**
- `decompose(task: Task) -> DAG[SubTask]`
- `validate_dag(dag: DAG) -> bool`

---

### 3.3 AgentProxy

Represents a remote LLM agent from the Orchestrator's perspective.

**Responsibilities:**
- Dispatch a SubTask to the underlying agent (via Message Bus or direct call).
- Enforce timeout policies.
- Report the resulting Execution back to the Orchestrator.
- Send and receive heartbeat signals to the Fault Manager.

**Key methods:**
- `dispatch(subtask: SubTask) -> Execution`
- `heartbeat() -> None`

---

### 3.4 EvaluatorService

Assesses the quality and reliability of each Execution.

**Responsibilities:**
- Apply one or more `HallucinationDetector` strategies to the raw output.
- Compute a `hallucination_score` and `consistency_score`.
- Emit a verdict (`accepted`, `rejected`, `needs_revision`).
- Persist the `EvaluationResult` and notify the Fault Manager on rejection.

**Key methods:**
- `evaluate(execution: Execution) -> EvaluationResult`
- `register_strategy(strategy: HallucinationDetector) -> None`

---

### 3.5 ConsensusService

Aggregates multiple Execution outputs for the same SubTask.

**Responsibilities:**
- Collect all accepted Executions for a given SubTask.
- Apply the configured `ConsensusStrategy`.
- Produce a `ConsensusRecord` with a `final_output` and `confidence_score`.
- Notify the Orchestrator that the SubTask is definitively resolved.

**Key methods:**
- `resolve(subtask_id: str, executions: list[Execution]) -> ConsensusRecord`
- `set_strategy(strategy: ConsensusStrategy) -> None`

---

### 3.6 FaultManager

Monitors agent health and enforces fault isolation.

**Responsibilities:**
- Track per-agent failure counts and evaluation rejection rates.
- Manage the `CircuitBreaker` state machine for each agent.
- Trigger SubTask reassignment when an agent is quarantined.
- Emit alerts for persistent failures.

**Key methods:**
- `on_heartbeat_missed(agent_id: str) -> None`
- `on_evaluation_rejected(agent_id: str, execution_id: str) -> None`
- `get_circuit_state(agent_id: str) -> CircuitState`

---

### 3.7 Repositories

Each repository provides a stable interface over the database for its entity:

| Repository | Entity | Key Methods |
|---|---|---|
| `TaskRepository` | `Task` | `create`, `get_by_id`, `update_status`, `list_by_status` |
| `SubTaskRepository` | `SubTask` | `create_bulk`, `get_by_task`, `update_status`, `get_ready` |
| `ExecutionRepository` | `Execution` | `create`, `get_by_subtask`, `get_latest_by_agent` |
| `AgentRepository` | `Agent` | `register`, `get_available`, `update_status` |
| `AgentMessageRepository` | `AgentMessage` | `create`, `get_by_task`, `get_by_sender` |
| `EvaluationRepository` | `EvaluationResult` | `create`, `get_by_execution`, `list_rejected` |
| `ConsensusRepository` | `ConsensusRecord` | `create`, `get_by_subtask`, `get_by_task` |

---

## 4. Relationship to Other Design Documents

This document is intended to be read alongside:

- `system-architecture.md` — high-level C4 / 4+1 architecture.
- `design-patterns.md` — patterns used across the class structure.
- `data-model/conceptual-erd.md` — conceptual domain relationships.
- `data-model/logical-erd.md` and `data-model/physical-erd.md` — logical and physical database models.

Together, these documents show:

1. **What** the system does (research questions and goals).
2. **How** it is structured (architecture and class diagrams).
3. **Which** design patterns were chosen to ensure reliability and extensibility.
