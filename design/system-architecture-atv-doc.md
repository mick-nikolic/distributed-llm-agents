# System Architecture: Adaptive Tiered Validation (ATV)

This document describes the system architecture for the Adaptive Tiered Validation framework. The corresponding Mermaid diagram is at `diagrams/system-architecture-atv.mmd`.

---

## Overview

The ATV architecture adds **two independent defense mechanisms** to a standard distributed MAS:

1. **Validation Pipeline on the Message Bus** (Steps 1–4) — intercepts and validates every inter-agent message before delivery.
2. **Conditional Compliance Module (CCM)** — embedded inside each agent, validates the agent's own decisions before execution.

These mechanisms are complementary: the pipeline catches bad messages in transit, while the CCM catches bad decisions made by a compromised agent even when the incoming message was valid.

---

## Architecture Layers

### 1. Client Layer

| Component | Description |
|-----------|-------------|
| **External User / System** | Submits tasks via REST API, polls status, retrieves results. |

### 2. API Gateway

| Component | Description |
|-----------|-------------|
| **REST API Gateway** | Authenticates requests, rate-limits, routes to Orchestration. On task submission, **freezes the original task specification** as an immutable copy in Object Storage. |

### 3. Orchestration Layer

| Component | Description |
|-----------|-------------|
| **Orchestrator Service** | Coordinates decomposition, scheduling, task lifecycle, and result aggregation. |
| **Task Decomposer** | Decomposes top-level tasks into a sub-task DAG. |
| **Agent Registry** | Maintains agent capabilities, health status, and **historical reliability scores**. Feeds the Severity Classifier with per-agent reliability features. |

### 4. Agent Pool

Each agent embeds a **Conditional Compliance Module (CCM)** as internal middleware.

| Component | Description |
|-----------|-------------|
| **Planner / Executor / Critic / Synthesizer Agent** | Standard MAS agents with specialized roles. |
| **CCM (per agent)** | Before the agent commits to any action, CCM reads the **frozen original system prompt** and **frozen task specification** from Object Storage and compares them against the agent's intended action. If contradiction is detected (e.g., corrupted prompt says "trust everything" but original says "verify inputs"), CCM **pauses the agent** and escalates to the Fault Manager. This is **Mechanism 2**, completely separate from the Validation Pipeline. |

### 5. Message Bus

| Component | Description |
|-----------|-------------|
| **Pub/Sub Broker** | Asynchronous message passing between all components. |
| **Validation Interceptor** | Middleware hook that captures **every inter-agent message** before delivery and routes it into the Validation Pipeline. Non-invasive — agents are unaware of the interception. This is the entry point for **Mechanism 1**. |

### 6. Validation Layer — Pipeline Steps 1 to 4 (Mechanism 1)

Every intercepted message flows through these steps sequentially. Each step either clears the message or escalates to the next.

| Component | What it does | Cost | When it activates |
|-----------|-------------|------|-------------------|
| **Step 1: Rule Gate** | Deterministic structural checks: JSON schema validation, message deduplication (hash comparison), cycle detection, context length enforcement, role-instruction consistency. Produces a **numeric feature vector** as a side-product. | **Zero** (no LLM) | Always — every message. |
| **Step 2: Severity Classifier** | Lightweight ML model (XGBoost) that takes the feature vector from Step 1 plus contextual features (embedding similarity to frozen task spec, agent reliability score from Registry). Outputs a severity label: **Low**, **Medium**, or **High**. This is a router, not a validator — it only decides where the message goes next. | **Negligible** (no LLM) | Always — every message that passes Step 1. |
| **Step 3: Primary Validator** | One LLM call. Reads the message content and the frozen task specification from Object Storage. Performs semantic consistency check: detects hallucination markers, logical contradictions, unsupported claims. Returns: Accept / Reject / Escalate. | **1 LLM call** | Only when Step 2 outputs **Medium**, or on random spot-checks. |
| **Step 4: Quorum Validator** | Three independent LLM validators using **heterogeneous models** (e.g., GPT, Claude, open-source). Each independently evaluates the message. Majority vote (2/3) determines verdict. | **3 LLM calls** | Only when Step 3 escalates, or when Step 2 outputs **High**. |

**Routing logic summary:**

```
Message → Step 1 (Rule Gate) → Step 2 (Classifier)
  ├── Low    → deliver to recipient (zero LLM cost)
  ├── Medium → Step 3 (Primary Validator)
  │     ├── Accept  → deliver
  │     ├── Reject  → Fault Manager
  │     └── Escalate → Step 4 (Quorum)
  └── High   → Step 4 (Quorum) directly
        ├── 2/3 Accept → deliver
        └── 2/3 Reject → Fault Manager
```

### 7. Reliability Layer

| Component | Description |
|-----------|-------------|
| **Evaluator Service** | Post-execution quality assessment. Reads stored artifacts, applies hallucination detection. Works alongside (not instead of) the Validation Pipeline. |
| **Consensus Service** | Reconciles multiple execution results via configurable strategies (majority vote, weighted average, critic review). |
| **Fault Manager** | Receives signals from: Validation Pipeline rejections (Steps 3–4), CCM contradiction escalations, Evaluator rejections, and agent heartbeats. Actions: retry, reassign, quarantine (circuit breaker), or escalate to human review. **Updates agent reliability scores** in the Registry after each event. |

### 8. Data Plane

| Component | Description |
|-----------|-------------|
| **PostgreSQL** | Task state, agent registry, validation logs, circuit breaker state. |
| **Experiment Tracker** | Stores per-step validation metrics (activation counts, escalation rates, latency) for cost-efficiency analysis. |
| **Object Storage** | Stores **frozen task specifications** (written at submission, read-only), **frozen agent system prompts** (written at initialization, read-only), and prompt/response artifacts from every LLM call. The frozen copies are the immutable reference points for both the Validation Pipeline and the CCM. |

### 9. LLM Providers

| Component | Description |
|-----------|-------------|
| **OpenAI / Anthropic / Local Inference** | Used by agents for task execution AND by Steps 3–4 for validation. Using different providers for agents vs. validators enables heterogeneous quorum validation. |

---

## Key Data Flows

### Normal Flow (no faults)

1. User submits task → Gateway freezes task spec in Object Storage → Orchestrator decomposes
2. Sub-task published to Message Bus → **Interceptor captures** → Step 1 (structural OK) → Step 2 (Low) → **message delivered**
3. Agent receives message → **CCM checks** against frozen prompt/spec → no contradiction → agent executes
4. Agent publishes result → goes through Validation Pipeline again → delivered to next agent

### Fault Detected by Pipeline

1. Agent publishes output → Interceptor captures → Step 1 OK → Step 2 = **Medium**
2. Step 3 (Primary Validator) reads frozen task spec, compares → detects fabricated requirements → **Reject**
3. Fault Manager triggers retry on different agent, updates reliability score

### Fault Detected by CCM

1. Message passes all 4 pipeline steps (content was valid)
2. Agent processes message and decides on action
3. **CCM** compares intended action with frozen original system prompt → detects contradiction (corrupted prompt says "skip validation" but original says "always validate")
4. CCM **pauses agent** → escalates to Fault Manager → Fault Manager resolves

---

## Why Two Separate Mechanisms

| Aspect | Validation Pipeline | CCM |
|--------|-------------------|-----|
| **Location** | On the Message Bus, between agents | Inside each agent |
| **What it checks** | Messages in transit | Agent's own decisions before execution |
| **When it acts** | Before the message reaches the recipient | After the agent processes the message, before it acts |
| **What it catches** | Bad information being sent between agents | Bad decisions by a compromised agent, even when the incoming message was valid |
| **Primary threat** | Hallucinations, fabricated content, communication faults | Configuration faults (Blind Trust, Role Ambiguity) where the agent's own prompt is corrupted |

---

## Mapping to MAS-FIRE Fault Taxonomy

| Fault Category | Primary Defense | Secondary Defense |
|----------------|----------------|-------------------|
| Planning Faults (IP, CIL) | Step 3 (semantic check) | Step 4 (quorum) |
| Memory Faults (ML, CLV) | Step 1 (context length, dedup) | Step 3 (completeness) |
| Reasoning Faults (H) | Step 3 (hallucination detection) | Step 4 (cross-model) |
| Action Faults (PFE, TFE, TSE) | Step 1 (schema validation) | Step 3 (semantic review) |
| Configuration Faults (RA, BT) | **CCM** (contradiction detection) | Steps 3–4 as backup |
| Instruction Faults (ILC, IA) | Step 3 (conflict detection) | Step 4 + human escalation |
| Communication Faults (MC, MS, MBA) | Step 1 (cycle detection, dedup, routing) | — (deterministic) |
