# Per-Slice RCA Framework for Throughput KPI Degradation

This directory contains a Chaos Mesh based root-cause-analysis experiment framework for the two Open5GS slices deployed in this testbed:

- Slice `1-000001`: `ueransim-ue1` -> `open5gs-upf1`
- Slice `2-000002`: `ueransim-ue2` -> `open5gs-upf2`

The main KPI is always delivered slice throughput. The runner measures it as delivered ICMP payload throughput over the UE tunnel (`uesimtun0`) and also records Monarch/Prometheus UPF interface counters for the same slice UPF. Faults are injected into the slice-specific UPF with Chaos Mesh.

## Tooling

Chaos Mesh was installed with Helm:

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update chaos-mesh
helm install chaos-mesh chaos-mesh/chaos-mesh   -n chaos-mesh --create-namespace   --set chaosDaemon.runtime=containerd   --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

References used:

- Chaos Mesh StressChaos supports container CPU stress with `workers` and `load`: https://chaos-mesh.org/docs/2.6.7/simulate-heavy-stress-on-kubernetes/
- Chaos Mesh NetworkChaos supports `delay`, `loss`, `bandwidth`, and the `device` field: https://chaos-mesh.dev/reference/master/

## Files

- `run_rca_experiments.py`: runs baseline, injects a fault, collects metrics, removes the fault, checks recovery, and saves CSV/plot/summary files.
- `chaos/upf-cpu-stress.yaml`: `StressChaos` for UPF CPU contention.
- `chaos/upf-packet-loss.yaml`: `NetworkChaos` packet loss on UPF `ogstun`.
- `chaos/upf-bandwidth-cap.yaml`: `NetworkChaos` bandwidth cap on UPF `ogstun`.
- `chaos/upf-network-delay.yaml`: `NetworkChaos` delay/jitter on UPF `ogstun`.
- `plots/`: KPI plots for each accepted scenario.
- `results/`: CSV time series and JSON summaries.

## How to Run

Run all scenarios for one slice:

```bash
cd /home/oem/data-pipeline/rca_framework
/usr/bin/python3 ./run_rca_experiments.py --slice 1-000001 --all
/usr/bin/python3 ./run_rca_experiments.py --slice 2-000002 --all
```

Run one scenario:

```bash
/usr/bin/python3 ./run_rca_experiments.py --slice 1-000001 --scenario upf_cpu_stress
```

The runner rejects a scenario if the fault-window KPI drop is below 20%.

## Fault Scenarios and Root-Cause Metrics

| Scenario | Chaos Mesh kind/action | Target | Expected KPI effect | RCA root-cause metric |
|---|---|---|---|---|
| `upf_cpu_stress` | `StressChaos`, CPU workers=12, load=95 | slice UPF container `upf` | Throughput drops because UPF packet processing is CPU-contended while sessions stay alive. | `upf_cpu_utilization_cores` |
| `upf_packet_loss` | `NetworkChaos`, `loss=50`, `correlation=25` | slice UPF `ogstun` | Throughput drops from packet loss, but some packets still pass. | `upf_ogstun_packet_loss_pct` |
| `upf_bandwidth_cap` | `NetworkChaos`, `bandwidth rate=40kbps` | slice UPF `ogstun` | Throughput is capped and queues add latency, but the slice does not go fully down. | `upf_ogstun_bandwidth_cap_bps` |
| `upf_network_delay` | `NetworkChaos`, `latency=180ms`, `jitter=30ms` | slice UPF `ogstun` | Delivered throughput drops because per-sample completion time increases; packets still pass. | `upf_ogstun_added_latency_ms` |

For loss, bandwidth, and delay, the root-cause metric is the low-level UPF `ogstun` impairment configured and observed through the active Chaos Mesh object. cAdvisor in this cluster does not expose Linux qdisc drop/latency counters directly, so the runner also records delivered loss percentage and RTT from the UE probe as confirmation signals.

## Metrics Monitored

The framework is per-slice: it maps the slice to the selected UPF pod, then evaluates these metrics for that UPF and its related slice state. Prometheus is served by Monarch at `http://10.0.0.5:30095`.

| Metric name | PromQL or source | Description |
|---|---|---|
| `slice_delivered_throughput_bps` | runner UE probe | Main KPI: delivered payload bits per second through `uesimtun0`. |
| `slice_icmp_loss_pct` | runner UE probe | Probe packet loss during each KPI sample. |
| `slice_icmp_avg_rtt_ms` | runner UE probe | Probe RTT; useful for delay and CPU contention. |
| `upf_cpu_utilization_cores` | `sum(rate(container_cpu_usage_seconds_total{namespace="open5gs",pod=~"<upf-pod>",container!=""}[1m]))` | UPF CPU usage; root for CPU stress. |
| `upf_memory_working_set_bytes` | `sum(container_memory_working_set_bytes{namespace="open5gs",pod=~"<upf-pod>",container!=""})` | UPF memory pressure. |
| `upf_ogstun_rx_bps` | `sum(rate(container_network_receive_bytes_total{namespace="open5gs",pod=~"<upf-pod>",interface="ogstun"}[1m])) * 8` | Bytes entering the UPF UE tunnel interface. |
| `upf_ogstun_tx_bps` | `sum(rate(container_network_transmit_bytes_total{namespace="open5gs",pod=~"<upf-pod>",interface="ogstun"}[1m])) * 8` | Bytes leaving the UPF UE tunnel interface. |
| `upf_ogstun_rx_drop_rate` | `sum(rate(container_network_receive_packets_dropped_total{namespace="open5gs",pod=~"<upf-pod>",interface="ogstun"}[1m]))` | cAdvisor receive drops on `ogstun`. |
| `upf_ogstun_tx_drop_rate` | `sum(rate(container_network_transmit_packets_dropped_total{namespace="open5gs",pod=~"<upf-pod>",interface="ogstun"}[1m]))` | cAdvisor transmit drops on `ogstun`. |
| `upf_n3_rx_bps` | `sum(rate(container_network_receive_bytes_total{namespace="open5gs",pod=~"<upf-pod>",interface="n3"}[1m])) * 8` | N3 ingress traffic at the UPF. |
| `upf_n3_tx_bps` | `sum(rate(container_network_transmit_bytes_total{namespace="open5gs",pod=~"<upf-pod>",interface="n3"}[1m])) * 8` | N3 egress traffic at the UPF. |
| `upf_eth0_rx_bps` | same receive query with `interface="eth0"` | UPF Kubernetes pod-network ingress. |
| `upf_eth0_tx_bps` | same transmit query with `interface="eth0"` | UPF Kubernetes pod-network egress. |
| `upf_ogstun_packet_loss_pct` | Chaos Mesh object + UE probe confirmation | Configured packet-loss impairment on UPF `ogstun`; root for packet-loss fault. |
| `upf_ogstun_bandwidth_cap_bps` | Chaos Mesh object | Configured kernel traffic-control cap on UPF `ogstun`; root for bandwidth-cap fault. |
| `upf_ogstun_added_latency_ms` | Chaos Mesh object + UE RTT confirmation | Configured netem delay on UPF `ogstun`; root for delay fault. |
| `fivegs_upffunction_upf_sessionnbr` | `fivegs_upffunction_upf_sessionnbr` | UPF session count; confirms the slice is degraded, not down. |
| `fivegs_smffunction_sm_sessionnbr` | `fivegs_smffunction_sm_sessionnbr` | SMF session count. |
| `ues_active` | `ues_active` | Active UE count reported by SMF metrics. |
| `ran_ue` | `ran_ue` | RAN UE count reported by AMF metrics. |
| `fivegs_amffunction_rm_registeredsubnbr` | `fivegs_amffunction_rm_registeredsubnbr` | Registered subscriber count. |
| `fivegs_smffunction_sm_qos_flow_nbr` | `fivegs_smffunction_sm_qos_flow_nbr` | SMF QoS flow count. |

## RCA Logic

For each slice and scenario:

1. Establish a baseline KPI window.
2. Inject exactly one fault against the slice UPF.
3. Continue sampling the main KPI and the monitored metrics during the fault.
4. Remove the fault and sample recovery.
5. Accept the scenario only if average KPI drops by at least 20% during the fault window and recovers afterward.
6. Pinpoint the root cause as the low-level metric that both changes in the fault window and matches the injected fault family.

This gives supervised RCA data: every row in `results/*.csv` has the KPI and low-level metrics, and every `*_summary.json` records the known root metric.

## Verified Results

| Slice | Scenario | Baseline avg bps | Fault avg bps | Recovery avg bps | Drop |
|---|---:|---:|---:|---:|---:|
| `1-000001` | `upf_cpu_stress` | 859433.8 | 461739.2 | 854251.3 | 46.3% |
| `1-000001` | `upf_packet_loss` | 860167.7 | 330700.0 | 834011.8 | 61.6% |
| `1-000001` | `upf_bandwidth_cap` | 717426.7 | 330807.6 | 764965.1 | 53.9% |
| `1-000001` | `upf_network_delay` | 851017.9 | 458517.5 | 832730.4 | 46.1% |
| `2-000002` | `upf_cpu_stress` | 845769.1 | 472344.6 | 761704.6 | 44.2% |
| `2-000002` | `upf_packet_loss` | 616022.4 | 285403.0 | 748791.2 | 53.7% |
| `2-000002` | `upf_bandwidth_cap` | 695509.9 | 331949.0 | 717261.7 | 52.3% |
| `2-000002` | `upf_network_delay` | 678961.0 | 459673.7 | 712452.5 | 32.3% |

All scenarios preserve pod/session availability and avoid a full slice outage.

## Plot Locations

Plots are saved in:

```text
/home/oem/data-pipeline/rca_framework/plots/
```

Generated plot files:

```text
/home/oem/data-pipeline/rca_framework/plots/1-000001_upf_cpu_stress.png
/home/oem/data-pipeline/rca_framework/plots/1-000001_upf_packet_loss.png
/home/oem/data-pipeline/rca_framework/plots/1-000001_upf_bandwidth_cap.png
/home/oem/data-pipeline/rca_framework/plots/1-000001_upf_network_delay.png
/home/oem/data-pipeline/rca_framework/plots/2-000002_upf_cpu_stress.png
/home/oem/data-pipeline/rca_framework/plots/2-000002_upf_packet_loss.png
/home/oem/data-pipeline/rca_framework/plots/2-000002_upf_bandwidth_cap.png
/home/oem/data-pipeline/rca_framework/plots/2-000002_upf_network_delay.png
```

## Safety / Cleanup

Check that no fault remains active:

```bash
kubectl get stresschaos,networkchaos -n open5gs
```

Delete any RCA fault manually if needed:

```bash
kubectl delete stresschaos,networkchaos -n open5gs --all
```
