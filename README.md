# Distributed Graph Algorithms with MPI

Runs two distributed algorithms over generated network graphs using MPI:
1. **Leader election** (FloodMax-style)
2. **Single-source shortest paths** (distributed Dijkstra baseline)

Workflow:

`Graph generation/import -> graph enrichment (positive weights + IDs) -> partition across MPI ranks -> distributed algorithm run -> logs + metrics + experiment summary`

---

## Requirements

- macOS or Linux
- OpenMPI
- CMake (3.10+)
- Python 3
- C++17 compiler

On macOS:
```bash
brew install open-mpi cmake
```

---

## Repository layout

```
distributed-graph-mpi/
├── netgamesim/
├── configs/
├── tools/
│   ├── graph_export/
│   └── partition/
├── mpi_runtime/
├── experiments/
├── outputs/
├── tests/
├── README.md
└── REPORT.md
```

---

## Build instructions

Clone the repository and build from the root directory:

```bash
git clone https://github.com/KrishnSo/distributed-graph-mpi.git
cd distributed-graph-mpi
cmake -S mpi_runtime -B build
cmake --build build
```

This builds the runtime executable at `build/ngs_mpi`.

---

## 1) Generate graph (synthetic path)

Small graph:
```bash
./tools/graph_export/run.sh configs/small.json outputs/graphs/graph_small.json
```

Medium graph:
```bash
./tools/graph_export/run.sh configs/medium.json outputs/graphs/graph_medium.json
```

This creates a connected graph, assigns positive edge weights, stores a random seed in the output JSON, and normalizes node IDs to `0..N-1`.

---

## 2) Generate graph (NetGameSim-style import path)

```bash
python3 tools/graph_export/export_graph.py \
  --raw-netgamesim outputs/raw/raw_netgamesim_graph.json \
  --seed 12345 \
  --out outputs/graphs/graph_from_netgamesim.json
```

Or using the config wrapper:
```bash
./tools/graph_export/run.sh configs/netgamesim_import.json outputs/graphs/graph_from_netgamesim.json import
```

---

## 3) Partition graph across ranks

Mod partition:
```bash
./tools/partition/run.sh outputs/graphs/graph_small.json --ranks 4 --strategy mod --out outputs/partitions/part_small_r4_mod.json
```

Block partition:
```bash
./tools/partition/run.sh outputs/graphs/graph_small.json --ranks 4 --strategy block --out outputs/partitions/part_small_r4_block.json
```

Partition output includes: `owner` map, `local_nodes` per rank, `ghost_nodes` per rank, `cut_edges`.

---

## 4) Run MPI runtime

Run both algorithms:
```bash
mpirun -n 4 ./build/ngs_mpi --graph outputs/graphs/graph_small.json --part outputs/partitions/part_small_r4_mod.json --algo both --source 0
```

Leader election only:
```bash
mpirun -n 4 ./build/ngs_mpi --graph outputs/graphs/graph_small.json --part outputs/partitions/part_small_r4_mod.json --algo leader --rounds 200
```

Dijkstra only:
```bash
mpirun -n 4 ./build/ngs_mpi --graph outputs/graphs/graph_small.json --part outputs/partitions/part_small_r4_mod.json --algo dijkstra --source 0
```

Optional args: `--rounds R` (max rounds for leader election, `0` = until convergence), `--seed S` (records run seed in logs for reproducibility).

---

## 5) Run experiments

Partition strategy comparison:
```bash
./experiments/run_partition_compare.sh
```

Small vs. medium scaling:
```bash
./experiments/run_scale_compare.sh
```

Outputs: raw logs in `outputs/experiment_logs/`, summaries in `outputs/summaries/`.

---

## Tests

```bash
./tests/run_tests.sh
```

Runs: export tool tests, partition tool tests, reference Dijkstra correctness test, and an MPI smoke test (when `mpirun` and the build are available).

---

## Logging and metrics

Per-rank runtime logs: `outputs/logs/runtime_rank0.log`, `runtime_rank1.log`, etc.

Runtime metrics include: leader-election rounds, Dijkstra iterations, runtime per algorithm, total message count, and approximate bytes sent.

---

## Reproducibility

All graph generation is seed-based. The seed for each graph is stored in the output JSON, allowing exact reproduction of experiments. Runtime seeds can also be passed via CLI.

---

## Assumptions

- Graph is connected
- Edge weights are strictly positive
- Node IDs are normalized to `0..N-1`
- Partition rank count matches `mpirun -n`

---

## Quick start

```bash
git clone https://github.com/KrishnSo/distributed-graph-mpi.git
cd distributed-graph-mpi
cmake -S mpi_runtime -B build && cmake --build build
./tools/graph_export/run.sh configs/small.json outputs/graphs/graph_small.json
./tools/partition/run.sh outputs/graphs/graph_small.json --ranks 4 --strategy mod --out outputs/partitions/part_small_r4_mod.json
mpirun -n 4 ./build/ngs_mpi --graph outputs/graphs/graph_small.json --part outputs/partitions/part_small_r4_mod.json --algo both --source 0
```

If OpenMPI complains about the network interface on macOS:
```bash
mpirun --mca btl tcp,self --mca pml ob1 -n 4 ./build/ngs_mpi ...
```

---

## Acknowledgment

Graph generation builds on [NetGameSim](https://github.com/0x1DOCD00D/NetGameSim) by Professor Grechanik.
