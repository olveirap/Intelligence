# Intelligence Mesh Framework: Technical Specification & Operational Protocols

**Date:** April 10, 2026
**Status:** Canonical Spec v2.3
**Author:** Paulo Olveira (Synthesis)

---

## Purpose & Scope

The Intelligence Mesh is a personal operating system for autonomous backlog management. Its domains include personal projects, applied research, academic learning, and financial opportunity tracking.

It does not replace domain-specific tooling — repositories, kanbans, investment trackers — but operates as the canonical orchestration layer above them. External workflows are treated as **Delegated Execution Environments**: the Mesh owns intent and terminal state; internal execution is opaque to the Bus.

The system represents personal intelligence offloaded into autonomous execution. Over time, the learning mechanism (Section 4.5) exists specifically to reduce the frequency of $P_H$ intervention as the system accumulates validated rules — not only as a safety feature.

---

## 1. Core Axioms & Theoretical Basis

### 1.1 Entropy Reduction
The system is an Inference-Driven Synthesis engine. It processes raw information through a series of sinks until it becomes high-density, actionable logic.

* **L0 (Observation):** Unstructured, high-entropy input — manual notes, raw data feeds.
* **L1 (Orientation):** Machine-synthesized structure — links, summaries, proposals.
* **L2 (Decision):** Human-curated Ground Truth — pruned, annotated, and finalized state.

### 1.2 State-Compute Decoupling
The Mesh prioritizes Consistency and Partition Tolerance (CAP Theorem) by anchoring Global State on a 24/7 persistent node, while offloading high-compute inference to transient, high-capability nodes.

---

## 2. Atomic Entities: Tasks, Capabilities, and Provenance

### 2.1 The Task ($T$)
A discrete unit of work defined as $T = \{Input, Objective, C_{req}, \text{durability\_class}\}$.

* **Durability Class:** Categorized as `ephemeral` or `durable`. Default state is **durable**.
* **Schema Enforcement:** Task creation without a specified class is treated as a validation failure at the Bus entry point, requiring HITL classification before the task is queued.

### 2.2 Task Hierarchy
The Mesh operates two tiers of tasks:

* **Mesh Tasks:** Top-level intent, owned by the Mesh, resolved via State Sink Acknowledgment. These are the canonical unit of work.
* **Delegated Tasks:** Sub-tasks offloaded to an external workflow — a GitHub issue, a kanban card, a backlog entry. The Mesh tracks their existence but not their internal execution. Only their terminal state propagates back to the Bus as an Acknowledgment.

This distinction allows project-specific tooling to manage its own execution while the Mesh retains ownership of intent.

### 2.3 The Capability ($C$)
A resource provided by a node, evaluated at the moment of consumption.

* **Computational:** GPU/CPU resources (e.g., `GPU_INFERENCE`).
* **Permissioned:** Access-based (e.g., `FS_WRITE`, `NET_ACCESS`).
* **Human ($P_H$):** Specialized capability reserved for manual decision-making and curation.

### 2.4 Content Provenance
Data on the Bus is tagged with its origin to dictate trust and processing paths:

* **Human ($P_H$):** Ground truth. Defines goals and resolves conflicts.
* **External ($P_E$):** High-entropy raw data from APIs or external feeds.
* **Synthetic ($P_S$):** Machine-generated content. Subject to $P_H$ validation and security gating before it reaches the State Sink or triggers a Delegated Task.

---

## 3. Infrastructure Topology

* **Persistent Node (The Bus):** Always-on. Hosts the message broker, manages task queues, and serves as the master context repository.
* **Compute Node (The Muscle):** High-resource hardware. Provides heavy-duty $P_S$ capabilities such as LLM inference and code execution.
* **Interface Node (The Sensory):** Portable hardware. Provides the $P_H$ capability for interaction and conflict resolution.

---

## 4. Operational Annex: Distributed Systems Constraints

### 4.1 Delivery Semantics: Bus-Side Exactly-Once

* **Bus Contract:** The Bus guarantees exactly-once delivery via persistent offsets and idempotency keys.
* **Consumer Contract (Sink-Gated Completion):** A task is considered executed **only** when its output is committed as a **State Sink Acknowledgment Record**. The output need not be the full artifact — a record confirming what was done (e.g., "PR opened", "backlog item created", "rule appended") is sufficient. The artifact itself may live in an external Delegated Execution Environment.
* **Atomic Commitment:** Consumers must ensure external side effects are idempotent. If a node fails before the Sink Acknowledgment, the Bus re-releases the task; the consumer must recognize the re-delivery and skip external side effects before re-attempting the Acknowledgment.

### 4.2 Ordering and Parallelism: Multi-Lane DAG

* **Tag-Based Lanes:** Each domain operates as an independent logical lane, identified by a tag created either statically at system configuration or dynamically at runtime.
* **Intra-Lane Ordering:** Tasks follow a Partial-Order DAG. Dependencies are strictly enforced. A failure in a mid-graph task triggers a **DAG Rollback Deferral**, flagged for manual review if re-execution logic fails to converge.
* **Parallel Execution:** Lanes are processed in parallel. A block in one lane has no impact on the throughput of others.

### 4.3 Dispatch Mechanism: Pull-Based with Long-Polling

* **Ephemeral Claiming:** Nodes self-evaluate their current capability state (e.g., VRAM availability) at the moment of the pull.
* **Task Ownership:** Once a task is pulled, it is hidden from other consumers for a visibility timeout period. The node is the final authority on whether it can fulfill the task's $C_{req}$ at the moment of consumption.

### 4.4 Dynamic Tag Creation & Lane Provenance

Tag Domains are not statically defined. A $P_S$ node may create a new tag mid-execution, instantiating a new logical lane.

* **Trust Inheritance:** A dynamically created lane inherits the trust level and blocking state of its founding task until $P_H$ validates the tag.
* **Contested State:** If the founding task is later invalidated by $P_H$, all tasks queued in the inherited lane are flagged as `Contested` and require human verification before execution resumes.
* **Multi-Tag Ownership:** Tasks belonging to multiple tags are owned by the highest-priority lane for execution. Results are broadcast to all relevant tag states simultaneously.

### 4.5 Learning Loop Termination: Depth-2 Recursion

Recursion is capped to prevent synthetic drift:

1. **Level 1 (Domain):** Learning from the $P_S / P_H$ delta in the subject matter.
2. **Level 2 (Meta):** Learning from the synthesis process itself — how rules are formed and conflicts are identified.

Any task generated by an L2 output is terminal and exempt from further recursive learning analysis. This cap is also the primary mechanism by which $P_H$ intervention frequency decreases over time: validated L1 and L2 rules reduce the surface area of future conflicts.

### 4.6 Synthetic Output Security Gate

Content provenance (Section 2.4) tracks the origin of data but does not validate its content. Before any $P_S$ output reaches the State Sink or triggers a Delegated Task in an external system, it must pass through a **Security Gate**.

* **Scope:** All $P_S$ outputs that exercise a permissioned capability (`FS_WRITE`, `NET_ACCESS`, code commits, external API calls) are subject to gating. Read-only or purely internal $P_S$ outputs (e.g., summaries routed to the State Sink without side effects) may be gated at a lower severity level.
* **Gate Mechanism:** The gate evaluates outputs against a set of human-authored rules (`$P_H$-provenance`). These rules define what a $P_S$ node is permitted to produce for a given capability and task type. Rules are themselves versioned entries in the State Sink.
* **Failure Behavior:** A $P_S$ output that fails the gate is blocked, logged as an auditable event, and escalated to a $P_H$ review task. It does not re-enter the execution queue automatically.
* **Relation to Learning Loop:** When $P_H$ resolves a gate failure, the Bus generates an L1 Learning Task from the delta, refining the gate rules. This connects the security gate directly to the Depth-2 recursion mechanism.

### 4.7 Backpressure and Staleness Policy: 24-Hour TTL

Tasks in the queue are subject to a **24-hour TTL**. Upon expiry, the Bus evaluates the `durability_class`:

* **Ephemeral:** Silently discarded. The knowledge is assumed to be re-derivable by future $P_S$ activity.
* **Durable:** Never discarded on TTL. Escalated to a $P_H$ notification task for manual re-prioritization.

### 4.8 State Sink Consistency: Commutativity-Gated Merge

* **Commutative Operations (Auto-Merge):** Additive operations (e.g., adding a block to an existing entry) merge automatically and are logged as auditable $P_S$ events for deferred review.
* **Non-Commutative Operations (Escalate):** Destructive or mutative operations — deletions, replacements, property renames — are inherently non-commutative ($A \rightarrow B \neq B \rightarrow A$) and escalate immediately to a $P_H$ Conflict Resolution Task.

### 4.9 Refactoring Lane

$P_S$ output accumulates in the State Sink over time. Without a maintenance pass, duplicated logic, stale rules, and structural inconsistencies degrade knowledge quality silently. The Refactoring Lane is a dedicated periodic task class that addresses this.

* **Task Class:** `refactor`. Durability class is always `ephemeral`. Priority is lower than all standard execution lanes.
* **Scope:** Refactoring tasks are scoped exclusively to $P_S$-authored content in the State Sink. $P_H$-authored content is never modified by a refactoring task.
* **Trigger:** Refactoring tasks are generated on a scheduled basis by the Bus, or when the volume of auditable auto-merge events in a Tag Domain exceeds a defined threshold.
* **Output:** Proposed structural changes (deduplication, rule consolidation, dead link removal) are treated as non-commutative operations and escalate to $P_H$ review before being committed, per Section 4.8.
* **Relation to Security Gate:** Refactoring task outputs pass through the Security Gate (Section 4.6) before reaching the State Sink, since they exercise `FS_WRITE` capability on existing knowledge nodes.

### 4.10 $P_H$ Latency: Lane-Scoped Indefinite Blocking

$P_H$ is a high-authority, high-latency node. Tasks requiring human resolution block their specific Tag Lane indefinitely. The rest of the Mesh continues to function. This ensures $P_H$ is never rushed by system-wide performance pressure, and that unrelated domains are never stalled by a single unresolved conflict.

---

## Summary of Design Trade-offs

| Axis | Choice | Rationale |
| :--- | :--- | :--- |
| **Durability** | Bus-Side Exactly-Once | Guarantees delivery; delegates side-effect safety to consumer idempotency. |
| **Throughput** | Multi-Lane DAG | Prevents domain-specific $P_H$ stalls from causing system-wide deadlocks. |
| **Coupling** | Pull-Based Dispatch | Nodes self-evaluate capabilities at poll time via long-polling. |
| **Stability** | Depth-2 Recursion | Limits feedback loops to Domain (L1) and Meta-Learning (L2). |
| **Consistency** | Commutativity-Gated | Additive data auto-merges; destructive edits require $P_H$. |
| **$P_H$ Latency** | Lane-Scoped Blocking | Prioritizes precision; stalls only the affected Tag Domain. |
| **Completion** | Sink-Gated Acknowledgment | Defines "Done" as a committed record in the State Sink. |
| **Staleness** | 24hr TTL + Class Policy | Discards ephemeral logic rot; escalates durable items to $P_H$. |
| **Tag Provenance** | Dynamic Lane Inheritance | New lanes inherit founding task trust until $P_H$ validation. |
| **Security** | $P_H$-Rule Security Gate | Validates $P_S$ outputs before side effects reach external systems. |
| **Maintenance** | Refactoring Lane | Periodic ephemeral pass prevents $P_S$ knowledge rot in the State Sink. |
| **Rollback** | DAG Rollback Deferral | **Open Requirement:** Identifies failure in dependency chains for manual recovery. |
