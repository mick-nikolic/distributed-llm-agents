# Design Patterns – Distributed LLM Agents System

This document describes the **software design patterns** applied in the Distributed LLM Agents system.

It connects the high-level architecture (`system-architecture.md`), data model (`data-model/*`), and class structure (`class-diagrams.md`) to well-known patterns from software engineering, adapted to the domain of distributed AI systems.

The goal is to make explicit:

- Which patterns we use.
- Why we chose them.
- Where they appear in the system.
- How they support reliability, extensibility, and research reproducibility.

---

## 1. Architectural Patterns

### 1.1 Layered Architecture

**Pattern:** Layered Architecture
**Where:** Entire system (API → Orchestration → Agent/Evaluator/Consensus → Repository → Data).

We structure the system into well-defined layers:

1. **API Layer** – REST endpoints for task submission, status queries, result retrieval.
2. **Orchestration Layer** – OrchestratorService, TaskDecomposer, AgentRegistry; owns the task lifecycle.
3. **Specialist Layer** – EvaluatorService, ConsensusService, FaultManager; domain-specific processing.
4. **Agent Layer** – AgentProxy and role-specific agents; interface to external LLM providers.
5. **Repository Layer** – Encapsulates all database access; hides SQLAlchemy from upper layers.
6. **Domain Model Layer** – ORM entities; one-to-one mapping to physical tables.

This separation:
- Keeps API controllers thin and focused on HTTP concerns.
- Centralizes orchestration logic in OrchestratorService.
- Isolates persistence in repositories, enabling independent testing.

---

### 1.2 Orchestrator Pattern

**Pattern:** Orchestrator (centralized coordinator)
**Where:** `OrchestratorService`; overall task lifecycle management.

The Orchestrator is the single authority over task state. It:
- Accepts a Task from the API, delegates decomposition to `TaskDecomposer`.
- Publishes SubTasks to the Message Bus as their DAG dependencies are resolved.
- Collects Execution results, triggers Evaluation, and then Consensus.
- Advances the task DAG and marks the Task as complete or failed.

**Why Orchestrator over Choreography:**
- Choreography (each agent reacts to events independently) distributes control but makes global state harder to reason about and debug — critical for a research system requiring reproducibility.
- The Orchestrator provides a single source of truth for task state, which directly supports answering the research question: *how does architecture affect reliability?*

---

### 1.3 Event-Driven Architecture (Message Bus)

**Pattern:** Event-Driven / Publish-Subscribe
**Where:** `MessageBus`, inter-agent communication, Evaluator and FaultManager notifications.

Agents do not call each other directly. Instead:
- The Orchestrator publishes SubTask assignments to the Message Bus.
- Agents subscribe to topics matching their roles.
- Execution results, heartbeats, and error signals are published as events.
- EvaluatorService and FaultManager subscribe to relevant topics.

Benefits:
- Agents are decoupled; adding a new agent role does not require modifying existing agents.
- Supports parallelism: multiple agents can process independent SubTasks simultaneously.
- Enables asynchronous retry without blocking the Orchestrator.

---

## 2. Object-Level Patterns

### 2.1 Strategy Pattern (Task Decomposition, Consensus, Hallucination Detection)

**Pattern:** Strategy
**Where:** `TaskDecomposer` (decomposition strategy), `ConsensusService` (aggregation strategy), `EvaluatorService` (detection strategy).

Each of these areas has a **pluggable behavior** abstracted behind an interface:

- **DecompositionStrategy**: `LLMPlannerDecomposition`, `RuleBasedDecomposition`.
- **ConsensusStrategy**: `MajorityVoteStrategy`, `WeightedAverageStrategy`, `CriticReviewStrategy`.
- **HallucinationDetector**: `CrossReferenceDetector`, `ConsistencyScorer`, `FactGroundingDetector`.

Concrete strategies implement a common interface and can be swapped at configuration time. This is central to the research goal: **different architectural configurations can be compared by varying strategies**.

---

### 2.2 Circuit Breaker Pattern

**Pattern:** Circuit Breaker
**Where:** `FaultManager` → `CircuitBreaker` (per agent).

Each agent has an associated CircuitBreaker with three states:

1. **CLOSED** (normal): requests pass through.
2. **OPEN** (tripped): agent is quarantined; all SubTask dispatch to this agent is blocked; the Orchestrator reassigns.
3. **HALF-OPEN** (recovery probe): one SubTask is allowed through; if it succeeds, the circuit closes again.

The circuit trips when:
- Consecutive heartbeat misses exceed the configured threshold.
- The proportion of rejected Evaluations for this agent exceeds the failure rate threshold.

**Why this matters:** Without a circuit breaker, a hallucinating agent can consume SubTask retries indefinitely and corrupt dependent tasks. This pattern directly implements the fault tolerance requirement.

---

### 2.3 Repository Pattern

**Pattern:** Repository
**Where:** `TaskRepository`, `SubTaskRepository`, `ExecutionRepository`, `AgentRepository`, `AgentMessageRepository`, `EvaluationRepository`, `ConsensusRepository`.

All database access is abstracted behind repositories. Each exposes methods such as `create`, `get_by_id`, `update_status`, `list_by_*`. SQLAlchemy queries are hidden from the service layer.

Benefits:
- Services depend on repository interfaces, not on SQLAlchemy directly.
- Tests can mock repositories, enabling unit testing of OrchestratorService, EvaluatorService, etc. without a live database.
- Query optimization (indexes, eager loading) can be applied in the repository without touching service logic.

---

### 2.4 Proxy Pattern

**Pattern:** Proxy (Remote Proxy)
**Where:** `AgentProxy`

From the Orchestrator's perspective, `AgentProxy` is the local representation of a remote LLM agent. It:
- Hides the underlying transport (Message Bus dispatch or direct HTTP call).
- Enforces timeout policies.
- Handles retry at the dispatch level before escalating to the FaultManager.
- Translates transport-level errors into domain-level `Execution` failure records.

The Orchestrator interacts with `AgentProxy` using the same interface regardless of whether the agent is local, containerized, or remote — consistent with the Proxy pattern's intent.

---

### 2.5 Data Transfer Object (DTO) / Schema Pattern

**Pattern:** Data Transfer Object (DTO)
**Where:** Pydantic schemas: `TaskCreate`, `TaskRead`, `SubTaskRead`, `ExecutionRead`, `EvaluationRead`, `ConsensusRead`.

Schemas serve as DTOs between the API layer and domain logic:
- They define what the API accepts and returns, independently of ORM models.
- Routers validate requests via `*Create` schemas and serialize responses via `*Read` schemas.
- Domain models are never exposed directly to external consumers.

This separation allows the API contract to evolve independently of the database schema.

---

### 2.6 Factory Pattern

**Pattern:** Factory Method / Abstract Factory
**Where:** `AgentRegistry` (agent instantiation), `LLMProviderAdapter` (provider selection).

`AgentRegistry.get_available(role, capability)` acts as a factory that selects and returns an appropriate `AgentProxy` for a given SubTask requirement. Internally, agents are constructed with the correct `LLMProviderAdapter` (OpenAI, Anthropic, local) based on configuration — without the Orchestrator needing to know which provider backs a given agent.

---

### 2.7 Chain of Responsibility (Evaluation Pipeline)

**Pattern:** Chain of Responsibility
**Where:** `EvaluatorService` evaluation pipeline.

Each `HallucinationDetector` strategy in the evaluation pipeline processes the Execution output and either:
- Passes it to the next strategy in the chain (if its own check passes), or
- Raises a rejection verdict immediately (short-circuit on definitive hallucination signal).

The final verdict aggregates scores from all detectors that ran. This allows composing lightweight checks (fast) before expensive checks (slow cross-reference with external knowledge).

---

## 3. Cross-Cutting Patterns and Practices

### 3.1 Dependency Injection

**Pattern:** Dependency Injection
**Where:** FastAPI `Depends` system; service constructors.

Services, repositories, and the current agent context are injected into route handlers and service constructors. This allows:
- Test overrides (mock repositories, mock LLM adapters).
- Swapping consensus or decomposition strategies via configuration without code changes.

---

### 3.2 Centralized Configuration

**Pattern:** Configuration Object (Singleton-like)
**Where:** `core/config.py`.

All environment-specific settings (database URL, LLM API keys, circuit breaker thresholds, retry limits, consensus strategy selection) are managed in a single Pydantic Settings object. This ensures consistent configuration across all services and simplifies deployment parameterization for research experiments.

---

### 3.3 Audit Log / Append-Only Write Pattern

**Pattern:** Append-Only Event Log
**Where:** `AgentMessage`, `Execution`, `EvaluationResult`, `ConsensusRecord` tables.

All agent messages, executions, evaluations, and consensus decisions are written as immutable records (never updated in place; `status` updates create new records or update a status field only). This:
- Preserves the full history of every agent interaction.
- Enables post-hoc replay and analysis — essential for answering the research question: *do distributed agents reduce hallucinations?*
- Supports dispute resolution if an agent's output is later contested.

---

## 4. Relationship to Other Design Documents

This document is intended to be read alongside:

- `system-architecture.md` — high-level C4 / 4+1 architecture.
- `class-diagrams.md` — service, repository, and model class structure.
- `data-model/conceptual-erd.md` — conceptual domain relationships.
- `data-model/logical-erd.md` and `data-model/physical-erd.md` — logical and physical database models.

Together, these documents show:

1. **What** the system does (research goals and domain).
2. **How** it is structured (architecture and class diagrams).
3. **Which** design patterns were chosen to achieve reliability, fault tolerance, and research reproducibility.
