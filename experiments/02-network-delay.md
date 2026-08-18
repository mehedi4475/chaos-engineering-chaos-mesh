# Experiment 02 — Network Delay (NetworkChaos)

> Status: **Attempted → root-caused a driver-level limitation** (not a clean pass).
> This is documented honestly because diagnosing *why* a fault could not be
> injected is itself a core reliability-engineering skill.

## Steady State (before)
- podinfo 3/3 Ready, loadgen ~5 req/s
- Grafana p99 latency baseline: ~5 ms (essentially flat)

## Blast Radius
- Target: namespace=demo, app=podinfo, ALL matching pods
- Fault: inject network delay on all podinfo pods for a fixed duration

## Hypothesis
- Pods will STAY Running (Kubernetes sees no failure — a "gray failure").
- Request p99 latency will spike sharply (by roughly the injected delay).
- loadgen throughput (req/s) will DROP, because each request now takes much
  longer to complete (fewer requests finish per second).
- After the duration elapses, the delay auto-clears and latency returns to baseline.

## Actual Result
- **Did pods stay Running?** Yes — all 3 podinfo pods stayed `1/1 Running`.
- **Latency during chaos?** No change. p99 stayed at ~5 ms (see
  `27-netdelay-latency-flat.png`). The delay was **never actually applied**.
- **Root cause:** The `NetworkChaos` object got stuck and the chaos-daemon logged:
  ```
  Failed to apply chaos: unable to flush ip sets for pod demo/podinfo-...
  ```
  On minikube's **docker driver**, `NetworkChaos` needs `ipset`/`tc`
  (traffic-control) operations inside the pod network namespace. The
  container-in-container docker driver does not expose the kernel-level network
  capabilities these require, so the injection fails. `PodChaos` (Experiment 01)
  worked because killing a process needs no network manipulation.
- **Secondary finding:** The stuck object could not be deleted normally because a
  Chaos Mesh **finalizer** waited for a recovery step that never succeeded. It was
  released with:
  ```
  kubectl patch networkchaos network-delay-podinfo -n demo \
    -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```

## Conclusion / Learnings
- **Tooling has an environment envelope.** A chaos experiment can silently
  "run" while injecting nothing. Always confirm the fault actually landed
  (check daemon events / the metric you expect to move) before trusting a result.
- **Correct fix for a real run:** use a driver with a fuller network stack —
  minikube with a VM driver (Hyper-V / VirtualBox) or `kind` — where
  `ipset`/`tc` are available. This is the planned follow-up.
- **Finalizers can trap resources.** When a controller can't finish cleanup, the
  object hangs in "Terminating"; clearing the finalizer force-releases it.

## How to reproduce a working run (next iteration)
```bash
# Recreate the cluster on a VM-based driver instead of docker
minikube delete -p chaos-mesh
minikube start -p chaos-mesh --cpus=4 --memory=6144 --driver=hyperv
# ...reinstall monitoring + podinfo + chaos-mesh, then:
kubectl apply -f experiments/02-network-delay.yaml
# p99 latency in Grafana should jump to ~the injected delay, then recover.
```
