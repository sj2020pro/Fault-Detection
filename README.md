# N4/PFCP RX Packet-Rate Storm Fault

This folder documents a control-plane traffic fault that can be injected after the 5G core is already running and UE traffic is healthy. The fault creates constant N4/PFCP traffic toward `upf1`, causing the UPF to receive a very high packet rate on its N4 interface. In the local test, this increased slice RTT without crashing the UPF.

## Fault Summary

The fault is an N4/PFCP control-plane packet storm from the SMF-side network to `upf1`.

- Target function: `open5gs-upf1`
- Target interface: `n4`
- Target protocol: PFCP over UDP
- Target port: `8805`
- Fault scope: slice-level when the affected slice is anchored on `upf1`
- Main KPI affected: throughput and/or RTT for the slice using `upf1`
- Root-cause metric: `upf1` N4 RX packet rate

This fault does not require restarting the UE, restarting the PDU session, or redeploying the UPF. It can be injected while traffic is already flowing.

## Why This Can Degrade Slice Performance

PFCP is the control-plane protocol used between the SMF and UPF on the N4 interface. A high-rate PFCP storm forces `upf1` to continuously receive and process control-plane packets. Even if the packets are invalid or ignored, the UPF still spends CPU time, socket receive-buffer capacity, interrupt/softirq processing, and logging/parser work on the N4 traffic.

Because the same UPF process and host resources are also responsible for user-plane forwarding, excessive N4 RX traffic can interfere with user-plane packet handling. The expected user-visible symptom is increased RTT and, under heavier load, reduced throughput for the slice anchored on that UPF.

## Root-Cause Metric

Only one root-cause metric is used for this fault:

| Metric | Description | Why It Is The Root Cause |
| --- | --- | --- |
| `upf1_n4_rx_packet_rate_pps` | Packet receive rate on the `n4` interface of the `open5gs-upf1` pod, measured in packets per second. | The injected fault is directly an abnormal N4/PFCP packet storm. When this metric spikes, the UPF is receiving excessive control-plane packets. The KPI degradation is caused by that control-plane RX load competing with normal UPF forwarding work. |

## How To Measure The Root-Cause Metric

Inside the `open5gs-upf1` pod, the Linux interface counters expose cumulative RX packet counts for each interface.

Check the current N4 packet counter:

```bash
kubectl exec -n open5gs open5gs-upf1-7d98bf9f8d-pgdwc -- sh -c "cat /sys/class/net/n4/statistics/rx_packets"
```

Measure RX packet rate over a 5-second window:

```bash
kubectl exec -n open5gs open5gs-upf1-7d98bf9f8d-pgdwc -- sh -c 'a=$(cat /sys/class/net/n4/statistics/rx_packets); sleep 5; b=$(cat /sys/class/net/n4/statistics/rx_packets); echo $(( (b-a)/5 ))'
```

The output is the approximate N4 RX packet rate in packets per second.

You can also inspect the full interface counters:

```bash
kubectl exec -n open5gs open5gs-upf1-7d98bf9f8d-pgdwc -- ip -s link show n4
```

During the tested fault run, `upf1` received about `3.76M` packets on `n4` in 30 seconds, which is roughly `125k packets/s`.

## Prerequisites

Before injecting the fault, confirm that the 5G core and UEs are healthy.

Check pods:

```bash
kubectl get pods -n open5gs -o wide
```

Expected relevant pods:

- `open5gs-smf1`
- `open5gs-upf1`
- `ueransim-ue1`
- `ueransim-ue2`

Confirm UE traffic works before injecting the fault:

```bash
kubectl exec -n open5gs ueransim-ue1-5c4db98cfb-5kmks -- ping -I uesimtun0 -c 80 -i 0.05 -s 1200 8.8.8.8
```

Optional comparison slice:

```bash
kubectl exec -n open5gs ueransim-ue2-674df7f946-44lst -- ping -I uesimtun0 -c 80 -i 0.05 -s 1200 8.8.8.8
```

## Fault Injection Command

Run the following command from the host. It executes inside the `smf1` pod and sends PFCP-like UDP packets to `upf1` on N4 port `8805` for 30 seconds.

```bash
kubectl exec -n open5gs open5gs-smf1-7fb9848485-r5w46 -- /usr/bin/perl -MIO::Socket::INET -e '$s=IO::Socket::INET->new(PeerAddr=>"10.10.4.1",PeerPort=>8805,Proto=>"udp") or die $!; $p="\x21\x34\x00\x23\x00\x00\x00\x00\x00\x00\x09\x23\x01\x23\x46\x00\x00\x0a\x00\x13\x00\x6c\x00\x04\x00\x00\x00\x02\x00\x2c\x00\x02\x0c\x00\x00\x58\x00\x01\x01"; $end=time()+30; $n=0; while(time()<$end){ for(1..1000){$s->send($p); $n++} select undef,undef,undef,0.001 } print "sent=$n\n";'
```

The command prints the number of packets sent when it finishes.

In the local test, the output was:

```text
sent=3765000
```

## How To Validate The Effect

Use three terminals.

Terminal 1: monitor the root-cause metric before and during the fault.

```bash
kubectl exec -n open5gs open5gs-upf1-7d98bf9f8d-pgdwc -- sh -c 'while true; do ts=$(date +%s); a=$(cat /sys/class/net/n4/statistics/rx_packets); sleep 1; b=$(cat /sys/class/net/n4/statistics/rx_packets); echo "$ts $((b-a))"; done'
```

Terminal 2: run UE traffic for the slice anchored on `upf1`.

```bash
kubectl exec -n open5gs ueransim-ue1-5c4db98cfb-5kmks -- ping -I uesimtun0 -c 300 -i 0.05 -s 1200 8.8.8.8
```

Terminal 3: inject the fault.

```bash
kubectl exec -n open5gs open5gs-smf1-7fb9848485-r5w46 -- /usr/bin/perl -MIO::Socket::INET -e '$s=IO::Socket::INET->new(PeerAddr=>"10.10.4.1",PeerPort=>8805,Proto=>"udp") or die $!; $p="\x21\x34\x00\x23\x00\x00\x00\x00\x00\x00\x09\x23\x01\x23\x46\x00\x00\x0a\x00\x13\x00\x6c\x00\x04\x00\x00\x00\x02\x00\x2c\x00\x02\x0c\x00\x00\x58\x00\x01\x01"; $end=time()+30; $n=0; while(time()<$end){ for(1..1000){$s->send($p); $n++} select undef,undef,undef,0.001 } print "sent=$n\n";'
```

Expected validation pattern:

- Before fault: `upf1_n4_rx_packet_rate_pps` is low.
- During fault: `upf1_n4_rx_packet_rate_pps` spikes sharply.
- During fault: UE1 RTT increases, or throughput decreases if using a throughput generator.
- After fault: `upf1_n4_rx_packet_rate_pps` returns near baseline.
- After fault: UE1 RTT/throughput recovers.

## Observed Local Result

The following result was observed in the local `open5gs` namespace:

| Phase | UE1 Packet Loss | UE1 RTT Avg |
| --- | ---: | ---: |
| Before fault | `0%` | `16.166 ms` |
| During N4/PFCP storm | `0%` | `31.562 ms` |
| After fault | `0%` | `12.117 ms` |

The affected KPI was RTT. This is still useful for the RCA framework because RTT increase is a direct performance degradation and can also reduce measured throughput for TCP-like traffic.

## Real-World Fault Scenarios

This injected behavior represents real classes of control-plane faults that can happen in deployed 5G cores.

### 1. Buggy SMF PFCP Retry Loop

An SMF bug or race condition repeatedly retries PFCP session modification, deletion, association, or heartbeat messages toward the same UPF. This can happen if the SMF incorrectly believes the UPF did not acknowledge a previous request, or if transaction-state cleanup fails.

Expected symptom:

- High `upf1_n4_rx_packet_rate_pps`
- Increased UPF CPU and parser work
- Higher RTT or lower throughput for slices anchored on that UPF

### 2. SMF-UPF Version Or Configuration Mismatch

If SMF and UPF versions, PFCP feature support, enterprise IEs, or configuration expectations do not match, the SMF may repeatedly send messages that the UPF rejects or cannot parse. The control plane remains active, but the N4 interface becomes noisy.

Expected symptom:

- Repeated PFCP messages on N4
- Possible UPF PFCP parsing or rejection logs
- Slice performance degradation without a full PDU session restart

### 3. Misconfigured Slice Or UPF Selection Policy

A slice policy error can cause too many control-plane operations to target one UPF. For example, the SMF may keep re-evaluating the UPF or repeatedly applying policy updates for a slice that is mapped to `upf1`.

Expected symptom:

- N4 RX packet-rate spike on only the UPF serving that slice
- Other slices on other UPFs are less affected
- The degraded slice shows RTT or throughput impact

### 4. Control-Plane Automation Or Orchestrator Loop

An external controller, automation job, or orchestration script may repeatedly trigger policy/session updates through the control plane. Even if every individual action is small, a tight loop can create a persistent N4 storm.

Expected symptom:

- Periodic or sustained `upf1_n4_rx_packet_rate_pps` spikes
- KPI degradation aligned with controller activity
- Recovery when the automation loop stops

### 5. Internal Control-Plane Abuse Or Compromise

If an internal pod or compromised control-plane component can reach the N4 network, it may intentionally or accidentally flood the UPF PFCP port. This does not require public exposure of the UPF.

Expected symptom:

- Very high N4 RX packet rate
- UPF remains up but data-plane service quality degrades
- Fault is localized to UPFs reachable from the noisy source

## RCA Interpretation

For this fault, the RCA framework should identify:

```text
Root cause metric: upf1_n4_rx_packet_rate_pps
Root cause function: upf1
Root cause interface: n4
Fault type: control-plane N4/PFCP packet-rate storm
Affected KPI: slice throughput degradation and/or RTT increase
```

The causal chain is:

```text
Excessive PFCP/N4 traffic
-> high upf1 N4 RX packet rate
-> UPF control-plane receive/parser load
-> less stable user-plane forwarding latency
-> slice RTT increase or throughput degradation
```

## Cleanup

The injection command is time bounded and exits automatically after 30 seconds. No Kubernetes object is created, and no cleanup command is normally required.

After the fault ends, verify recovery:

```bash
kubectl exec -n open5gs ueransim-ue1-5c4db98cfb-5kmks -- ping -I uesimtun0 -c 40 -i 0.05 -s 1200 8.8.8.8
```

Also verify that UPF is still healthy:

```bash
kubectl get pods -n open5gs -o wide
```
