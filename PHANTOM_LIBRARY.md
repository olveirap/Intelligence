# Intelligence Mesh: Phantom Library Proposal

**Status:** Implementation Proposal (v1.0-alpha)
**Date:** April 2026
**Target Runtimes:** Go 1.22+ (Bus), Python 3.12+ (Agents)
**Backbone:** Redis 7.2+

---

## 1. Core Data Schemas (JSON Schema)

The Mesh enforces strict schema validation at the ingestion layer of the Bus Daemon. All tasks and security rules must conform to these definitions.

### 1.1 Mesh Task (`task.schema.json`)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Mesh Task",
  "type": "object",
  "required": [
    "id", "objective", "input", "c_req", "durability_class", 
    "provenance", "tag_domains", "depth", "created_at"
  ],
  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "objective": { "type": "string" },
    "input": { "type": ["object", "string"] },
    "c_req": {
      "type": "array",
      "items": { "type": "string", "enum": ["GPU_INFERENCE", "FS_WRITE", "NET_ACCESS"] }
    },
    "durability_class": { "type": "string", "enum": ["ephemeral", "durable"] },
    "provenance": { "type": "string", "enum": ["P_H", "P_E", "P_S"] },
    "tag_domains": { "type": "array", "items": { "type": "string" } },
    "parent_task_id": { "type": ["string", "null"], "format": "uuid" },
    "depth": { "type": "integer", "maximum": 2 },
    "created_at": { "type": "string", "format": "date-time" },
    "ttl_expires_at": { "type": ["string", "null"], "format": "date-time" }
  }
}
```

### 1.2 Security Gate Rule (`gate_rule.schema.json`)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Security Gate Rule",
  "type": "object",
  "required": ["rule_id", "applies_to", "conditions", "on_failure", "version"],
  "properties": {
    "rule_id": { "type": "string" },
    "applies_to": {
      "type": "object",
      "properties": {
        "provenance": { "type": "string", "enum": ["P_S", "*"] },
        "capability": { "type": "string" },
        "tag_domains": { "type": "array", "items": { "type": "string" } }
      }
    },
    "conditions": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "field": { "type": "string" },
          "operator": { "type": "string", "enum": ["not_contains", "matches", "gt"] },
          "value": { "type": ["array", "string", "number"] }
        }
      }
    },
    "on_failure": { "type": "string", "enum": ["block", "escalate_ph"] },
    "version": { "type": "integer" }
  }
}
```

---

## 2. Bus Daemon (Go Interfaces)

The Go-based Bus Daemon is responsible for stream management, DAG resolution, and security gating.

### 2.1 Message Bus & Stream Management

```go
package mesh

import "context"

// BusDaemon defines the primary orchestration interface.
type BusDaemon interface {
    // Ingest validates a task against JSON Schema and adds it to the appropriate Redis stream.
    Ingest(ctx context.Context, task Task) error

    // Dispatch handles the pull-based claiming logic for nodes.
    Dispatch(ctx context.Context, capability Capability) (Task, error)

    // Acknowledge commits task completion and triggers DAG resolution.
    Acknowledge(ctx context.Context, taskID string, ack SinkAcknowledgment) error
}

// StreamManager handles low-level Redis Stream operations.
type StreamManager interface {
    // Claim blocking-reads from a consumer group lane.
    Claim(group, consumer string) (Message, error)
    
    // DLQ moves a message to the dead-letter stream after delivery exhaustion.
    DLQ(ctx context.Context, msg Message) error
    
    // DiskSpill serializes low-priority tasks to disk when stream pressure is high.
    DiskSpill(ctx context.Context, threshold int) error
}
```

### 2.2 DAG & Dependency Tracker

```go
// DAGManager manages the intra-lane task ordering.
type DAGManager interface {
    // ResolveAtomically updates downstream dependencies using Lua scripts.
    ResolveAtomically(ctx context.Context, completedTaskID string) ([]string, error)

    // DeferRollback flags downstream tasks for manual PH review on mid-graph failure.
    DeferRollback(ctx context.Context, failedTaskID string) error

    // SetContested flags a lane as requiring validation due to invalid lane-founder.
    SetContested(ctx context.Context, tagDomain string) error
}
```

### 2.3 Security Gate

```go
// SecurityGate evaluates Synthetic (P_S) output before Sink commitment.
type SecurityGate interface {
    // Evaluate checks a task output against PH-authored rule sets.
    Evaluate(ctx context.Context, output PSOutput) (GateVerdict, error)

    // HotReload refreshes rules from Git-versioned JSON files.
    HotReload() error
}

```go
type GateVerdict struct {
    Passed  bool
    RuleID  string
    Message string
}
```

---

## 3. State Sink (Go Interfaces)

The State Sink handles the canonical persistence of knowledge into Logseq markdown files, anchored by Git for auditability.

### 3.1 Sink Writer & Commutativity Gate

```go
// SinkWriter is the single-authority for Logseq + Git mutations.
type SinkWriter interface {
    // Commit authoring a structured Git commit for a Sink mutation.
    Commit(ctx context.Context, mutation SinkMutation) error

    // BlockedWrite logs a non-commutative operation for PH resolution.
    BlockForPH(ctx context.Context, mutation SinkMutation) error
}

// CommutativityGate enforces Section 4.8 of the Spec.
type CommutativityGate interface {
    // Check evaluates if a write is additive (auto-merge) or destructive (escalate).
    Check(mutation SinkMutation) OperationClass
}

type OperationClass string

const (
    OpAdditive    OperationClass = "additive"
    OpDestructive OperationClass = "destructive"
)
```

### 3.2 Knowledge Navigation (Index Maintenance)

```go
// IndexGenerator maintains the semantic traversal points for PS agents.
type IndexGenerator interface {
    // Regenerate scheduled or event-driven Index Page maintenance.
    Regenerate(ctx context.Context, tagDomain string) error
}
```

---

## 4. Node Agent Runtime (Python Protocols)

The Compute Node agents are primarily Python-based, wrapping inference and tool execution in a Bus-aware harness.

### 4.1 Compute Node Agent

```python
from typing import Protocol, List, Optional
from dataclasses import dataclass

class NodeAgent(Protocol):
    """Protocol for capability-declaring Python workers."""
    
    capabilities: List[str]  # e.g., ["GPU_INFERENCE", "FS_WRITE"]

    async def poll_and_claim(self) -> Optional['Task']:
        """Blocking long-poll to Redis Streams via XREADGROUP."""
        ...

    async def execute(self, task: 'Task') -> 'ExecutionResult':
        """The primary inference/tool-use loop."""
        ...

    async def acknowledge(self, task_id: str, result: 'ExecutionResult') -> None:
        """Writes Sink Acknowledgment and ACKs the Redis Stream."""
        ...
```

### 4.2 Refactoring Lane Scheduler (APScheduler)

```python
class RefactoringScheduler(Protocol):
    """Refactoring lane orchestrator running in the Python layer."""

    async def trigger_periodic(self, tag_domain: str) -> None:
        """Weekly scheduled refactor emission."""
        ...

    async def trigger_threshold_based(self, tag_domain: str) -> None:
        """Threshold-based emission (e.g., >50 commits since last pass)."""
        ...
```

---

## 5. $P_H$ Interface (Telegram Bot)

The existing Telegram Bot handles human intervention tasks.

```go
// HumanGate handles the PH resolution lifecycle.
type HumanGate interface {
    // Escalate sends an interactive notification to the Telegram Bot.
    Escalate(ctx context.Context, task ResolutionTask) error

    // HandleResolution processes the inline keyboard feedback from PH.
    HandleResolution(ctx context.Context, response ResolutionPayload) error
}

type ResolutionTask struct {
    Type    ResolutionType # e.g., GateFailure, Conflict, TTL_Escalation
    TaskID  string
    Context map[string]interface{}
}
```

---

## 6. Audit & Observability (Loki/Grafana)

Operational logs are structured JSON shipped to Loki.

| Metric | Source | Logic |
| :--- | :--- | :--- |
| `mesh_queue_depth` | Bus Daemon | XLEN per stream lane |
| `mesh_gate_failures` | Security Gate | Count of `Passed=false` verdicts |
| `mesh_ph_latency` | HumanGate | Time from `Escalate` to `HandleResolution` |
| `mesh_refactor_count` | Refactor Scheduler | Count of tasks with `tag_domains: [refactor]` |
