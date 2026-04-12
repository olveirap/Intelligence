# Intelligence Mesh Framework

A personal operating system for autonomous backlog management and distributed decision-making.

## Overview

The **Intelligence Mesh** is an inference-driven synthesis engine designed to orchestrate work across multiple domains:

- **Personal Projects** — development, creation, and iteration
- **Applied Research** — experimental validation and learning
- **Academic Learning** — structured knowledge acquisition
- **Financial Opportunity Tracking** — investment and income analysis

Rather than replacing existing tools (repositories, kanbans, trackers), the Mesh operates as a **canonical orchestration layer** above them, treating external workflows as **Delegated Execution Environments**. The Mesh owns intent and terminal state; internal execution remains opaque to the Bus.

## Core Concept

The system represents **personal intelligence offloaded into autonomous execution** through three key mechanisms:

### 1. **Entropy Reduction**
Raw information is processed through a series of decision sinks:

- **L0 (Observation)** — Unstructured input (notes, raw data)
- **L1 (Orientation)** — Machine-synthesized structure (links, summaries)
- **L2 (Decision)** — Human-curated ground truth (pruned and finalized)

### 2. **State-Compute Decoupling**
The Mesh prioritizes consistency through:

- A **24/7 persistent Bus node** as the master context repository
- **High-compute nodes** for transient inference
- Clear separation between state management and execution

### 3. **Adaptive Learning**
A feedback loop continuously validates decision rules, reducing the need for human intervention over time while maintaining safety constraints.

## Theoretical Basis & Inspiration

The Intelligence Mesh is grounded in emerging research on decentralized agentic systems and high-throughput autonomous engineering.

### Related Papers
- **[Agentic Fog: A Policy-driven Framework for Distributed Intelligence in Fog Computing](https://arxiv.org/abs/2601.20764)** — Explores treating distributed nodes as policy-driven autonomous agents. It provides the mathematical basis for stability and convergence in asynchronous, decentralized environments like the Mesh.
- **[GEAKG: Generative Executable Algorithm Knowledge Graphs](https://arxiv.org/abs/2603.27922v1)** — Introduces representing procedural knowledge as executable graph structures. This informs the Mesh's approach to synthesizing actionable logic from high-entropy observation data.

### Inspiration
- **[Extreme Harness Engineering](https://www.latent.space/p/harness-eng)** — A shift in the engineering bottleneck from "writing code" to "building context." The Mesh adopts the philosophy of ultra-fast build loops and encoding "taste" into specs to drive autonomous behavior.
- **Ghost Library ([Symphony](https://github.com/openai/sy))** — A spec-driven approach to software distribution where coding agents reproduce systems from high-fidelity specifications. The Mesh’s orchestration layer is inspired by these patterns of spec-first development.

## Key Components

### The Bus (Persistent Node)
- Message broker and task orchestration
- Master repository of all state transitions
- Exactly-once delivery semantics via persistent offsets

### Compute Nodes
- **The Muscle** — High-resource hardware for LLM inference and code execution
- **The Sensory** — Portable hardware for human decision-making

### Atomic Units
- **Tasks ($T$)** — Discrete work units with defined objectives and capability requirements
- **Capabilities ($C$)** — Computational, permissioned, or human resources
- **Provenance Tags** — Human ($P_H$), External ($P_E$), or Synthetic ($P_S$) origins

## Architecture Highlights

- **Multi-Lane DAG** — Independent logical lanes for parallel execution with dependency ordering
- **Pull-Based Dispatch** — Nodes self-evaluate capability state at task consumption
- **Dynamic Tag Creation** — New execution lanes can be instantiated at runtime with trust inheritance
- **Sink-Gated Completion** — Tasks complete when state acknowledgments are committed, not when artifacts are produced

## Documentation

- **[SPEC.md](SPEC.md)** — Complete technical specification, theoretical basis, and operational protocols
- **LICENSE** — Apache License 2.0

## Author

Paulo Olveira

---

**Status:** Canonical Spec v2.3 (April 10, 2026)

For detailed system architecture, distributed systems constraints, and operational procedures, see [SPEC.md](SPEC.md).


Copyright 2026 Paulo Olveira

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.