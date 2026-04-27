# UML Use Case Diagram: Adaptive Tiered Validation (ATV)

This document describes the UML Use Case Diagram for the ATV system. The corresponding Mermaid diagram is at `diagrams/use-case-diagram-atv.mmd`.

---

## Overview

The ATV system has **two independent defense mechanisms**, reflected as two distinct use case groups:

1. **Validation Pipeline** (on the Message Bus) — Steps 1–4 that intercept and validate inter-agent messages.
2. **Conditional Compliance Module (CCM)** (inside each agent) — validates the agent's own decisions before execution.

All other use case groups (Submission, Orchestration, Evaluation, Consensus, Fault Management, Administration, Research) are carried over from the baseline system, with extensions where noted.

| Relationship  | Notation                                    | Meaning |
|---------------|---------------------------------------------|---------|
| Association   | Solid line between actor and use case       | The actor participates in the use case |
| `<<include>>` | Dashed arrow from base → included use case  | The included behaviour is **always** executed |
| `<<extend>>`  | Dashed arrow from extension → base use case | The extension adds **optional/conditional** behaviour |

---

## Actors

| Actor | Type | Description |
|-------|------|-------------|
| **External User / System** | Primary | Submits tasks, polls status, retrieves results, responds to clarification requests. |
| **System Administrator** | Primary | Configures system parameters, circuit breakers, validation tier settings, classifier model updates. |
| **Research Analyst** | Primary | Queries metrics, exports reports, analyzes validation cost efficiency. |
| **Human Reviewer** | Secondary | Responds to escalation when automated consensus is insufficient. |
| **LLM Provider** | Secondary | Responds to agent execution calls AND to Step 3/Step 4 validation calls. |

---

## Use Cases by Group

### Task Submission & Query

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Submit Task** | External User | Baseline | Entry point for the orchestration pipeline. |
| **Authenticate Request** | — | Baseline | Validates caller identity. |
| **Freeze Task Specification** | — | **NEW** | Persists an immutable copy of the original user task spec in Object Storage at submission time. This frozen copy is the reference point for all downstream validation (both Pipeline and CCM). |
| **Monitor Task Status** | External User | Baseline | Polls current state of a submitted task. |
| **Retrieve Task Result** | External User | Baseline | Fetches the final aggregated answer. |
| **Configure Decomposition Strategy** | External User | Baseline | Optionally overrides the default decomposition algorithm. |

### Orchestration

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Decompose Task into SubTasks** | — | Baseline | Produces a sub-task DAG. |
| **Schedule SubTask** | — | Baseline | Determines execution order. |
| **Dispatch SubTask** | — | Baseline | Routes sub-tasks to agents via Message Bus. |

### Agent Execution & Conditional Compliance (Mechanism 2)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Execute SubTask** | — | Extended | Runs a sub-task. Now includes CCM check before action commitment. |
| **Call LLM API** | LLM Provider | Baseline | Inference request to external LLM endpoint. |
| **Store Prompt / Response Artifacts** | — | Baseline | Persists raw prompt and response for audit. |
| **Retry Execution** | — | Baseline | Re-runs a sub-task after failure. |
| **Check Conditional Compliance (CCM)** | — | **NEW** | Pre-execution checkpoint inside the agent. Reads the frozen original system prompt and frozen task spec from Object Storage. Compares them against the agent's intended action. If no contradiction → agent proceeds. If contradiction → pauses and escalates. |
| **Pause & Escalate Contradiction** | — | **NEW** | When CCM detects contradiction, pauses agent execution and escalates to Fault Manager for resolution. |
| **Request User Clarification** | External User | **NEW** | When Fault Manager cannot resolve the contradiction automatically, asks the user to clarify. |

### Validation Pipeline — Message Bus (Mechanism 1)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Intercept Inter-Agent Message** | — | **NEW** | Validation Interceptor captures every inter-agent message before delivery. Entry point. |
| **Step 1: Apply Rule Gate** | — | **NEW** | Deterministic structural checks: JSON schema, deduplication, cycle detection, context length. Zero LLM cost. Produces a numeric feature vector. |
| **Step 2: Classify Severity** | — | **NEW** | XGBoost classifier consumes feature vector + contextual features (embedding similarity, agent reliability). Outputs: Low / Medium / High. Routes the message accordingly. |
| **Step 3: Apply Primary Validation** | LLM Provider | **NEW** | One LLM call. Compares message content against frozen task spec. Detects hallucinations, logical contradictions. Returns: Accept / Reject / Escalate. Activates only on Medium severity or random spot-check. |
| **Step 4: Apply Quorum Validation** | LLM Provider | **NEW** | Three heterogeneous LLM validators, majority vote. Activates only on High severity or Step 3 escalation. |
| **Deliver Validated Message** | — | **NEW** | Clears the message and delivers it to the intended recipient. |
| **Reject & Remediate** | — | **NEW** | On rejection, signals Fault Manager for retry/reassignment/quarantine. |

### Evaluation (Baseline, unchanged)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Evaluate Execution** | — | Baseline | Scores execution result for hallucination and consistency. |
| **Read Artifacts from Object Storage** | — | Baseline | Retrieves stored prompts/responses for evaluation. |
| **Detect Hallucinations** | — | Baseline | Applies hallucination detection strategies. |
| **Trigger Fault Recovery** | — | Baseline | Signals Fault Manager when evaluation rejects. |

### Consensus (Baseline, unchanged)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Aggregate Consensus** | — | Baseline | Reconciles multiple execution results. |
| **Apply Consensus Strategy** | — | Baseline | Executes configured aggregation strategy. |
| **Escalate to Human Review** | Human Reviewer | Baseline | Involves human when automated consensus is insufficient. |

### Fault Management (Extended)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Monitor Agent Health** | System Administrator | Baseline | Tracks agent liveness and reliability. |
| **Track Heartbeats** | — | Baseline | Periodic heartbeat signals. |
| **Isolate Failing Agent** | — | Baseline | Trips circuit breaker. |
| **Process Validation Rejection** | — | **NEW** | Receives rejection signals from Pipeline Steps 3–4 and CCM escalations. Determines remediation action. |
| **Update Agent Reliability Score** | — | **NEW** | After each validation outcome (pass or fail), updates the agent's score in the Registry. Feeds back into the Severity Classifier. |

### Administration (Extended)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Configure System** | System Administrator | Baseline | Sets deployment-wide parameters. |
| **Set Circuit Breaker Thresholds** | — | Baseline | Defines heartbeat and failure-rate limits. |
| **Override Consensus Strategy** | — | Baseline | Replaces consensus strategy without redeployment. |
| **Configure Validation Tier Parameters** | System Administrator | **NEW** | Sets Step 3 spot-check sampling rate, Step 4 activation threshold, classifier confidence bounds, quorum size. |
| **Update Severity Classifier Model** | System Administrator | **NEW** | Deploys retrained classifier model to Object Storage. |

### Research & Analytics (Extended)

| Use Case | Actor | New? | Description |
|----------|-------|------|-------------|
| **Analyze Research Metrics** | Research Analyst | Baseline | Queries aggregate metrics. |
| **Query Experiment Tracker** | — | Baseline | Fetches raw logs. |
| **Export Metrics Report** | — | Baseline | Produces formatted report. |
| **Analyze Validation Cost Efficiency** | Research Analyst | **NEW** | Queries per-step activation counts, LLM call overhead, escalation rates. Computes cost-per-fault-prevented. Central to evaluating Hypothesis H1. |

---

## Include Relationships

| Base Use Case | Included Use Case | Rationale |
|---|---|---|
| Submit Task | Authenticate Request | All API entry points require authorisation. |
| Submit Task | Freeze Task Specification | Frozen spec is the reference for all validation. |
| Submit Task | Decompose Task into SubTasks | Decomposition is inseparable from submission. |
| Monitor Task Status | Authenticate Request | " |
| Retrieve Task Result | Authenticate Request | " |
| Decompose Task into SubTasks | Schedule SubTask | Scheduling immediately follows decomposition. |
| Schedule SubTask | Dispatch SubTask | Dispatch is the outcome of scheduling. |
| Dispatch SubTask | Execute SubTask | Dispatching always triggers execution. |
| Execute SubTask | Check Conditional Compliance (CCM) | Every execution must pass CCM before action commitment. |
| Execute SubTask | Call LLM API | Every execution requires an LLM call. |
| Execute SubTask | Store Prompt / Response Artifacts | Every prompt/response is persisted. |
| Execute SubTask | Evaluate Execution | Every execution is immediately evaluated. |
| Evaluate Execution | Read Artifacts from Object Storage | Evaluation always reads stored artifacts. |
| Evaluate Execution | Detect Hallucinations | Hallucination detection is the core evaluation step. |
| Retrieve Task Result | Aggregate Consensus | Results require consensus. |
| Aggregate Consensus | Apply Consensus Strategy | Strategy is the mechanism of aggregation. |
| Monitor Agent Health | Track Heartbeats | Health monitoring is built on heartbeats. |
| Configure System | Set Circuit Breaker Thresholds | Always part of system setup. |
| Configure System | Configure Validation Tier Parameters | Always part of system setup. |
| Analyze Research Metrics | Query Experiment Tracker | Analysis always queries the tracker. |
| Intercept Inter-Agent Message | Step 1: Apply Rule Gate | Every intercepted message passes through Step 1. |
| Step 1: Apply Rule Gate | Step 2: Classify Severity | Every Step 1 check produces a severity classification. |
| Process Validation Rejection | Update Agent Reliability Score | Every rejection updates the agent's reliability. |

---

## Extend Relationships

| Extending Use Case | Base Use Case | Condition / Trigger |
|---|---|---|
| Configure Decomposition Strategy | Submit Task | Only when submitter provides non-default strategy. |
| Retry Execution | Execute SubTask | Only on failure or rejection. |
| Trigger Fault Recovery | Evaluate Execution | Only when Evaluator rejects. |
| Escalate to Human Review | Aggregate Consensus | Only when confidence below threshold. |
| Isolate Failing Agent | Monitor Agent Health | Only when heartbeat/failure rate exceeds threshold. |
| Override Consensus Strategy | Configure System | Only on explicit administrator override. |
| Update Severity Classifier Model | Configure System | Only when administrator deploys retrained model. |
| Export Metrics Report | Analyze Research Metrics | Only when analyst requests export. |
| Analyze Validation Cost Efficiency | Analyze Research Metrics | Only when analyst requests cost analysis. |
| Pause & Escalate Contradiction | Check Conditional Compliance | Only when CCM detects contradiction. |
| Request User Clarification | Pause & Escalate Contradiction | Only when Fault Manager cannot resolve automatically. |
| Step 3: Apply Primary Validation | Step 2: Classify Severity | Only when severity = Medium, or on random spot-check. |
| Step 4: Apply Quorum Validation | Step 3: Apply Primary Validation | Only when Step 3 escalates or confidence is low. |
| Step 4: Apply Quorum Validation | Step 2: Classify Severity | Only when severity = High (skips Step 3). |
| Deliver Validated Message | Step 2: Classify Severity | When severity = Low (fast path, zero LLM cost). |
| Deliver Validated Message | Step 3: Apply Primary Validation | When Step 3 accepts. |
| Deliver Validated Message | Step 4: Apply Quorum Validation | When Step 4 quorum accepts. |
| Reject & Remediate | Step 3: Apply Primary Validation | When Step 3 rejects with high confidence. |
| Reject & Remediate | Step 4: Apply Quorum Validation | When Step 4 quorum rejects. |
| Process Validation Rejection | Reject & Remediate | Always triggered on validation rejection. |

---

## Relationship to Other Documents

| Document | Relationship |
|---|---|
| `system-architecture-atv.md` | Container/runtime view backing these use cases. |
| `system-architecture-atv.mmd` | Mermaid diagram of component communication. |
| `proposal.md` | Research hypotheses and scientific contribution. |
