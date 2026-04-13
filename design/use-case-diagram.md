# UML Use Case Diagram

This document describes the UML Use Case Diagram for the Distributed LLM Agents system, located at `~/diagrams/use-case-diagram.puml` (PlantUML source).

---

## Overview

The diagram captures every externally visible behaviour of the system and the relationships between those behaviours. It follows the four standard UML use-case relationship types:

| Relationship    | Notation                                        | Meaning |
|-----------------|-------------------------------------------------|---------|
| Association     | Solid line between actor and use case           | The actor participates in the use case |
| `<<include>>`   | Dashed arrow from base → included use case      | The included behaviour is **always** executed as part of the base |
| `<<extend>>`    | Dashed arrow from extension → base use case     | The extension adds **optional or conditional** behaviour to the base |
| Generalization  | Solid arrow with hollow head (child → parent)   | The child actor/use-case inherits all associations of the parent |

---

## Actors

### Primary Actors (initiate use cases)

| Actor | Description |
|-------|-------------|
| **External User / System** | Submits tasks, queries status, and retrieves final results via the REST API. |
| **System Administrator** | Configures circuit-breaker thresholds, consensus strategies, and overall system parameters. |
| **Research Analyst** | Queries the Experiment Tracker and exports reliability metrics for offline analysis. |

### Secondary Actors (respond to system requests)

| Actor | Description |
|-------|-------------|
| **LLM Provider** | Abstract parent; represents any external inference endpoint. |
| **OpenAI API** | Concrete LLM provider — specializes `LLM Provider` via generalization. |
| **Anthropic API** | Concrete LLM provider — specializes `LLM Provider` via generalization. |
| **Local Inference** | Concrete LLM provider — specializes `LLM Provider` via generalization. |
| **Agent** | Abstract parent actor for all agent roles; executes SubTasks and emits heartbeats. |
| **Planner Agent** | Specializes `Agent` via generalization; focuses on planning sub-tasks. |
| **Executor Agent** | Specializes `Agent` via generalization; executes computational sub-tasks. |
| **Critic Agent** | Specializes `Agent` via generalization; reviews and critiques outputs. |
| **Synthesizer Agent** | Specializes `Agent` via generalization; aggregates partial results into a final answer. |

### Generalization hierarchies

```
Agent
├── Planner Agent
├── Executor Agent
├── Critic Agent
└── Synthesizer Agent

LLM Provider
├── OpenAI API
├── Anthropic API
└── Local Inference
```

---

## Use Cases

### Task Submission & Query

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Submit Task** | External User | Entry point; triggers the full orchestration pipeline. |
| **Authenticate Request** | — | Validates the caller's identity and authorisation. |
| **Monitor Task Status** | External User | Polls the current state of a submitted task. |
| **Retrieve Task Result** | External User | Fetches the final aggregated answer for a completed task. |
| **Configure Decomposition Strategy** | External User | Optionally overrides the default task-decomposition algorithm at submission time. |

### Orchestration

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Decompose Task into SubTasks** | — | Produces a DAG of sub-tasks from the top-level task. |
| **Schedule SubTask** | — | Determines execution order respecting DAG dependencies. |
| **Dispatch SubTask to Agent** | — | Routes each sub-task to a suitable agent via the Message Bus. |

### Agent Execution

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Execute SubTask** | Agent | Runs a sub-task by calling an LLM and publishing the result. |
| **Call LLM API** | LLM Provider | The actual inference request sent to an external LLM endpoint. |
| **Store Prompt / Response Artifacts** | — | Persists the raw prompt and LLM response to Object Storage for audit and evaluation. |
| **Retry Execution** | — | Re-runs a sub-task on the same or a different agent after a failure signal. |

### Evaluation

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Evaluate Execution** | — | Scores an execution result for hallucination likelihood and internal consistency. |
| **Read Artifacts from Object Storage** | — | Retrieves stored prompts and responses to supply the evaluation pipeline. |
| **Detect Hallucinations** | — | Applies configured hallucination-detection strategies (cross-reference, consistency, fact-grounding). |
| **Trigger Fault Recovery** | — | Invoked when evaluation rejects an execution; signals the Fault Manager to reassign or quarantine. |

### Consensus

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Aggregate Consensus** | — | Reconciles multiple execution results for the same sub-task into one authoritative output. |
| **Apply Consensus Strategy** | — | Executes the configured aggregation strategy (majority vote, weighted average, or critic review). |
| **Escalate to Human Review** | Critic Agent | Involves the Critic Agent when no automated consensus can be reached with sufficient confidence. |

### Fault Management

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Monitor Agent Health** | System Administrator | Tracks the liveness and reliability of all agents. |
| **Track Heartbeats** | Agent | Agents emit periodic heartbeat signals consumed by the Fault Manager. |
| **Isolate Failing Agent** | — | Trips the circuit breaker for an agent whose heartbeat or evaluation failure rate exceeds the threshold. |

### Administration

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Configure System** | System Administrator | Sets deployment-wide parameters (thresholds, strategies, API keys). |
| **Set Circuit Breaker Thresholds** | — | Defines the heartbeat-miss count and failure-rate limits that trip an agent's circuit breaker. |
| **Override Consensus Strategy** | — | Replaces the active consensus aggregation strategy without redeployment. |

### Research & Analytics

| Use Case | Actor | Description |
|----------|-------|-------------|
| **Analyze Research Metrics** | Research Analyst | Queries aggregate reliability metrics from the Experiment Tracker. |
| **Query Experiment Tracker** | — | Fetches raw evaluation logs and consensus records from the analytics store. |
| **Export Metrics Report** | — | Produces a formatted report (CSV, JSON, or notebook) for offline analysis. |

---

## Include Relationships

These use cases are **always** invoked as a mandatory sub-behaviour of the base use case.

| Base Use Case | Included Use Case | Rationale |
|---|---|---|
| Submit Task | Authenticate Request | All API entry points require authorisation. |
| Monitor Task Status | Authenticate Request | " |
| Retrieve Task Result | Authenticate Request | " |
| Submit Task | Decompose Task into SubTasks | Decomposition is inseparable from submission. |
| Decompose Task into SubTasks | Schedule SubTask | Scheduling immediately follows decomposition. |
| Schedule SubTask | Dispatch SubTask to Agent | Dispatch is the concrete outcome of scheduling. |
| Execute SubTask | Call LLM API | Every execution requires an LLM inference call. |
| Execute SubTask | Store Prompt / Response Artifacts | Every prompt and response is persisted for audit. |
| Evaluate Execution | Read Artifacts from Object Storage | Evaluation always reads the stored prompt/response pair. |
| Evaluate Execution | Detect Hallucinations | Hallucination detection is the core evaluation step. |
| Retrieve Task Result | Aggregate Consensus | A result is only served once consensus has been reached. |
| Aggregate Consensus | Apply Consensus Strategy | Strategy execution is the mechanism of aggregation. |
| Monitor Agent Health | Track Heartbeats | Health monitoring is built on heartbeat signals. |
| Configure System | Set Circuit Breaker Thresholds | Threshold configuration is always part of system setup. |
| Analyze Research Metrics | Query Experiment Tracker | Metrics analysis always queries the tracker. |

---

## Extend Relationships

These use cases add **optional or conditional** behaviour to the base use case.

| Extending Use Case | Base Use Case | Condition / Trigger |
|---|---|---|
| Configure Decomposition Strategy | Submit Task | Only when the submitter provides a non-default strategy. |
| Retry Execution | Execute SubTask | Only when the execution fails or is rejected by the Evaluator. |
| Trigger Fault Recovery | Evaluate Execution | Only when the Evaluator verdict is `rejected`. |
| Escalate to Human Review | Aggregate Consensus | Only when automated consensus confidence falls below threshold. |
| Isolate Failing Agent | Monitor Agent Health | Only when heartbeat misses or failure rate exceeds the circuit-breaker threshold. |
| Override Consensus Strategy | Configure System | Only when the administrator explicitly supplies a strategy override. |
| Export Metrics Report | Analyze Research Metrics | Only when the analyst requests a formatted output artefact. |

---

## Rendering the Diagram

The diagram source is written in **PlantUML**. To render it:

```bash
# With the PlantUML JAR
java -jar plantuml.jar diagrams/use-case-diagram.puml

# With the PlantUML CLI (Homebrew / apt)
plantuml diagrams/use-case-diagram.puml

# VS Code: install the "PlantUML" extension (jebbs.plantuml), then
# open the .puml file and press Alt+D to preview.
```

---

## Relationship to Other Design Documents

| Document | Relationship |
|---|---|
| `design/system-architecture.md` | Provides the container/runtime view that backs the use cases defined here. |
| `diagrams/system-architecture.mmd` | Shows the communication paths between the components that implement these use cases. |
| `design/class-diagrams.md` | Details the classes that realise each use case internally. |
| `design/design-patterns.md` | Documents the patterns (Strategy, Circuit Breaker, Repository, etc.) used within each use case. |
