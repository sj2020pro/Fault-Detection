# PFCP Session Modification Churn Fault (UE Context-Release Storm)

This folder documents a control-plane fault that can be injected after the 5G
core is already running and both slices are healthy. Unlike a raw packet flood,
this fault makes the control plane **repeatedly change real state inside the
UPF**: the SMF sends a steady stream of legitimate PFCP Session Modification
messages to `upf1`, each one re-programming the UPF's downlink forwarding rules.
The UE stays registered and its PDU session stays up the whole time.

In the local test this raised the `upf1` N4 RX packet rate from ~0 pps to
~3-5 pps and degraded slice-1 RTT/packet-loss, while slice 2 (`upf2`/UE2)
remained completely healthy.

## Fault Summary

| Property | Value |
| --- | --- |
| Target function | `open5gs-upf1` |
| Target interface | `n4` |
| Target protocol | PFCP over UDP (port `8805`) |
| PFCP message driven | Session Modification (UP deactivation / activation) |
| Trigger | repeated gNB `ue-release` for UE1 + continuous UE1 uplink |
| Fault scope | slice 1 only (the slice anchored on `upf1`) |
| Main KPI affected | slice-1 RTT, jitter, packet loss |
| Root-cause metric | `upf1` N4 RX packet rate (PFCP control-plane rate) |
| Session impact | PDU session preserved (same SEID, same UE IP) |

This fault does **not** require editing any config, restarting the UPF/SMF,
deleting the PDU session, or sending crafted/invalid packets. It is injected at
runtime on a healthy system and recovers when the injector stops.

## How It Works

PFCP is the control-plane protocol between the SMF and UPF on the N4 interface.
When a UE goes from CM-CONNECTED to CM-IDLE and back, the SMF must tell the UPF
about it:

1. **`ue-release` on the gNB** -> AN Release -> the UE goes **CM-IDLE**. The AMF
   asks the SMF to deactivate the user-plane connection, and the SMF sends a
   **PFCP Session Modification** to `upf1` that switches the downlink FAR's
   apply-action to drop/buffer and removes the gNB N3 tunnel info.
2. **Next uplink packet** (the continuous ping) triggers a **Service Request**
   -> the UE goes **CM-CONNECTED**. The SMF sends **another PFCP Session
   Modification** to `upf1` to restore the downlink FAR with the **new** gNB N3
   F-TEID.

Repeating this in a loop produces a continuous stream of PFCP Session
Modification requests that the UPF genuinely parses and applies (FAR
apply-action and outer-header-creation are rewritten every cycle). This control
work competes with user-plane forwarding on the same UPF, degrading the slice
KPI.

It was verified that this is modification, not teardown: over 10 controlled
cycles the UPF session add/remove count did not change (same F-SEID, same UE IP
`10.41.0.x`), while the N4 RX counter kept incrementing. The PDU session is
preserved throughout.

## Root-Cause Metric

One root-cause metric is used for this fault, consistent with the N4/PFCP RCA
framework:

| Metric | Description | Why It Is The Root Cause |
| --- | --- | --- |
| `upf1_n4_rx_packet_rate_pps` | PFCP packet receive rate on the `n4` interface of `open5gs-upf1`, in packets/second. | In a real-world control-plane configuration fault, the user-visible problem is that the UPF is being driven by an abnormal rate of legitimate PFCP control messages. This metric directly captures that control-plane load. When it rises above baseline, the UPF is spending receive/parse/apply effort on PFCP Session Modifications instead of forwarding, which is what degrades the slice KPI. |

```text
Root cause metric:    upf1_n4_rx_packet_rate_pps
Root cause function:  upf1
Root cause interface: n4
Fault type:           control-plane PFCP session-modification churn
Affected KPI:         slice-1 RTT / jitter / packet loss
```

## How To Inject The Fault (Exactly)

Use the three helper scripts in this folder. Make them executable once:

```bash
cd /home/oem/Documents/Waterloo/5G_Core/pfcp_session_mod_churn_fault
chmod +x inject_fault.sh workload.sh monitor.sh
```

All scripts resolve pods by label (`component=gnb`, `nf=amf`, `name=upf1`,
`name=ue1`/`name=ue2`) and resolve the gNB nr-cli node name dynamically, so they
keep working after pod restarts (pod names change, labels do not).

Use four terminals.

**Terminal 1 - root-cause metric on the affected UPF:**

```bash
./monitor.sh            # upf1 N4 RX packet rate, once per second
```

**Terminal 2 - victim slice traffic (also reactivates the UE):**

```bash
UE=ue1 COUNT=600 ./workload.sh
```

**Terminal 3 - control slice traffic (should stay healthy):**

```bash
UE=ue2 COUNT=600 ./workload.sh
```

**Terminal 4 - inject the fault:**

```bash
INTERVAL=0.5 DURATION=30 ./inject_fault.sh
```

Expected pattern:

- Before: `upf1` N4 RX ~0-1 pps; UE1 and UE2 both ~0% loss, ~8-10 ms RTT.
- During: `upf1` N4 RX ~3-5 pps; UE1 RTT/jitter up and packet loss rises;
  **UE2 stays at ~0% loss** (slice isolation).
- After (`inject_fault.sh` exits): `upf1` N4 RX back to baseline, UE1 recovers.

### Manual one-liner (no scripts)

The same selection logic inline (target UE1 = IMSI `...0000000001`, MSIN
`0000000001`). Requires a continuous UE1 ping running separately.

```bash
NS=open5gs
GNB=$(kubectl get pods -n $NS -l component=gnb -o jsonpath='{.items[0].metadata.name}')
AMF=$(kubectl get pods -n $NS -l nf=amf -o jsonpath='{.items[0].metadata.name}')
GNBID=$(kubectl exec -n $NS $GNB -c gnb -- /ueransim/nr-cli --dump | head -1)
SECONDS=0
while [ $SECONDS -lt 30 ]; do
  A1=$(kubectl logs -n $NS $AMF --tail=80 2>&1 | sed -E 's/\x1b\[[0-9;]*m//g' \
        | awk '/AMF_UE_NGAP_ID\[/{s=$0;sub(/.*AMF_UE_NGAP_ID\[/,"",s);sub(/\].*/,"",s);a=s;getline l;if(l ~ /0000000001/)u=a} END{print u}')
  UEID=$(kubectl exec -n $NS $GNB -c gnb -- /ueransim/nr-cli "$GNBID" --exec "ue-list" 2>/dev/null \
        | awk -v a="$A1" '/ue-id:/{uid=$NF} /amf-ngap-id:/{if($NF==a)print uid}')
  [ -n "$UEID" ] && kubectl exec -n $NS $GNB -c gnb -- /ueransim/nr-cli "$GNBID" --exec "ue-release $UEID" >/dev/null 2>&1
  sleep 0.5
done
```

### Measuring the root-cause metric manually

```bash
kubectl exec -n open5gs $(kubectl get pods -n open5gs -l name=upf1 -o jsonpath='{.items[0].metadata.name}') -c upf -- \
  sh -c 'a=$(cat /sys/class/net/n4/statistics/rx_packets); sleep 5; b=$(cat /sys/class/net/n4/statistics/rx_packets); echo $(( (b-a)/5 ))'
```

## Why Targeting The Right UE Matters (Multi-UE Selection)

This is the single most important implementation detail. With more than one UE
attached to the same gNB, you cannot simply "release the first UE in the list":

- The gNB `ue-list` shows `ue-id`, `ran-ngap-id`, and `amf-ngap-id`, but **not**
  the IMSI, so the entries are not self-identifying.
- All three of those ids are **reallocated** every time a UE reactivates from
  idle, so there is no stable id to hard-code. A hard-coded `ue-release 4`
  becomes a silent no-op after the first cycle, and "release the first entry"
  randomly hits the wrong UE.

The only stable identifier is the **IMSI**. The injector therefore re-derives
the target's current gNB `ue-id` every iteration:

1. The AMF log prints, per InitialUEMessage / UE Context Release, an
   `AMF_UE_NGAP_ID[..]` line immediately followed by a `SUCI[suci-...-<MSIN>]`
   line. Parse the AMF log to get the target IMSI's current `amf-ngap-id`.
2. Match that `amf-ngap-id` against the gNB `ue-list` to get the current
   `ue-id`, then release it.

This is **safe by construction**: if the derived id is briefly stale (target UE
mid-reactivation), it matches no `ue-list` entry and the release is skipped -
other UEs are never released by accident. This is why UE2 stays healthy.

## Concerns And Notes

- **Continuous uplink on the target UE is mandatory.** The churn comes from
  idle->active->idle transitions. With no UE traffic, the UE stays idle after
  the first release and nothing churns. Always run `workload.sh` for the target
  UE alongside `inject_fault.sh`.

- **Keep the churn rate- and time-bounded.** Sustained, very aggressive churn
  (tight loop, no sleep) can wedge the RAN into a stuck state where UEs remain
  CM-IDLE and stop reactivating (pings then see 100% loss even though the PDU
  session shows PS-ACTIVE). The validated, well-behaved setting is a release
  every ~0.5 s for ~30 s. Do not remove the `INTERVAL` sleep.

- **Recovery from a wedged RAN.** If UEs get stuck idle and will not reactivate,
  restart the RAN (this does not touch the core):

  ```bash
  kubectl rollout restart deploy/ueransim-gnb -n open5gs
  kubectl rollout status  deploy/ueransim-gnb -n open5gs
  sleep 8
  kubectl rollout restart deploy/ueransim-ue1 deploy/ueransim-ue2 -n open5gs
  kubectl rollout status  deploy/ueransim-ue1 -n open5gs
  kubectl rollout status  deploy/ueransim-ue2 -n open5gs
  ```

  Then re-verify both UEs ping with 0% loss before injecting again.

- **The metric magnitude is modest but unambiguous.** Each idle/active cycle is
  only a few PFCP packets, so the N4 RX rate is ~3-5 pps during the fault, not
  thousands. That is expected: this is real per-session control signaling, not a
  flood. The signal is the rise above the ~0 pps idle baseline and its strict
  localization to `upf1`.

- **Host awk is `mawk`.** The parsing avoids the 3-argument `match()` form
  (gawk-only) and uses `sub()`/`getline` instead. Keep that if you edit the
  scripts.

- **Container names matter in `kubectl exec`.** UE pods default to multiple
  containers; the scripts pass `-c ue` (UE), `-c gnb` (gNB), `-c upf` (UPF).

- **Pod names are not stable.** Everything is resolved by label, and the gNB
  nr-cli node name is read via `nr-cli --dump`. Do not hard-code pod names.

- **Slice mapping.** UE1 (IMSI `...0000000001`) is on slice 1 -> `upf1`
  (subnet `10.41.0.0/16`). UE2 (IMSI `...0000000002`) is on slice 2 -> `upf2`
  (subnet `10.42.0.0/16`). To target slice 2 instead, set
  `TARGET_IMSI=001010000000002` and monitor `UPF=upf2`.

## Observed Local Result

Both UEs running continuous traffic; churn applied to UE1 only for ~30 s
(release every 0.5 s, 27 releases, 0 mis-selects):

| Stream | Packet Loss | RTT Avg | RTT Max |
| --- | ---: | ---: | ---: |
| UE1 (victim, slice 1 / upf1) | `7.3%` | `11.1 ms` | `39.6 ms` |
| UE2 (control, slice 2 / upf2) | `0%` | `9.7 ms` | `34.1 ms` |

`upf1` N4 RX packet rate during the fault: ~3-5 pps (idle baseline ~0-1 pps).
`upf2` N4 RX packet rate: unchanged. PDU sessions preserved throughout.

## Real-World Scenarios Where This Behavior Happens

This injected behavior reproduces real classes of control-plane faults seen in
deployed 5G cores, where the common signature is an abnormal rate of legitimate
PFCP Session Modification traffic to one UPF.

### 1. Misconfigured RAN inactivity / UE-inactivity timer

If the gNB (or AMF) inactivity timer is set far too low, UEs are released to
CM-IDLE almost immediately after each small burst of traffic, then a Service
Request brings them back. Bursty/keep-alive traffic then produces a continuous
idle<->active oscillation, exactly like this fault. The UPF sees constant PFCP
Session Modification churn. This is a pure **configuration** fault upstream; the
N4 modification rate is its UPF-side signature.

### 2. Misconfigured / flapping N2 (NGAP) or paging policy

A bad AMF policy, an N2 link that flaps, or an over-aggressive
connection-release policy can repeatedly tear down and re-activate the UE's
RAN connection. Each transition triggers UP deactivation/activation and the
corresponding PFCP modifications on the anchoring UPF.

### 3. Buggy SMF UP-activation state machine

An SMF defect or race that repeatedly re-issues UP deactivation/activation (for
example, mishandling a Service Request or a partial modification ACK) produces
the same Session Modification storm toward one UPF without ever dropping the
session.

### 4. Control-plane automation / orchestration loop

An external controller, policy engine, or test harness that repeatedly drives
UE state transitions or session updates for one slice creates a sustained
modification churn. Each individual action is valid; the loop is the fault.

### 5. Handover / mobility thrash

A UE oscillating between cells, or a mobility-management misconfiguration, causes
repeated path-switch / UP-activation updates. The SMF re-programs the UPF's
downlink F-TEID on every transition, which on the N4 side looks identical to this
fault.

In all of these, the UPF stays up and the PDU session is preserved, but the
slice anchored on the affected UPF degrades because the UPF is busy applying a
stream of legitimate control-plane changes.

## RCA Interpretation

For this fault the RCA framework should identify:

```text
Root cause metric:    upf1_n4_rx_packet_rate_pps
Root cause function:  upf1
Root cause interface: n4
Fault type:           control-plane PFCP session-modification churn
Affected KPI:         slice-1 RTT / jitter / packet-loss increase
```

Causal chain:

```text
Abnormal UE context-release / Service-Request churn for slice 1
  (real-world: a control-plane configuration fault, e.g. RAN inactivity timer)
-> SMF emits a steady stream of PFCP Session Modification requests to upf1
-> high upf1 N4 RX packet rate  (root-cause metric)
-> UPF spends receive/parse/apply effort re-programming downlink FAR/F-TEID
-> less stable user-plane forwarding for slice 1
-> slice-1 RTT / jitter / packet-loss degradation
   (slice 2 on upf2 is unaffected)
```

## Cleanup

The injector is time-bounded (`DURATION`) and exits on its own; it creates no
Kubernetes objects. After it stops, the UE stops being released, stays
CM-CONNECTED, and the slice recovers. Verify recovery:

```bash
UE=ue1 COUNT=40 ./workload.sh     # expect ~0% loss, baseline RTT
```

If the RAN was wedged by over-aggressive churn, use the rollout-restart recovery
in the Concerns And Notes section.
