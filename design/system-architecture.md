# System Architecture

This document describes the architecture of the Distributed LLM Agents system across two complementary dimensions:
1. **C4-style structural views (System Context and Container views)**, providing the high-level system decomposition.
2. **4+1 view model** of software architecture which comprises:
- A `logical view`: shows key abstractions in the system as objects or object classes.
- A `process view`: shows how, at run-time, the system is composed of interacting processes.
- A `development view`: shows how the software is decomposed for development.
- A `physical view`: shows the system hardware and how software components are distributed across the processors in the system.

Component-level details are documented separately in `~/design/class-diagrams.md` and `~/design/data-model/`.

---

## 1) High-Level Architecture (C4 Level-2: Container View)

The high-level structure of the system is represented using a C4 Level-2 Container Diagram (`~/diagrams/system-architecture.mmd`). It shows the major runtime containers, their responsibilities, and the communication paths between them.

The main containers are as follows:

- **Client Interface**: External users or systems that submit tasks and receive final aggregated results via HTTPS/REST.
- **API Gateway**: Single entry point for all client requests; performs authentication, rate-limiting, and routes to the Orchestrator.
- **Orchestrator Service**: The central coordinator. Receives tasks, invokes the Task Decomposer to produce a Directed Acyclic Graph (DAG) of sub-tasks, schedules sub-tasks onto available agents, and collects results.
- **Agent Pool**: A collection of heterogeneous LLM Agent instances, each assigned a role (Planner, Executor, Critic, Synthesizer). Agents communicate via the Message Bus and return execution results.
- **Message Bus**: Asynchronous publish/subscribe transport for inter-agent communication (task dispatching, result reporting, heartbeats, error signals).
- **Evaluator Service**: Receives each agent execution result and scores it for hallucination likelihood and internal consistency. Feeds verdicts back to the Orchestrator and Fault Manager.
- **Consensus Service**: When multiple agents produce conflicting or redundant outputs for the same sub-task, applies a configurable aggregation strategy (majority vote, weighted average, critic review) to produce a single authoritative result.
- **Fault Manager**: Monitors agent health via heartbeats; applies circuit-breaker logic to isolate failing or repeatedly hallucinating agents; triggers retries or reassignments.
- **Data Store**: PostgreSQL for persistent storage of tasks, sub-tasks, executions, messages, evaluation results, and consensus records.
- **Experiment Tracker**: Stores aggregate reliability metrics and evaluation logs for offline research analysis.

---

## 2) 4+1 Views

### Logical View (Domain and Services)
- **Core entities**: Agent, Task, SubTask, Execution, AgentMessage, EvaluationResult, ConsensusRecord.
- **Core services**: OrchestratorService, TaskDecomposer, AgentRegistry, EvaluatorService, ConsensusService, FaultManager.
- **External services**: LLM Providers (e.g. OpenAI, Anthropic, local inference endpoints).

### Process View (Runtime and Concurrency)
- **API Gateway** receives a Task submission and forwards it to the **Orchestrator**.
- **Orchestrator** calls **TaskDecomposer** to produce a DAG of SubTasks, then publishes each SubTask to the **Message Bus** as soon as its dependencies are satisfied.
- **Agents** subscribe to the Message Bus, claim SubTasks matching their role/capability, run the LLM call, and publish an Execution result.
- **Evaluator Service** listens for completed Executions, scores them, and emits a verdict. If rejected, the Fault Manager decides whether to retry on the same agent or reassign.
- **Consensus Service** is invoked when multiple Executions exist for the same SubTask, producing a ConsensusRecord before the Orchestrator proceeds to dependent SubTasks.
- **Fault Manager** monitors heartbeat signals; a missing heartbeat or a threshold of rejected Evaluations triggers circuit-breaker isolation of the agent.
- **Experiment Tracker** receives events from Evaluator and Consensus services asynchronously for offline analysis.

### Development View
- `orchestrator/`: task intake, DAG construction, scheduling logic.
- `agents/`: agent roles (planner, executor, critic, synthesizer), LLM provider adapters.
- `evaluator/`: hallucination detection strategies, consistency scoring.
- `consensus/`: aggregation strategies (majority vote, weighted average, critic review).
- `messaging/`: message bus adapter, topic definitions, routing.
- `fault/`: circuit breaker, retry policies, agent health monitoring.
- `persistence/`: repositories, SQLAlchemy ORM models.
- `api/`: REST endpoints, request/response schemas (DTOs).
- `diagrams/`: Mermaid source files (system architecture, class diagram).

### Physical View (Deployment)
- **Cloud**: API Gateway and Orchestrator deployed as stateless services in a VPC; horizontally scalable.
- **Agent Pool**: Individual agent containers, scalable independently; each makes outbound calls to LLM Provider APIs.
- **Message Broker**: Redis Streams or RabbitMQ deployed as a managed service; provides durable, ordered message delivery.
- **PostgreSQL**: Managed database instance (e.g. RDS); primary data store for all entities.
- **Experiment Tracker**: Lightweight service writing to a time-series store or columnar database for analytics.
- **Network**: Internal service-to-service calls over mTLS; LLM Provider calls over HTTPS with API key authentication.

### Scenarios (+1)
- **Task execution flow**: Submit → Decompose → Dispatch SubTasks → Agent Executes → Evaluate → Consensus → Return Result.
- **Fault recovery flow**: Agent fails / hallucination detected → Evaluator rejects → Fault Manager isolates agent → Orchestrator reassigns SubTask → Retry Execution.
- **Collective evaluation flow**: Multiple agents execute same SubTask → Consensus Service aggregates → confidence score recorded → Research metrics updated.

---

## 3) Architectural Style and Patterns

- **Layered**: API ↔ Application (Orchestrator/Evaluator/Consensus) ↔ Agents ↔ Data; each layer has well-defined responsibilities and interfaces.
- **Event-Driven / Message-Passing**: Agents communicate asynchronously through the Message Bus; no direct agent-to-agent coupling.
- **Orchestrator Pattern**: A central Orchestrator coordinates the entire task lifecycle, avoiding the complexity of a fully decentralized choreography while still enabling parallelism.
- **Strategy Pattern**: Task decomposition strategies, consensus aggregation strategies, and hallucination detection strategies are pluggable.
- **Circuit Breaker**: Fault Manager applies a circuit-breaker to protect the system from cascading failures caused by misbehaving or hallucinating agents.
- **Repository & DAO**: Persistence boundaries are abstracted behind repositories, enabling clean testing and separation from business logic.

---

## 4) Reliability and Fault Tolerance Considerations

- **Hallucination detection**: Every Execution is scored by the Evaluator before being accepted; rejected outputs trigger a retry pipeline.
- **Agent redundancy**: Critical SubTasks can be dispatched to multiple agents simultaneously; Consensus Service resolves conflicts.
- **Circuit breaker thresholds**: Configurable per agent role; once tripped, the agent is quarantined and a notification is raised.
- **Retry limits**: Each SubTask tracks `attempt_number`; exceeding the retry limit marks the SubTask as `failed` and escalates to the Orchestrator.
- **Dependency isolation**: The DAG structure ensures that a failure in one SubTask branch does not corrupt independent branches.
- **Audit trail**: All messages, executions, evaluations, and consensus decisions are persisted for post-hoc analysis and reproducibility.

---

## 5) Trade-offs and Risks

- **Orchestrator as bottleneck**: Centralizing coordination simplifies reasoning about state but introduces a single point of contention; mitigated by stateless design and horizontal scaling.
- **LLM latency variability**: Agents depend on external LLM APIs with non-deterministic latency; timeout policies and async dispatch mitigate blocking.
- **Evaluation accuracy**: Automated hallucination detection is imperfect; false positives waste compute, false negatives corrupt results; human-in-the-loop review is planned for the critic role.
- **Consensus overhead**: Running multiple agents on the same SubTask increases reliability but multiplies LLM API cost; applied selectively for high-stakes sub-tasks.
- **Experiment reproducibility**: LLM outputs are stochastic; full context and seed logging are required to reproduce research results.

---

## 6. Design Artifacts in This Repository

The following design artifacts complement the system architecture by detailing specific implementation views:

- `design/data-model/conceptual-erd.md` — Core domain entities and their relationships at the conceptual level.
- `design/data-model/logical-erd.md` — Logical data model with attributes, primary keys, and foreign keys.
- `design/data-model/physical-erd.md` — PostgreSQL-specific physical schema with types, constraints, and indexes.
- `design/class-diagrams.md` — Service, repository, and domain model class structure.
- `design/design-patterns.md` — Documented design pattern choices, rationale, and trade-offs.
