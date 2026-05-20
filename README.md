# xNet Oceanus cluster access

This folder is set up for using this laptop as the client machine for the
shared xNet Kubernetes cluster. You do not need a separate Proxmox VM for this
workflow.

## Files

- `ssjohari-xnet-mercury.yaml`: kubeconfig for the `ssjohari` namespace on
  `xnet-mercury`. Keep this private because it contains an access token.
- `oceanus-tunnel.sh`: opens the SSH tunnel that makes the cluster API
  available on this laptop at `https://127.0.0.1:6443`.
- `kubectl-xnet.sh`: runs `kubectl` with the repo kubeconfig.

## SSH config

Add this to `~/.ssh/config`:

```sshconfig
Host oceanus
    HostName oceanus.cs.uwaterloo.ca
    User ssjohari
    LocalForward 8006 atlas601:8006
```

The `8006` forward is for the Proxmox UI at `http://127.0.0.1:8006`. It is not
needed for Kubernetes experiments, but it matches the Oceanus getting-started
guide.

The helper script can also connect without this alias because it defaults to
`ssjohari@oceanus.cs.uwaterloo.ca`.

## Connect to the Kubernetes cluster

In one terminal, start the tunnel:

```bash
./oceanus-tunnel.sh
```

Then, in another terminal, verify cluster access:

```bash
./kubectl-xnet.sh config current-context
./kubectl-xnet.sh get pods
```

All commands use the `ssjohari` namespace by default because that namespace is
set in the kubeconfig.

## If the tunnel target is different

The kubeconfig points at `127.0.0.1:6443`, so the SSH tunnel forwards local port
`6443` to the Kubernetes API server from inside the Oceanus network. This script
defaults to `192.168.126.50:6443`, matching the xNet cluster guide.

If the cluster admin gave you a different API host, run:

```bash
XNET_API_TARGET=actual-hostname:6443 ./oceanus-tunnel.sh
```

or edit `XNET_API_TARGET` in the script.

## Running experiments

Use the wrapper for normal Kubernetes operations:

```bash
./kubectl-xnet.sh apply -f experiment.yaml
./kubectl-xnet.sh get pods
./kubectl-xnet.sh logs job/my-job
./kubectl-xnet.sh delete -f experiment.yaml
```

Avoid deploying cluster-wide resources unless the admin has explicitly granted
that permission. Namespace-scoped resources such as Pods, Jobs, Deployments,
Services, ConfigMaps, and Secrets are the expected path.

## Installing Helm charts

Use the existing namespace permissions and service account when installing
charts:

```bash
helm install <release> <chart> \
  --namespace ssjohari \
  --kubeconfig ./ssjohari-xnet-mercury.yaml \
  --set rbac.create=false \
  --set serviceAccount.create=false
```

Those two `--set` values are important for namespace-scoped cluster access:
the chart should not try to create its own RBAC resources or service account.

## RCA offline dataset

The RCA client scripts are in this folder:

- `xnet-rca-export.sh`: creates an offline labeled dataset from past RCA sweep
  windows.
- `xnet-rca-live.py`: polls the live snapshot API.

Create a 1-hour offline dataset:

```bash
./oceanus-tunnel.sh
```

Then in another terminal:

```bash
KUBECONFIG=/home/oem/Documents/Xnet/ssjohari-xnet-mercury.yaml \
  ./xnet-rca-export.sh -n ssjohari -w 1h -o ./rca-offline-out
```

This produces one directory per labeled scenario plus `manifest.jsonl`. The
first export here produced 20 scenario windows under `./rca-offline-out`.

To include pod logs:

```bash
KUBECONFIG=/home/oem/Documents/Xnet/ssjohari-xnet-mercury.yaml \
  ./xnet-rca-export.sh -n ssjohari -w 1h -o ./rca-offline-out-with-logs -- \
  --include-logs --log-limit 5000
```

## Manual RCA faults

`manual_rca.py` adds a manual mode on top of the existing xNet RCA image:

- pause the automated random sweep,
- run the same baseline eMBB + URLLC UE workload continuously with no faults,
- trigger one chosen fault scenario on demand using the same Chaos Mesh recipes,
- resume baseline afterwards.

Start manual mode:

```bash
./manual_rca.py pause-sweep
./manual_rca.py start-baseline
./manual_rca.py status
```

Trigger one fault scenario:

```bash
./manual_rca.py trigger cpu_stress --target-class smf --duration 60
./manual_rca.py trigger gnb_to_core_loss --duration 60
./manual_rca.py trigger gnb_to_core_delay --duration 60
./manual_rca.py trigger gnb_to_core_partition --duration 60
./manual_rca.py trigger upf_bandwidth_cap --duration 60
```

During a trigger, the script pauses the manual baseline deployment, runs one
scenario with the same default workload and the requested fault, then resumes
the baseline deployment. The one-shot fault pod is labeled so the existing
`xnet-rca-sweep` `VMPodScrape` can scrape `xnet_fault_active` while the fault
is active.

Return to the automated sweep:

```bash
./manual_rca.py stop-baseline
./manual_rca.py resume-sweep
```

## RCA FastAPI app

The interactive manual RCA app lives in `RCA_APP/`.

```bash
cd /home/oem/Documents/Xnet
./oceanus-tunnel.sh
```

Then:

```bash
cd /home/oem/Documents/Xnet/RCA_APP
./run.sh
```

Open `http://127.0.0.1:8088`. The app plots live throughput KPI, can start
manual baseline mode, inject a selected 60-second fault, mark injection and
detection times, and plot a root-cause metric from 30 seconds before injection
through the collected post-injection window.
