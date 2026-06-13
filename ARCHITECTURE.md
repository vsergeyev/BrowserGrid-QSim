# Proposed Architecture

## BrowserGrid-QSim: Volunteer Browser-Based Distributed State-Vector Quantum Circuit Simulator

**Author:** Volodymyr Serhieiev  
**Contact:** vova.sergeyev@gmail.com

---

## 1. Overview

BrowserGrid-QSim is a proposed architecture for a browser-based volunteer distributed quantum circuit simulator.

The core idea is to allow users to contribute idle computational resources from their web browsers and, in return, receive access to a shared queue of quantum circuit simulation jobs. The system focuses on state-vector simulation of quantum circuits, where the global quantum state is partitioned across multiple browser workers.

This architecture is intended for:

- educational quantum computing experiments;
- exploratory circuit simulation;
- distributed systems research;
- feasibility analysis of browser-based volunteer computing;
- future benchmarking against local and server-side simulators.

It is not intended to replace HPC-grade quantum simulators. Instead, it explores whether browser-based volunteer resources can provide a useful shared simulation layer for selected classes of quantum circuits.

---

## 2. High-Level Architecture

```text
+--------------------+
|    User Browser    |
|  Circuit Submitter |
+---------+----------+
          |
          v
+--------------------+
|   Web Frontend     |
|  QASM / Circuit UI |
+---------+----------+
          |
          v
+--------------------+
|  Coordinator API   |
|  Auth / Jobs / API |
+---------+----------+
          |
          v
+--------------------+
|    Job Queue       |
| Pending / Running  |
| Completed / Failed |
+---------+----------+
          |
          v
+-----------------------------+
| Gate-Aware Scheduler        |
| Partitioning / Assignment   |
| Worker Scoring / Credits    |
+---------+-------------------+
          |
          v
+-----------------------------+
| Volunteer Browser Workers   |
| WASM / Web Workers / WebGPU |
+---------+-------------------+
          |
          v
+-----------------------------+
| Result Aggregator           |
| Validation / Checkpoints    |
| Sampling / Final Results    |
+-----------------------------+
```

---

## 3. Main Components

### 3.1 Web Frontend

The frontend allows users to:

- submit quantum circuits;
- upload or paste OpenQASM;
- configure simulation parameters;
- inspect queue status;
- view simulation results;
- optionally contribute browser resources as workers.

Possible frontend stack:

- TypeScript;
- React, Svelte, or HTMX;
- Web Workers;
- WebAssembly;
- optional WebGPU support.

---

### 3.2 Coordinator API

The coordinator is the central control plane of the system.

Responsibilities:

- user authentication;
- job submission;
- job lifecycle management;
- worker registration;
- worker heartbeat tracking;
- credit accounting;
- result collection;
- task reassignment after worker failure.

Possible backend stack:

- Python FastAPI;
- Node.js;
- Go;
- PostgreSQL for persistent metadata;
- Redis, NATS, RabbitMQ, or SQS for queueing.

---

### 3.3 Job Queue

The queue stores simulation jobs and decomposed execution tasks.

A job may include:

- circuit source;
- number of qubits;
- circuit depth;
- gate list;
- requested shots;
- simulation mode;
- memory estimate;
- priority;
- owner;
- credit cost;
- status.

Example job states:

```text
SUBMITTED
VALIDATED
PARTITIONED
QUEUED
RUNNING
AGGREGATING
COMPLETED
FAILED
CANCELLED
```

---

### 3.4 Gate-Aware Scheduler

The scheduler is the key research component of the architecture.

Its task is to assign state-vector partitions and gate execution tasks to browser workers while minimizing inter-worker communication.

The scheduler considers:

- available worker RAM;
- CPU benchmark score;
- optional GPU/WebGPU availability;
- network latency;
- bandwidth estimate;
- historical reliability;
- worker churn probability;
- tab visibility;
- battery or power status when available;
- expected task duration;
- communication cost of gates.

The scheduler should distinguish between:

- local gates, which can be executed within a single partition;
- non-local gates, which require communication between partitions.

---

### 3.5 Browser Worker Runtime

A browser worker contributes compute resources through an opt-in runtime.

Possible implementation:

- Web Worker for background execution;
- WebAssembly kernel for state-vector operations;
- SharedArrayBuffer where available;
- WebGPU backend for accelerated linear algebra;
- heartbeat mechanism;
- task lease and timeout handling.

Worker responsibilities:

- receive assigned partition tasks;
- execute gate kernels;
- compute checksums;
- return partial results;
- report progress;
- periodically send heartbeat messages.

---

### 3.6 State-Vector Kernel

The simulator uses a state-vector representation:

```text
|ψ⟩ ∈ C^(2^n)
```

For `n` qubits, the state vector contains `2^n` complex amplitudes.

Memory usage:

```text
complex64:  8 bytes per amplitude
complex128: 16 bytes per amplitude
```

Approximate memory requirements for complex64:

| Qubits | Amplitudes | Memory |
|------:|-----------:|-------:|
| 30 | 1.07 billion | ~8 GB |
| 31 | 2.15 billion | ~16 GB |
| 32 | 4.29 billion | ~32 GB |
| 33 | 8.59 billion | ~64 GB |
| 34 | 17.18 billion | ~128 GB |
| 35 | 34.36 billion | ~256 GB |
| 36 | 68.72 billion | ~512 GB |
| 37 | 137.44 billion | ~1 TB |
| 38 | 274.88 billion | ~2 TB |

This exponential memory requirement is the main motivation for distributed partitioning.

---

## 4. State Partitioning Model

The global state vector is divided into partitions:

```text
ψ = ψ₀ ∪ ψ₁ ∪ ... ∪ ψₚ₋₁
```

Where:

- `ψ` is the global state vector;
- `p` is the number of partitions;
- each partition is assigned to one or more browser workers.

A simple partitioning strategy is contiguous amplitude blocks:

```text
Partition 0: amplitudes 0 ... k-1
Partition 1: amplitudes k ... 2k-1
Partition 2: amplitudes 2k ... 3k-1
...
```

Single-qubit gates update pairs of amplitudes whose indices differ in one bit. If both amplitudes are in the same partition, the operation is local. If they are in different partitions, the operation requires communication.

---

## 5. Execution Flow

### 5.1 Job Submission

```text
User submits OpenQASM / circuit
        ↓
Frontend validates input
        ↓
Coordinator creates job
        ↓
Circuit parser extracts gates and metadata
        ↓
Scheduler estimates memory and communication cost
```

---

### 5.2 Partitioning

```text
Circuit accepted
        ↓
State vector size estimated
        ↓
Partition count selected
        ↓
Partitions assigned to workers
        ↓
Initial state prepared
```

---

### 5.3 Gate Execution

```text
Scheduler selects executable gate block
        ↓
Local tasks are sent to workers
        ↓
Workers execute WASM/WebGPU kernels
        ↓
Partial results are returned
        ↓
Non-local synchronization is performed when required
```

---

### 5.4 Aggregation

```text
All partitions complete
        ↓
Checksums and validation results compared
        ↓
State vector or sampled results aggregated
        ↓
Final output returned to user
```

---

## 6. Credit-Based Resource Sharing

The system may use a credit model to encourage contribution.

Users earn credits by contributing compute resources:

```text
credits_earned = f(completed_tasks, uptime, reliability, task_size, validation_score)
```

Users spend credits when submitting jobs:

```text
job_cost = g(qubits, depth, shots, estimated_memory, priority)
```

The goal is not to introduce a cryptocurrency or financial token, but to provide a simple fairness mechanism for shared access to volunteer resources.

---

## 7. Fault Tolerance

Browser-based volunteer computing is unreliable by design. A worker may disconnect, close the tab, lose network access, or become throttled by the browser.

The system should support:

- task leases;
- periodic heartbeat messages;
- automatic task reassignment;
- checkpointing;
- redundant execution for selected tasks;
- checksum validation;
- worker reliability scoring;
- timeout-based failure detection.

Example lifecycle:

```text
Task assigned
        ↓
Worker heartbeat active
        ↓
Task completed
        ↓
Checksum verified
        ↓
Result accepted
```

Failure case:

```text
Task assigned
        ↓
Heartbeat lost
        ↓
Lease expires
        ↓
Task returned to queue
        ↓
Task assigned to another worker
```

---

## 8. Validation and Integrity

Volunteer workers should not be fully trusted.

Validation methods may include:

- deterministic kernels;
- checksums per partition;
- redundant task execution;
- random spot checks;
- comparison against small reference simulations;
- rejection of inconsistent results;
- reliability scoring per worker.

The coordinator should never execute arbitrary user-provided code. Users submit circuits, not executable programs.

---

## 9. Security Considerations

Security goals:

- protect users who contribute browser resources;
- prevent malicious jobs from abusing workers;
- prevent malicious workers from corrupting results;
- protect private or unpublished circuits;
- avoid arbitrary code execution.

Suggested mechanisms:

- sandboxed WebAssembly runtime;
- fixed simulator kernel;
- strict circuit validation;
- job size limits;
- rate limiting;
- authentication;
- optional private jobs;
- encrypted transport;
- no access to local filesystem;
- explicit opt-in for compute contribution.

---

## 10. Suitable Workloads

The architecture is most suitable for:

- educational simulations;
- low-to-medium qubit circuits;
- shallow circuits;
- batch simulation;
- parameter sweeps;
- independent circuit families;
- circuits with limited global communication;
- exploratory quantum algorithm development.

Less suitable workloads:

- deep random circuits;
- circuits with frequent global entanglement;
- simulations requiring low latency;
- very large state vectors requiring intensive all-to-all communication;
- confidential workloads without a private execution mode.

---

## 11. Benchmarking Plan

Initial benchmarks should compare:

- single browser execution;
- multiple browser workers on one machine;
- multiple browser workers across machines;
- WASM vs JavaScript;
- WASM vs WebGPU where available;
- different circuit families;
- different partition sizes;
- different worker churn scenarios.

Suggested circuit families:

- GHZ circuits;
- QFT circuits;
- random Clifford+T circuits;
- low-depth variational ansatz circuits;
- simple Grover circuits;
- random shallow circuits.

Metrics:

- total execution time;
- queue wait time;
- speedup over single browser;
- worker utilization;
- communication overhead;
- task failure rate;
- reassignment rate;
- validation overhead;
- credit fairness.

---

## 12. Prototype Roadmap

### Phase 1: Local Browser Simulator

- Implement state-vector kernel in JavaScript or Rust/WASM.
- Support a small gate set.
- Parse a subset of OpenQASM 2.0.
- Run circuits locally in a browser worker.

### Phase 2: Central Queue

- Add backend API.
- Add job submission.
- Add task queue.
- Add basic worker registration.
- Execute independent tasks across browsers.

### Phase 3: Distributed State Partitioning

- Implement partitioned state-vector execution.
- Add local and non-local gate classification.
- Add communication between worker partitions through the coordinator.

### Phase 4: Scheduler and Credits

- Add worker scoring.
- Add reliability tracking.
- Add simple credit accounting.
- Add priority queueing.

### Phase 5: Validation and Benchmarking

- Add redundant execution.
- Add checksum validation.
- Benchmark across multiple browsers and machines.
- Prepare research report or arXiv preprint.

---

## 13. Research Questions

This project can support several research questions:

1. Can browser-based volunteer computing provide useful capacity for quantum circuit simulation?
2. Which classes of circuits benefit most from this architecture?
3. How much communication overhead is introduced by distributed state-vector partitioning?
4. Can gate-aware scheduling reduce inter-worker communication?
5. What is the practical upper bound for browser-based state-vector simulation?
6. How should credits be assigned fairly in a mixed submitter/worker system?
7. How much redundancy is required to make volunteer results trustworthy?

---

## 14. Limitations

This architecture has important limitations:

- browser workers are unreliable;
- background tabs may be throttled;
- mobile devices may overheat or lose battery quickly;
- network bandwidth is often the bottleneck;
- large entangled circuits require expensive synchronization;
- browser APIs differ across devices;
- WebGPU is not uniformly available;
- privacy guarantees require careful design;
- the system is not a replacement for HPC simulators.

---

## 15. Positioning

BrowserGrid-QSim should be positioned as a feasibility study and prototype architecture for shared quantum circuit simulation using browser-based volunteer resources.

A realistic claim:

> This system explores whether opt-in browser workers can provide a useful distributed execution layer for selected state-vector quantum circuit simulation workloads.

Claims to avoid:

> This system replaces HPC quantum simulators.

> This system can efficiently simulate arbitrary 100-qubit circuits.

> This system provides quantum speedup.

---

## 16. Future Work

Possible future directions:

- WebGPU acceleration;
- tensor-network simulation mode;
- hybrid state-vector/tensor-network backend;
- peer-to-peer worker communication with WebRTC;
- adaptive qubit remapping;
- checkpoint compression;
- privacy-preserving private jobs;
- integration with Qiskit or Cirq;
- browser-based visual circuit editor;
- public benchmark dashboard;
- arXiv preprint based on prototype results.

---

## 17. Summary

BrowserGrid-QSim proposes a distributed browser-based volunteer computing architecture for state-vector quantum circuit simulation.

The main contribution is not a new quantum simulation algorithm, but a system architecture combining:

- browser-based compute workers;
- state-vector partitioning;
- gate-aware scheduling;
- job queueing;
- credit-based access;
- fault tolerance;
- validation of untrusted volunteer results.

The architecture is especially relevant for educational, exploratory, and batch-oriented quantum circuit simulations where low-cost shared access may be more important than HPC-level performance.
