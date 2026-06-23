# 5G RCA Dataset — Foundation Model Experiments

## Overview

This dataset contains **115 labeled failure scenarios** collected from a live
5G Standalone (SA) network running on Kubernetes.  Each scenario provides
multi-modal observability data (metrics, logs, KPIs) together with a
structured causal-graph analysis produced by a purpose-built RCA algorithm.
The intended use is to benchmark and explore how **foundation models (LLMs)**
can perform root-cause analysis (RCA) of 5G network faults, and to study
which types of context most improve their accuracy and response latency.

---

## Network Background

Read **`network_context.txt`** first.  It gives a complete description of:

- The 5G SA network topology (gNB, UPF, AMF, SMF, NRF, UDM and their interfaces)
- All the monitored metrics, what each measures, and which interface it sits on
- The three end-to-end KPIs (throughput, packet loss, round-trip latency) and
  the severity labels used in the dataset
- The format of every file present per scenario

---

## Dataset Structure

```
Multi_Modal_Dataset/
├── README.md                   ← this file
├── network_context.txt         ← LLM system-context description of the network
└── ep<NNN>_r<RR>f<FF>_<fault_type>__<target>/
    ├── normal.csv              ← 60 s baseline metrics (20 metrics, 1 s resolution)
    ├── anomalous.csv           ← 60 s fault-window metrics (same columns)
    ├── kpi_summary.json        ← structured KPI impact (throughput / loss / latency)
    ├── logs.jsonl              ← NF pod logs spanning the full scenario window
    └── rca_interpretation.json ← RCA algorithm output (causal graph + top-3 prediction)
```

Episode directory names encode:
- `<NNN>` — global episode number
- `r<RR>`   — round number (01–20)
- `f<FF>`   — fault index within the round (1–7)
- `<fault_type>` — one of the seven fault types listed below
- `<target>` — the Kubernetes pod name of the injected NF

### Fault Types (7 classes)

| Fault type | Injected NF | Expected root-cause metrics |
|---|---|---|
| `cpu_stress` on gNB | gnb2 | `gnb2::cpu_usage_rate`, `gnb2::cpu_throttle_rate` |
| `cpu_stress` on UPF | upf2 | `upf2::cpu_usage_rate`, `upf2::cpu_throttle_rate` |
| `gnb_to_core_partition` | gnb2 | `gnb2::n3_rx_bps`, `gnb2::n3_tx_bps` |
| `gnb_to_core_loss` | gnb2 | `gnb2::n3_rx_bps`, `gnb2::n3_tx_bps` |
| `upf_bandwidth_cap` | upf2 | `upf2::ogstun_rx_bps`, `upf2::ogstun_tx_bps` |
| `pfcp_storm` | upf2 | `upf2::n4_rx_packet_rate`, `upf2::n4_rx_bps` |
| `ue_context_churn` | gnb2 | `gnb2::n2_rx_packet_rate`, `gnb2::n2_tx_packet_rate` |

**Ground truth is intentionally excluded from the per-episode folders.**
The ground truth (fault type + target NF + correct root-cause metrics) is
only used during evaluation (see Evaluation section below).

### Class distribution

| Fault type | Episodes |
|---|---|
| cpu_stress / gnb2 | 20 |
| cpu_stress / upf2 | 13 |
| gnb_to_core_partition | 19 |
| gnb_to_core_loss | 12 |
| upf_bandwidth_cap | 13 |
| pfcp_storm | 19 |
| ue_context_churn | 19 |
| **Total** | **115** |

---

## Per-Episode Files — Detailed Format

### `normal.csv` and `anomalous.csv`

CSV with a `timestamp` column (Unix epoch seconds, UTC) and one column per
metric (`<nf>::<metric_name>`).  `normal.csv` covers the 60 s immediately
before fault onset; `anomalous.csv` covers the 60 s starting at onset.
Both files always contain the same 20 columns (some may be near-zero for
faults that do not affect that subsystem).

### `kpi_summary.json`

```json
{
  "throughput": {
    "baseline_mean": 48230000.0,
    "fault_mean": 1200000.0,
    "severity": "severe_drop",
    "description": "Throughput dropped from 48.2 Mbps to 1.2 Mbps"
  },
  "loss_pct": {
    "baseline_mean": 0.2,
    "fault_mean": 87.4,
    "severity": "severe_increase",
    "description": "Packet loss increased from 0.2% to 87.4%"
  },
  "avg_rtt_ms": {
    "baseline_mean": 2.1,
    "fault_mean": 4.3,
    "severity": "normal",
    "description": "Latency within normal range"
  }
}
```

Severity labels come from fixed thresholds defined in `network_context.txt`.

### `logs.jsonl`

One JSON object per line:
```json
{"timestamp": "2024-11-15T10:23:41Z", "nf": "upf2", "level": "error", "message": "GTP tunnel dropped: no forwarding rule"}
```

### `rca_interpretation.json`

Produced by the `effect_gradient_rca` algorithm in `rcd_v2/rcd.py`.
Top-level keys:

- `rca_result.root_causes` — top-3 ranked metrics with effect score, z-score,
  fold-change, direction (spike/drop), and a plain-English sentence
- `causal_graph.source_nodes` — root nodes (no upstream anomalous predecessor)
- `causal_graph.downstream_nodes` — metrics that are effects, not causes
- `causal_graph.edges` — directed edges among changed metrics with interpretation
- `causal_graph.causal_chains` — full propagation paths as strings
- `anomaly_evidence` — per-metric z-score, fold-change, baseline/fault means
- `narrative` — compact paragraph summarising the RCA finding

---

## Baseline Algorithm

The `effect_gradient_rca` algorithm (`rcd_v2/rcd.py`) operates without any
prior knowledge of fault types.

**Baseline accuracy on this 115-episode dataset:**

| Metric | Score |
|---|---|
| Top-1 accuracy | ~72 % |
| Top-3 accuracy | ~100 % |

A prediction is considered correct if at least one of the two ground-truth
root-cause metrics appears in the model's top-k output.

---

## Evaluation Protocol

Ground truth for each episode is a set of **2 root-cause metrics** (see the
fault-type table above).  A prediction is **correct** if at least one of those
two metrics appears in the model's top-k predictions.

Report:
- **Top-1 accuracy** — correct if the single best prediction is in the GT set
- **Top-3 accuracy** — correct if any of the top-3 predictions is in the GT set
- **Mean latency** — wall-clock time from prompt submission to first token /
  full response, averaged over all episodes
- Break results down **per fault type** to identify which classes benefit most
  from each context modality

---

## Proposed Experiments

The core research question is:

> *Which combination of observability context (metrics, logs, KPIs, causal
> graph) gives the best top-1 and top-3 RCA accuracy when fed to a foundation
> model, and at what latency cost?*

### Experiment 1 — Context Ablation Study

Run the LLM on the same 115 episodes with progressively richer context.
For each configuration, record top-1 accuracy, top-3 accuracy, and mean
response latency.

Suggested configurations (cumulative — each adds one modality):

| Config | Input to LLM |
|---|---|
| C0 — metrics only | `normal.csv` + `anomalous.csv` |
| C1 — + KPIs | C0 + `kpi_summary.json` |
| C2 — + logs | C1 + `logs.jsonl` |
| C3 — + causal graph narrative | C2 + `rca_interpretation.json` (narrative field only) |
| C4 — + full causal graph | C2 + `rca_interpretation.json` (all fields) |
| C5 — causal graph only (no raw metrics) | `rca_interpretation.json` + `kpi_summary.json` |

Expected finding: C3/C4 should outperform C0 on top-1 while C5 tests whether
the distilled causal-graph summary alone is sufficient.

### Experiment 2 — System-Context Ablation

Test how much the `network_context.txt` description matters.

| Config | System prompt |
|---|---|
| No context | Generic "you are a network engineer" prompt |
| Partial context | NF descriptions only (no metric definitions) |
| Full context | Complete `network_context.txt` |

Use the best data configuration from Experiment 1.  This isolates the value
of domain knowledge vs. raw observability data.

### Experiment 3 — Model Comparison

Using the best configuration from Experiments 1 and 2, compare multiple
foundation models to understand the capability floor required for this task:

- A small open-source model (e.g. Llama-3-8B or Mistral-7B)
- A mid-size model (e.g. Llama-3-70B or Mixtral 8×22B)
- A frontier API model (e.g. GPT-4o or Claude 3.5 Sonnet)

Report accuracy and latency for each.  The latency comparison is particularly
interesting because smaller models may be fast enough for near-real-time RCA
even if slightly less accurate.

### Experiment 4 — Prompting Strategy

With a fixed model and data configuration, test different prompt structures:

- **Direct answer** — "Which metric is the root cause? Answer with the metric
  name only."
- **Chain-of-thought** — "Think step by step before giving your answer."
- **Structured output** — Ask the model to return JSON with ranked predictions
  and a confidence score.
- **Few-shot** — Include 2–3 solved examples (from other episodes) before the
  query episode.

The few-shot condition tests whether in-context examples help the model learn
the `<nf>::<metric_name>` output format and the RCA reasoning pattern.

### Experiment 5 — Time-Series Representation

The raw metric CSVs contain 60 rows × 20 columns.  Feeding the full table
verbatim may not be optimal for LLMs.  Compare:

- **Raw CSV** — paste the full CSV text
- **Summary statistics** — mean, std, min, max, last value per metric
- **Delta summary** — mean anomalous − mean normal per metric (the shift)
- **Top-N anomalous metrics** — only the N metrics with the highest effect
  score (e.g. N = 5, 10) extracted from `anomaly_evidence` in
  `rca_interpretation.json`

This also directly measures the token-count / accuracy trade-off.

### Experiment 6 — Fault-Type Difficulty Analysis

After running Experiments 1–5, analyse results broken down by fault type.
Some faults (e.g. `gnb_to_core_loss`, `upf_bandwidth_cap`) produce weaker or
noisier signals.  Investigate:

- Which context modalities help most for hard fault types
- Whether logs provide a qualitative signal absent from metrics for any class
- Whether the causal-graph narrative "rescues" cases where raw metrics are
  ambiguous

---

## Suggested Prompt Template

```
[SYSTEM]
You are an expert 5G network engineer performing root-cause analysis.
<insert network_context.txt here>

[USER]
A failure has been detected in the 5G network.  Analyse the evidence below
and identify the most likely root-cause metric(s).  Return your answer as
JSON in the format:
{
  "top_1": "<nf>::<metric_name>",
  "top_3": ["<nf>::<metric_name>", ...],
  "reasoning": "<brief explanation>"
}

## Baseline metrics (60 s before fault onset)
<insert normal.csv or summary>

## Fault-window metrics (60 s after fault onset)
<insert anomalous.csv or summary>

## KPI impact
<insert kpi_summary.json>

## Log events
<insert logs.jsonl>

## RCA algorithm output
<insert rca_interpretation.json narrative or full JSON>
```

Adapt the template for each experiment by including or excluding sections.

---

## Notes and Caveats

- **Ground truth is withheld from the episode folders.**  Store it separately
  and only use it for scoring after the model has produced its prediction.
- Metrics in `normal.csv` / `anomalous.csv` may contain near-constant columns
  for faults that do not affect a given subsystem.  This is expected and is
  itself a signal (absence of anomaly in an NF can help rule out root causes).
- Some episodes show elevated `cpu_throttle_rate` on the non-faulted NF due to
  residual effects from a previous round's CPU-stress fault.  This is a known
  source of difficulty and makes a good test of model robustness.
- `logs.jsonl` may be empty or sparse for some episodes if no NF emitted
  notable log events during the window.
- Response latency should be measured end-to-end including prompt tokenisation,
  not just model inference time, since token count varies significantly across
  configurations.
