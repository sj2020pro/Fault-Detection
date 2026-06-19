# 5G Network RCA — `new_rcd`

Root Cause Analysis (RCA) for 5G network faults using the **SCRCA algorithm**

---

## Overview

The algorithm identifies the most likely root-cause metrics for a fault by:

1. Splitting a metrics time-series window into a **normal baseline** (60 s before fault) and an **anomalous window** (60 s after fault start).
2. Running a **PC skeleton-discovery** algorithm seeded with a pre-computed initial graph (`learned_initial_graph.json`) rather than starting from a complete graph. This biases the search toward known causal relationships in the 5G user plane
3. Ranking F-node neighbours by p-value and effect size to produce a top-k root cause list.

### Supported fault types

The algorithm works on any fault type whose scenarios include `root_cause_metrics` in `ground_truth.json`. The `5g_dataset` included in this folder covers:

| Fault | Target |
|---|---|
| `cpu_stress` | `upf2`, `gnb2` |
| `gnb_to_core_partition` | `gnb2` |
| `upf_bandwidth_cap` | `upf2` |
| `pfcp_storm` | `upf2` |
| `ue_context_churn` | `gnb2` |

---

## Setup

Requires **Python 3.8** and the **cmu-phil fork** of `causal-learn==0.1.2.3`. On Python 3.8, `pip install causal-learn==0.1.2.3` installs the correct cmu-phil build. On Python 3.9+ the same version resolves to the py-why fork which has breaking API changes (`CausalGraph.remove_edge`, `p_values`, `mi`, `no_ci_tests` and `append_to_mi` are absent) and will not work.

```bash
# 1. Create and activate a Python 3.8 virtual environment
python3.8 -m venv env
source env/bin/activate

# 2. Install dependencies
pip install --upgrade pip
pip install -r requirements.txt
```

> **Note:** `causal-learn==0.1.2.3` is pinned and **must be installed under Python 3.8**
> to get the cmu-phil fork with the required internal API.
> `pyarrow` is required for reading `.parquet` metric files.

---

## Repository structure

```
new_rcd/
├── rcd.py                     # Main RCA entry point (load_offline_scenario, top_k_rc)
├── utils.py                   # PC algorithm, graph building, data preprocessing
├── learned_initial_graph.json # Pre-saved initial graph (94 metric-metric edges)
├── evaluate.py                # Evaluation script (see below)
├── requirements.txt
├── 5g_dataset/                # Example dataset (120 fault scenarios)
│   ├── cpu_stress-gnb2_<id>/
│   │   ├── metrics.parquet
│   │   └── ground_truth.json
│   └── ...
└── README.md
```

---

## Dataset format

Each scenario is a directory containing exactly two files:

- **`metrics.parquet`** — a Parquet file with a datetime index and one column per metric (`target::metric_name`). Rows are ~1-second observations.
- **`ground_truth.json`** — fault metadata including the known root-cause metrics:
  ```json
  {
    "fault": {
      "fault_type": "cpu_stress",
      "target": "upf2",
      "actual_start_utc": "2026-06-10T12:00:00Z",
      "actual_end_utc":   "2026-06-10T12:01:00Z",
      "root_cause_metrics": [
        "upf2::cpu_usage_rate",
        "upf2::cpu_throttle_rate"
      ]
    }
  }
  ```

The `root_cause_metrics` field is required for accuracy evaluation. Scenario directories may be named freely; the script reads `ground_truth.json` to determine the fault type and ground truth.

---

## Running the evaluation

### Quickstart — evaluate on `5g_dataset` with the learned initial graph

```bash
python evaluate.py
```

### Compare learned initial graph vs. complete graph (side-by-side with delta)

```bash
python evaluate.py --compare
```

### Use the complete graph only (PC baseline)

```bash
python evaluate.py --mode complete
```

### Point to a different dataset folder

```bash
python evaluate.py --dataset /path/to/your/dataset --compare
```

### All options

```
usage: evaluate.py [-h] [--dataset DATASET] [--mode {learned_initial_graph,complete,...}]
                   [--k K] [--bins BINS] [--before BEFORE] [--after AFTER] [--compare]

  --dataset   Path to folder of scenario sub-directories (default: 5g_dataset)
  --mode      Initial graph mode:
                learned_initial_graph  (default) — pre-saved 94-edge graph
                complete               — full PC from complete graph
  --k         Top-k root causes to return (default: 3)
  --bins      Discretisation bins for chi-squared CI test (default: 2)
  --before    Seconds of normal data before fault start (default: 60)
  --after     Seconds of anomalous data after fault start (default: 60)
  --compare   Also run complete graph and print a delta table
```

---


## Monitored metrics (20 total all related to one network slice (slice 2))

| Metric | Description |
|---|---|
| `upf2::cpu_usage_rate` | UPF2 CPU utilisation |
| `upf2::cpu_throttle_rate` | UPF2 CPU throttling |
| `gnb2::cpu_usage_rate` | gNB2 CPU utilisation |
| `gnb2::cpu_throttle_rate` | gNB2 CPU throttling |
| `upf2::net_rx_packets_rate` | UPF2 network receive packet rate |
| `upf2::net_tx_packets_rate` | UPF2 network transmit packet rate |
| `gnb2::net_rx_packets_rate` | gNB2 network receive packet rate |
| `gnb2::net_tx_packets_rate` | gNB2 network transmit packet rate |
| `gnb2::n3_rx_bps` | gNB2 N3 interface receive throughput |
| `gnb2::n3_tx_bps` | gNB2 N3 interface transmit throughput |
| `upf2::n3_rx_bps` | UPF2 N3 interface receive throughput |
| `upf2::n3_tx_bps` | UPF2 N3 interface transmit throughput |
| `upf2::ogstun_rx_bps` | UPF2 GTP-U tunnel receive throughput |
| `upf2::ogstun_tx_bps` | UPF2 GTP-U tunnel transmit throughput |
| `gnb2::eth0_rx_bps` | gNB2 physical interface receive throughput |
| `upf2::eth0_tx_bps` | UPF2 physical interface transmit throughput |
| `upf2::n4_rx_packet_rate` | UPF2 N4 (PFCP) receive packet rate |
| `upf2::n4_rx_bps` | UPF2 N4 (PFCP) receive throughput |
| `gnb2::n2_rx_packet_rate` | gNB2 N2 (NGAP) receive packet rate |
| `gnb2::n2_tx_packet_rate` | gNB2 N2 (NGAP) transmit packet rate |
