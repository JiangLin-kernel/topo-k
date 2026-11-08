# topo-k
A Fault-Tolerant Hybrid C-Python Platform for Resilient LLM Inference and Graph-Augmented Reasoning.
# Topo-K: Resilient Edge LLM Inference Platform

**Topo-K** is a pure-C hybrid micro-kernel designed for deterministic graph execution, structural reasoning, and fault tolerance on resource-constrained edge devices (e.g., Android mobile devices).

## Core System-Level Contributions
- **Resilient Inference via Atomic Checkpointing**: Decouples reasoning state from generation. Recovers exactly from an ungraceful `SIGKILL` interruption via POSIX-compliant signal handling (`SIGHUP` + `os.setsid`).
- **Graph-Augmented Generation (GAG)**: Uses a native C engine with a CSR structure to perform exact AST Merkle-tree equivalence checks, substituting probabilistic long-text retrieval with deterministic topological memory.
- **AST-level Static Sandbox**: Intercepts LLM-generated code, physically rejecting malicious patches via AST-level rule matching.

## System Architecture (Textual Block Diagram)
Topo-K operates across four decoupled layers:
1. **Python Worker Layer**: Handles process lifecycle, atomic state persistence, and POSIX signal interception (`SIGHUP`).
2. **IPC Layer**: A high-throughput Unix Pipe bridging the Python orchestrator and the C engine.
3. **Native C Engine**: Executes deterministic CSR graph traversal, AST Merkle hashing, and offline SIMD auto-tuning.
4. **Storage Layer**: Persists atomic checkpoints (`tmp+rename`) and Merkle hash databases.

## Native C Benchmarks (Tested on Dimensity 9300+)

| Operation | Structure | Scale | Latency (p50) |
| :--- | :--- | :--- | :--- |
| **AST Isomorphism** | CSR + Integer Merkle Hash | 100k nodes | **280 us** |
| **Approx. Pathfinding** | Fixed-iteration (10-hop) | 100k nodes, 500k edges | **3.74 ms** |
| **Atomic State Updates**| Lock-Free Concurrency | 40 million ops | **1.09 s** |

## Quick Start (Termux Environment)
```bash
# Clone the repository
git clone https://github.com/JiangLin-kernel/topo-k.git
cd topo-k

# Compile the native C engine with maximum optimization
clang -O3 -march=native rigorous_ast_isomorphism.c -o ast_test
./ast_test

# Run the Python supervisor for SIGKILL recovery test
python3 v8_orchestrator.py
