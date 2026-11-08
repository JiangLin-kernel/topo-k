# Topo-K: A Fault-Tolerant Hybrid C-Python Platform for Resilient LLM Inference and Graph-Augmented Reasoning on Edge Devices

**Authors**: Anonymous Authors  
**Affiliation**: Independent Researcher  
**Date**: November 8, 2026

## Abstract
Deploying large language models (LLMs) on resource-constrained edge devices is constrained by the fragility of long-chain reasoning, context dilution, and unsupervised code execution risks. We present Topo-K, a hybrid C-Python platform designed for resilient inference orchestration and deterministic topological memory. 

Topo-K makes four core system-level contributions:
1. **Resilient Inference via Atomic Checkpointing**: Using a Python worker driven by POSIX-compliant signal handling, Topo-K recovers from an ungraceful `SIGKILL` interruption by detecting parent orphanhood and reloading the exact reasoning state.
2. **Graph-Augmented Generation (GAG)**: To overcome KV Cache bottlenecks, Topo-K utilizes a native C engine with a Compressed Sparse Row (CSR) structure, executing exact integer-based Merkle-tree AST equivalence checks (100k nodes) in 280 microseconds, substituting probabilistic long-text retrieval with deterministic memory.
3. **AST-level Static Sandbox**: A user-space static sandbox intercepts LLM-generated code, physically rejecting malicious patches that attempt process orchestration overrides via AST-level rule matching.
4. **Hardware-Aware Auto-Tuning**: An offline SIMD optimizer reduces a kernel's latency from 123.55 ms to 24.49 ms. We provide rigorous analysis of failure modes: I-Cache exhaustion due to loop unrolling, and a 1-ulp floating-point divergence that causes sampling boundary flips.

## 1. Introduction
While LLMs excel at generative tasks, their deployment in edge applications is constrained by their probabilistic nature and lack of robust state management. In long-chain reasoning (e.g., code generation), an Out-of-Memory (OOM) event or process kill results in total context loss. Furthermore, using expanding KV Caches for long-text retrieval is prone to context rot. Finally, executing unsupervised LLM-generated code poses severe security risks. 

We present Topo-K, a hybrid platform that treats the LLM as a generative peripheral while utilizing a deterministic C-engine and a Python supervisor as the resilient reasoning core.

## 2. Related Work
**Edge LLM Serving and OOM Challenges**: Frameworks like llama.cpp [1] and MLC-LLM [2] optimize memory compression, but lack native recovery mechanisms after OS-level kills. Systems like EdgeMoE [3] and LLMCad [4] optimize memory access but do not address process-level state persistence. Topo-K addresses this by decoupling reasoning state from generation.

**Graph-Augmented Generation (GAG)**: Standard RAG [5] suffers from semantic drift. GRAG [6] and topology-enhanced prompting [7] introduce graphs but require high construction overhead. Topo-K uses exact CSR-based AST isomorphism for zero-construction topological memory.

**State-Space Simulation**: Analyzing state-space perturbations is crucial for understanding error propagation [8]. Topo-K leverages its checkpointing mechanism to capture precise divergence footprints.

**Process Isolation**: Kernel-level seccomp [9] restricts system calls but is complex for rapid LLM patch execution. Topo-K introduces a lightweight user-space supervisor.

## 3. System Architecture and Resilience
### 3.1 System Block Diagram
Topo-K consists of four layers: The Python Supervisor orchestrates process lifecycles; the IPC layer bridges the supervisor and C engine; the Native C Engine executes CSR graph algorithms; and the Storage Layer handles atomic checkpoints.

### 3.2 Worker-Driven Resilient Recovery
To ensure resilience when the parent process is abruptly terminated, Topo-K relies on the surviving child process (Worker) to detect orphanhood and recover the state.

**Algorithm 1: Worker-Driven Atomic Recovery**
1. procedure StartInference(Task)
2.     Parent spawns Worker
3.     Worker initializes cursor <- 0, k <- 10  (cursor = completed units)
4.     Register SIGHUP handler: g_orphan_flag = 1
5.     while cursor < |Task| do
6.         Execute unit cursor
7.         if cursor mod k == 0 then
8.             state_manager.atomic_write(cursor)
9.         if g_orphan_flag == 1 then
10.            cursor <- state_manager.read()
11.            os.setsid()  (Detach from dead parent)
12.            g_orphan_flag = 0
13.        cursor <- cursor + 1
14.    end while
15. end procedure

*Implementation Note*: To comply with POSIX signal safety, the `SIGHUP` handler does not perform I/O. It merely sets a `volatile sig_atomic_t` flag. The main event loop detects this flag and safely executes checkpoint loading and `os.setsid()` outside the signal context, preventing deadlocks.

## 4. Graph-Augmented Generation (GAG)
**Table 1: Native C Graph Engine Benchmarks**
| Operation | Structure | Scale | Latency (p50) |
| :--- | :--- | :--- | :--- |
| AST Isomorphism | CSR + Integer Merkle Hash | 100k nodes | 280 us |
| Approx. Pathfinding | Fixed-iteration (10-hop) | 100k nodes, 500k edges | 3.74 ms |

The 280 us includes complete AST parsing and integer-based Merkle tree construction. Hash collisions are mitigated via multi-salt verification.

## 5. AST-level Static Sandbox
Topo-K introduces a user-space AST-level static sandbox enforced by the Python supervisor. The policy rules are implemented as AST-level syntax trees, statically matched against incoming patches.
**Threat Model**: The adversary is an LLM generating process orchestration patches.
* Benign Patch: PATCH_SAFE -> ALLOW.
* Malicious Patch: PATCH_OVERRIDE_WATCHDOG -> REJECT.

*Scope Boundary*: This paper focuses on process-orchestration-level patches. Resource exhaustion and filesystem access are delegated to OS-level seccomp restrictions.

## 6. Hardware-Aware Auto-Tuning and Failure Modes
### 6.1 Auto-Tuning Overhead
The SIMD optimizer utilizes a gene-based mutation search. **Cost**: 50 configurations, taking 15 minutes of CPU time (one-time offline overhead). A cache mechanism prevents redundant tuning. The tuner converged on the v3 batch-pipeline strategy.

*Throughput Clarification*: The 3.68 GFLOP/s effective throughput is strictly constrained by Python-C IPC overhead and FUSE filesystem latency in Termux, not the Dimensity 9300+ SoC peak. It establishes relative speedups.

### 6.2 The 1-ulp Divergence and Hash Causal Logic
The 1-ulp mismatch in v3 originates from instruction reordering. While AST hashing is strictly integer-based and immune to float errors, the 1-ulp divergence alters float32 logits during LLM sampling. This shift at the decision boundary flips the argmax, generating a different token. Consequently, the resulting code structure diverges, yielding a different discrete AST hash. Thus, v3 is rejected for strict reasoning tasks.

### 6.3 Negative Results: Integral Image Failure
Using an integral image for box filtering resulted in a 0.7x slowdown (Naive: 0.010s vs. Integral: 0.014s), highlighting severe memory-bandwidth bottlenecks on edge devices.

## 7. Divergence Propagation in State-Space Simulation
To demonstrate the general applicability of our atomic checkpointing mechanism beyond LLM inference, we apply Topo-K to a state-space simulation. In a 64x64 discrete ecological grid, we perturbed a single initial state and executed a 30-step simulation. The engine measured a precise divergence footprint of 404 cells (9.86%). This confirms that our mechanisms can handle state divergence in complex discrete simulations.

## 8. Evaluation Setup & Discussion
**Hardware**: All experiments were conducted on a commercial edge device: iQOO Z9 Turbo+, equipped with a MediaTek Dimensity 9300+ SoC (8-core) and 16GB RAM, running Android 14.
**Environment**: The C engine was compiled and executed within the Termux environment using clang version 14.0.6 with flags -O3 -march=native. Memory bandwidth is capped by the FUSE filesystem /storage/emulated/0.
**Reproducibility**: Raw logs, compiled binaries, and scripts will be open-sourced under AGPL-3.0.

## 9. Conclusion
Topo-K demonstrates that edge LLM applications can be hardened using a hybrid architecture. By combining POSIX-compliant process recovery, exact graph-augmented memory, and strict process-level sandboxing, we provide a resilient substrate for deploying AI in high-stakes environments.

## References
[1] G. Gerganov, "llama.cpp: Port of LLaMA model in pure C/C++," GitHub repository, 2023.
[2] MLC Team, "MLC-LLM: Universal Compilation for LLMs," GitHub repository, 2023.
[3] R. Yi et al., "EdgeMoE: Empowering Sparse Large Language Models on Mobile Devices," ACM MobiCom, 2023.
[4] D. Xu et al., "LLMCad: Fast and Scalable On-device Large Language Model Inference," arXiv:2309.04255, 2023.
[5] P. Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks," NeurIPS, 2020.
[6] Y. Hu et al., "GRAG: Graph Retrieval-Augmented Generation," NAACL 2025.
[7] R. Wang, K. Qin, et al., "Rethinking Graph Reasoning: Equip Large Language Models with Topology-Enhanced Prompt," arXiv:2405.10512, 2024.
[8] M. Mitchell, "Complexity: A Guided Tour," Oxford University Press, 2009.
[9] Linux Kernel Documentation, "Seccomp: Secure Computing Mode," 2024.
