# A Feasibility Study of Browser-Based Distributed State-Vector Quantum Circuit Simulation

**Volodymyr Serhieiev**  
Independent Researcher  
Email: vova.sergeyev@gmail.com

---

## Abstract

State-vector simulation remains one of the most direct and widely used methods for classical simulation of quantum circuits, but its memory requirements scale exponentially with the number of qubits. This makes simulation increasingly expensive for individual users, especially in educational, exploratory, and early-stage research settings.

This work studies the feasibility of a browser-based distributed state-vector quantum circuit simulator. The idea is - users voluntarily contribute idle computing resources through web browsers. The proposed architecture partitions the global state vector across browser workers and coordinates execution through a central scheduler, job queue, validation layer, and credit-based resource-sharing model.

The main contribution is not a new quantum simulation algorithm, but a system-level feasibility analysis of combining browser-based volunteer computing, state-vector partitioning, and gate-aware scheduling for selected quantum circuit simulation workloads. We identify suitable workloads, architectural constraints, communication bottlenecks, fault-tolerance requirements, and research questions for future prototype evaluation.

---

## 1. Introduction

Classical simulation of quantum circuits is essential for quantum algorithm development, education, debugging, benchmarking, and hardware-independent experimentation. Among classical methods, state-vector simulation is conceptually simple and broadly applicable: the full quantum state of an `n`-qubit system is represented as a vector of `2^n` complex amplitudes.

However, this representation leads to exponential memory growth. With single-precision complex amplitudes, a state vector requires approximately `8 × 2^n` bytes of memory. As a result, simulations above roughly 30 qubits become difficult on commodity machines, and larger simulations typically require high-memory servers, GPUs, or HPC clusters.

At the same time, modern web browsers have become capable execution environments. WebAssembly, Web Workers, and WebGPU make it possible to run non-trivial numerical workloads directly in the browser. This raises a practical question:

> Can browser-based volunteer resources provide a useful distributed execution layer for selected state-vector quantum circuit simulations?

This work proposes and analyzes a browser-based distributed quantum circuit simulation architecture where users can both submit simulation jobs and contribute idle compute resources. The system is designed around a central queue, browser workers, state-vector partitioning, validation of untrusted results, and a credit-based fairness mechanism.

---

## 2. Motivation

The motivation for this architecture is threefold.

First, many quantum circuit simulations are exploratory rather than production-grade. Students, independent researchers, and developers often need affordable access to medium-scale simulation rather than guaranteed HPC-level performance.

Second, browser-based execution removes installation friction. Users can contribute compute resources by opening a web page, without installing a native client.

Third, many simulation workloads are batch-oriented. Parameter sweeps, repeated circuit evaluations, educational tasks, and independent circuit families may tolerate queue latency and heterogeneous worker performance better than tightly coupled HPC workloads.

This work is not intended to replace optimized simulators running on GPUs or supercomputers. Instead, it explores a complementary access model for selected workloads where convenience, shared access, and low deployment friction are valuable.

---

## 3. Background

### 3.1 State-Vector Simulation

In state-vector simulation, an `n`-qubit pure quantum state is represented as:

```text
|ψ⟩ ∈ C^(2^n)
```

Each quantum gate is applied as a linear transformation over the relevant amplitudes. Single-qubit gates update pairs of amplitudes whose indices differ by one bit. Multi-qubit gates affect larger structured groups of amplitudes.

The main advantage of state-vector simulation is generality. The main disadvantage is exponential memory growth.

Approximate memory usage for complex64 amplitudes is:

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

This memory scaling motivates distributed partitioning of the state vector.

### 3.2 Browser-Based Volunteer Computing

Volunteer computing distributes computational tasks across machines contributed by users. A browser-based model reduces participation friction because the compute client can be delivered as a web application.

However, browser workers are heterogeneous and unreliable. They may disconnect, close the tab, become throttled in the background, or run on devices with limited memory and battery constraints. Any practical system must therefore include fault tolerance, validation, task reassignment, and conservative resource limits.

### 3.3 Distributed Quantum Circuit Simulation

Distributed state-vector simulators partition the state vector across multiple workers or nodes. Gates whose amplitude dependencies lie within a partition can be executed locally. Gates whose dependencies cross partitions require communication.

For browser-based volunteer workers, communication cost is particularly important. Unlike tightly connected HPC clusters, volunteer browsers communicate over heterogeneous public networks with variable latency and bandwidth.

---

## 4. Proposed Architecture

The proposed system consists of the following components:

```text
User submits circuit
        ↓
Frontend validates input
        ↓
Coordinator creates simulation job
        ↓
Circuit parser extracts gate list
        ↓
Scheduler partitions state vector
        ↓
Tasks are assigned to browser workers
        ↓
Workers execute WASM/WebGPU kernels
        ↓
Partial results are validated
        ↓
Aggregator constructs final output
```

The major components are:

1. **Web frontend** for circuit submission, queue inspection, and result viewing.
2. **Coordinator API** for job management, worker registration, and orchestration.
3. **Job queue** for pending, running, failed, and completed tasks.
4. **Gate-aware scheduler** for partition assignment and communication-aware execution.
5. **Browser worker runtime** using Web Workers, WebAssembly, and optionally WebGPU.
6. **Validation layer** for checksums, redundant execution, and result integrity.
7. **Credit ledger** for fair resource sharing between submitters and contributors.

---

## 5. State Partitioning Model

Let the global state vector be:

```text
ψ = [a₀, a₁, ..., a_(2^n - 1)]
```

The vector is partitioned into `p` blocks:

```text
ψ = ψ₀ ∪ ψ₁ ∪ ... ∪ ψₚ₋₁
```

A simple initial strategy is contiguous amplitude partitioning:

```text
Partition 0: amplitudes 0 ... k-1
Partition 1: amplitudes k ... 2k-1
Partition 2: amplitudes 2k ... 3k-1
...
```

For a single-qubit gate acting on qubit `q`, amplitude pairs differ in bit `q`. If both amplitudes in a pair are located in the same partition, the gate can be applied locally. If the pair spans different partitions, inter-worker communication is required.

This distinction gives rise to the key scheduling problem: minimize the number and cost of non-local gate operations.

---

## 6. Gate-Aware Scheduling

The scheduler is the central research component of the system.

A naive scheduler assigns partitions to workers based only on availability. A gate-aware scheduler additionally considers the structure of the circuit and the expected communication cost of each gate block.

The scheduler may use:

- worker memory capacity;
- CPU benchmark score;
- optional GPU/WebGPU availability;
- network latency;
- bandwidth estimate;
- historical reliability;
- churn probability;
- task duration estimate;
- partition size;
- gate locality;
- expected synchronization cost.

A possible scheduling objective is:

```text
minimize total_execution_time =
    local_compute_time
  + inter_partition_communication_time
  + validation_overhead
  + reassignment_overhead
```

Subject to:

```text
worker_memory_limit
worker_availability
job_priority
credit_balance
validation_policy
```

The system can also apply circuit transformations or qubit remapping to reduce communication, although this is left as future work.

---

## 7. Credit-Based Resource Sharing

The system may use a non-monetary credit mechanism to encourage fair use.

Users earn credits by contributing compute resources:

```text
credits_earned = f(completed_tasks, uptime, reliability, task_size, validation_score)
```

Users spend credits when submitting simulation jobs:

```text
job_cost = g(qubits, depth, shots, estimated_memory, priority)
```

This mechanism is intended as a fairness layer rather than a cryptocurrency or financial token. Its purpose is to align user incentives: users who consume shared simulation resources are encouraged to contribute resources back to the system.

---

## 8. Fault Tolerance and Validation

Browser-based volunteer workers cannot be assumed to be reliable or trusted.

The system should support:

- task leases;
- heartbeat messages;
- timeout-based reassignment;
- checkpointing;
- redundant execution for selected tasks;
- checksum validation;
- worker reliability scoring;
- rejection of inconsistent results.

A typical successful task lifecycle is:

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

A failure lifecycle is:

```text
Task assigned
        ↓
Worker disconnects
        ↓
Lease expires
        ↓
Task returns to queue
        ↓
Task is assigned to another worker
```

Validation is especially important because workers are volunteer devices outside the control of the coordinator. The coordinator should never execute arbitrary user code. Users submit circuits, while workers execute a fixed simulator kernel.

---

## 9. Suitable Workloads

The architecture is best suited for:

- educational quantum computing tasks;
- low-to-medium qubit simulations;
- shallow circuits;
- parameter sweeps;
- batch-oriented simulation;
- repeated execution of related circuits;
- circuits with limited global communication;
- exploratory algorithm development.

It is less suitable for:

- deep random circuits;
- circuits with frequent global entanglement;
- latency-sensitive simulations;
- workloads requiring large all-to-all synchronization;
- confidential simulations without a private execution mode;
- arbitrary large-scale circuits beyond memory and bandwidth limits.

---

## 10. Evaluation Plan

A prototype evaluation should compare:

- single-browser execution;
- multiple browser workers on one machine;
- multiple browser workers across machines;
- WebAssembly versus JavaScript kernels;
- WebAssembly versus WebGPU kernels where available;
- low-churn versus high-churn worker settings;
- local-only gates versus communication-heavy gates.

Suggested benchmark circuit families include:

- GHZ circuits;
- QFT circuits;
- random Clifford+T circuits;
- low-depth variational ansatz circuits;
- simple Grover circuits;
- random shallow circuits.

Important metrics include:

- total execution time;
- queue wait time;
- speedup over single-browser execution;
- worker utilization;
- communication overhead;
- validation overhead;
- task failure rate;
- reassignment rate;
- credit fairness;
- maximum practical circuit size under given resource assumptions.

---

## 11. Limitations

The proposed architecture has several limitations.

First, browser workers are unreliable. Tabs may close, devices may sleep, and browsers may throttle background execution.

Second, network communication is a major bottleneck. Distributed state-vector simulation can require significant synchronization, especially for gates that act across partition boundaries.

Third, the browser execution environment is heterogeneous. Devices vary widely in CPU speed, memory, thermal behavior, GPU support, and browser API availability.

Fourth, this approach does not provide quantum speedup. It is a classical simulation architecture for quantum circuits.

Fifth, the system should not claim to efficiently simulate arbitrary large quantum circuits. The exponential memory requirement remains fundamental.

---

## 12. Research Questions

This feasibility study motivates several research questions:

1. Which circuit classes are most suitable for browser-based distributed state-vector simulation?
2. How much overhead is introduced by browser-based worker heterogeneity?
3. Can gate-aware scheduling reduce inter-partition communication enough to make the system useful?
4. What is the practical upper bound on qubit count for realistic browser-worker pools?
5. How should credits be assigned to balance fairness and system throughput?
6. How much redundant execution is needed to validate untrusted volunteer results?
7. Can WebGPU significantly improve the performance of browser-based simulation kernels?

---

## 13. Expected Contributions

This work aims to contribute:

1. A proposed architecture for browser-based volunteer distributed state-vector quantum circuit simulation.
2. A partitioning and scheduling model centered on gate locality and communication cost.
3. A credit-based resource-sharing model for balancing compute contribution and simulation access.
4. A fault-tolerance and validation model for unreliable browser workers.
5. A benchmarking plan for future prototype evaluation.

---

## 14. Conclusion

This draft presents a feasibility study of browser-based distributed state-vector quantum circuit simulation. The proposed system combines browser-based volunteer computing, state-vector partitioning, gate-aware scheduling, job queueing, credit-based fairness, and validation of untrusted workers.

The architecture is not intended to replace HPC-grade simulators. Instead, it explores a lower-friction, shared-access model for selected quantum circuit simulation workloads, especially in educational, exploratory, and batch-oriented contexts.

A realistic next step is to implement a minimal prototype with a WebAssembly state-vector kernel, a central task queue, browser worker registration, simple partitioning, and benchmark circuits. Such a prototype would make it possible to evaluate whether the proposed architecture provides practical value and where its scaling limits appear.

---

## References

> Placeholder references to be replaced with full BibTeX entries before arXiv submission.

1. Distributed state-vector quantum circuit simulators, including QuEST, Qulacs, qsim, and related systems.
2. Browser-based volunteer computing systems using JavaScript, WebAssembly, WebRTC, or Web Workers.
3. Quantum circuit simulation literature on state-vector simulation, tensor-network simulation, and hybrid approaches.
4. WebAssembly and WebGPU documentation for browser-based numerical computing.
5. Scheduling and fault-tolerance literature for heterogeneous volunteer distributed systems.
