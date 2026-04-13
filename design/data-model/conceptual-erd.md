# Conceptual Entity–Relationship Model

This document defines the Conceptual Entity-Relationship model for the Distributed LLM Agents system.

At this level, we model only the **core domain entities** and the **relationships between them**, without attributes, data types, primary and foreign keys, or implementation concerns. The conceptual model is:

- **Technology-agnostic** (no PostgreSQL or schema details).
- **Role-agnostic** (Planner, Executor, Critic, Synthesizer are *roles* of `Agent`).
- A bridge between the **System Architecture** (`system-architecture.md`) and the **Logical / Physical ERDs**.

---

## 1. Modeling Scope

The conceptual model focuses on the core flows of distributed agent execution:

- Users submitting Tasks to the system.
- Tasks being decomposed into SubTasks with dependency relationships.
- Agents claiming and executing SubTasks, producing Executions.
- Executions being evaluated for hallucination and reliability.
- Multiple Executions for the same SubTask being resolved into a ConsensusRecord.
- Agents communicating with each other through AgentMessages.

The core entities at this level are:

- `Agent`
- `Task`
- `SubTask`
- `Execution`
- `AgentMessage`
- `EvaluationResult`
- `ConsensusRecord`

---

## 2. Core Entities

> Note: At the conceptual level, entities are defined by their **role in the domain**, not by their attributes.

### 2.1 Agent

Represents an **LLM-backed agent instance** registered in the system:
- Can act as **Planner**, **Executor**, **Critic**, or **Synthesizer**.
- Claims and executes SubTasks; sends and receives AgentMessages.
- Subject to health monitoring and fault isolation.

### 2.2 Task

Represents a **high-level computational problem** submitted by an external user or system:
- The unit of work that enters the system from the outside.
- Must be decomposed into SubTasks before any agent can work on it.
- Completed when all SubTasks are resolved.

### 2.3 SubTask

Represents a **unit of decomposed work** derived from a Task:
- The granular assignment that is dispatched to a specific Agent.
- May depend on the outputs of other SubTasks (forming a DAG).
- Can be retried or reassigned if the executing Agent fails.

### 2.4 Execution

Represents **a single agent's attempt** to process a SubTask:
- Produced by an Agent when it completes (or fails) a SubTask.
- Contains the raw output from the LLM call.
- Multiple Executions can exist for the same SubTask (retries or redundant dispatch).

### 2.5 AgentMessage

Represents **a communication event** between two agents (or broadcast to all):
- Used for coordination, information sharing, result propagation, and heartbeat signals.
- Scoped to a Task context.
- Captured for audit and research analysis.

### 2.6 EvaluationResult

Represents **the quality assessment** of a single Execution:
- Produced by the EvaluatorService after each Execution.
- Captures hallucination likelihood and output consistency scores.
- Drives the decision to accept, reject, or request revision of an Execution.

### 2.7 ConsensusRecord

Represents **the authoritative resolved output** for a SubTask:
- Produced by the ConsensusService when multiple Executions exist for the same SubTask.
- Captures which agents participated, the aggregation strategy used, and the confidence score.
- The final output that the Orchestrator uses to advance the task DAG.

---

## 3. Relationships

This section describes **how entities relate** in the domain. Cardinalities here are conceptual and will be refined in the Logical ERD.

1. **Agent — Execution**
   - An `Agent` produces one or more `Execution` records over its lifetime.
   - An `Execution` is always produced by exactly one `Agent`.
   - **Conceptual relationship:**
     - `Agent` 1 → * `Execution`

2. **Task — SubTask**
   - A `Task` is decomposed into one or more `SubTask` entries.
   - A `SubTask` belongs to exactly one `Task`.
   - **Conceptual relationship:**
     - `Task` 1 → * `SubTask`

3. **SubTask — SubTask (Dependency)**
   - A `SubTask` may depend on zero or more other `SubTask` entries within the same Task (DAG edges).
   - This is a self-referential relationship capturing the dependency graph.
   - **Conceptual relationship:**
     - `SubTask` * → * `SubTask` (depends on)

4. **SubTask — Execution**
   - A `SubTask` can have one or more `Execution` attempts (initial attempt plus retries or redundant agents).
   - An `Execution` is always associated with exactly one `SubTask`.
   - **Conceptual relationship:**
     - `SubTask` 1 → * `Execution`

5. **Execution — EvaluationResult**
   - An `Execution` can be assessed by one or more `EvaluationResult` records (e.g., multiple evaluator strategies).
   - An `EvaluationResult` always refers to exactly one `Execution`.
   - **Conceptual relationship:**
     - `Execution` 1 → * `EvaluationResult`

6. **SubTask — ConsensusRecord**
   - A `SubTask` may produce one `ConsensusRecord` once sufficient Executions have been evaluated.
   - A `ConsensusRecord` is tied to exactly one `SubTask`.
   - **Conceptual relationship:**
     - `SubTask` 1 → 0..1 `ConsensusRecord`

7. **Task — ConsensusRecord**
   - A `Task` is associated with one or more `ConsensusRecord` entries (one per resolved SubTask).
   - **Conceptual relationship:**
     - `Task` 1 → * `ConsensusRecord`

8. **Agent — AgentMessage (Sender)**
   - An `Agent` sends one or more `AgentMessage` records.
   - An `AgentMessage` has exactly one sender `Agent`.
   - **Conceptual relationship:**
     - `Agent` (Sender) 1 → * `AgentMessage`

9. **Agent — AgentMessage (Receiver)**
   - An `Agent` receives one or more `AgentMessage` records.
   - An `AgentMessage` targets zero or one receiver `Agent` (null for broadcast).
   - **Conceptual relationship:**
     - `Agent` (Receiver) 1 → * `AgentMessage` (nullable for broadcast)

10. **Task — AgentMessage**
    - An `AgentMessage` is scoped to a specific `Task` context.
    - A `Task` accumulates many `AgentMessage` records throughout its execution.
    - **Conceptual relationship:**
      - `Task` 1 → * `AgentMessage`

---
