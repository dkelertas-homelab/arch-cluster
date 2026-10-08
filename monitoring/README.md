# Monitoring: yogi host metrics

I wanted durable CPU, RAM, disk and network history for **yogi**, my Arch Linux machine that also runs this single-node k3s cluster, plus a dashboard to look at it. This folder is how I get that, managed by Flux like everything else in the repo.

## The stack

| Piece | Where it's defined | What it does |
| --- | --- | --- |
| [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 87.x | `controllers/base/kube-prometheus-stack/release.yaml` (Flux `HelmRelease`) | Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, node-exporter |
| [node_exporter](https://github.com/prometheus/node_exporter) | the chart's DaemonSet | Reads yogi's `/proc` and `/sys` (CPU, memory, filesystems, disks, network, hwmon temps, battery) and serves them on `:9100` |
| Prometheus | `prometheus.prometheusSpec` in the HelmRelease | Scrapes everything every 30s and stores it on an 8Gi `local-path` PVC |
| Grafana | `grafana` values + `configs/staging/kube-prometheus-stack/` | Prometheus datasource, dashboards loaded from ConfigMaps by the sidecar |
| Secrets | `configs/staging/kube-prometheus-stack/*.yaml` | SOPS/age encrypted, decrypted by the `monitoring-configs` Kustomization |

Flux wiring is in `clusters/staging/monitoring.yaml`: `monitoring-controllers` applies `controllers/staging`, and `monitoring-configs` applies `configs/staging` with SOPS decryption.

## How the metrics flow

```
yogi kernel (/proc, /sys)
  -> node-exporter (DaemonSet, hostNetwork, :9100)
  -> Prometheus (ServiceMonitor scrape every 30s, TSDB on a local-path PVC)
  -> Grafana (Prometheus datasource) -> "yogi host" dashboard
```

kube-state-metrics and the kubelet/cAdvisor endpoints feed the k3s side: pod restarts and per-pod CPU and memory.

## What I found broken and how I fixed it

Before the fixes, Prometheus already had node-exporter data, but the setup wasn't healthy:

1. **Restart storms.** node-exporter had ~3,720 restarts and kube-state-metrics ~2,360. Comparing restart counters with the scrape gaps in Prometheus showed that 97% of them happened while yogi's Wi-Fi was down. One example is a 19-hour stretch from 6 to 7 Oct when only in-pod targets were still being scraped. The k3s node IP lives on `wlan0`, so when the Wi-Fi drops:
   - The kubelet's probes for node-exporter (`hostNetwork`, probed on the node IP) fail, and it kills a healthy exporter (exit 143).
   - kube-state-metrics can't reach the API server through `10.43.0.1`, which points at the same node IP. It exits with `network is unreachable`.

   **Fix ([#11](https://github.com/dkelertas-homelab/arch-cluster/pull/11)):** a Flux [post-renderer](https://fluxcd.io/flux/components/helm/helmreleases/#post-renderers) patches the node-exporter probes to hit `127.0.0.1`, with a 5s timeout instead of 1s. I also set modest requests and limits. The kube-state-metrics part needs a host-level change; see "Still to do".

2. **No persistent history.** Prometheus used an `emptyDir` with the default 10-day retention, so recreating the pod wiped everything.

   **Fix ([#12](https://github.com/dkelertas-homelab/arch-cluster/pull/12)):** an 8Gi `local-path` PVC with `retention: 30d` and `retentionSize: 4GB`. I copied the existing TSDB blocks into the new volume so the 10 days I already had (back to 28 Sep) survived the switch. I lost about 2.5 hours of head data.

3. **Evictions from disk pressure.** About 50 dead pods were sitting in `monitoring` (`Evicted`: `DiskPressure` or low `ephemeral-storage`). yogi's root filesystem is ~87% full. The kubelet evicts below ~5% free and can't reclaim enough image space. I deleted the dead pods once the fixes were live. The size cap in (2) is there so Prometheus can't push the disk over the edge. `local-path` doesn't enforce PVC sizes, so `retentionSize` is the real limit.

4. **Plaintext Grafana password** in `release.yaml` in this public repo.

   **Fix ([#13](https://github.com/dkelertas-homelab/arch-cluster/pull/13)):** a new random password in a SOPS-encrypted Secret (`grafana-admin-credentials`), wired in with `grafana.admin.existingSecret`. The old password is still in git history but no longer works.

5. **Dashboard ([#14](https://github.com/dkelertas-homelab/arch-cluster/pull/14)):** "yogi host" (uid `yogi-host`). Its rows:
   - Overview
   - CPU: by mode, per core, load, temps, battery
   - Memory and swap
   - Disk: usage per mountpoint, I/O, IOPS
   - Network: wlan0 traffic, errors, carrier changes
   - k3s: restarts, top pods by memory, CPU by namespace

## Reaching Grafana

- **Ingress:** Traefik serves `https://grafana.d11s.space` on yogi's node IP (currently `10.14.214.131`) with a self-signed cert from `scripts/create-grafana-tls`. Public DNS for `grafana.d11s.space` still points at `192.168.153.131`, an old LAN address, so the name doesn't reach yogi on my current network. It isn't exposed through a Cloudflare tunnel; only audiobs and linkding are. To use the hostname locally, add an `/etc/hosts` entry for the current node IP or update the DNS record.
- **Port-forward (always works on yogi):**

  ```bash
  kubectl --context default -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
  # then open http://localhost:3000/d/yogi-host/yogi-host
  ```

- **Admin password** (user `admin`):

  ```bash
  kubectl --context default -n monitoring get secret grafana-admin-credentials \
    -o jsonpath='{.data.admin-password}' | base64 -d; echo
  ```

The chart also ships its own node dashboards ("Node Exporter / Nodes", "Node Exporter / USE Method / Node") alongside mine.

## Adding a dashboard

1. Build or import the dashboard in Grafana, then export it as JSON (Share → Export, with "Export for sharing externally" off). Use a `datasource` variable rather than a hard-coded datasource UID.
2. Save it as `configs/staging/kube-prometheus-stack/dashboards/<name>.json`.
3. Add it to the `configMapGenerator` in `configs/staging/kube-prometheus-stack/kustomization.yaml`, with the label `grafana_dashboard: "1"`.
4. Check it renders: `kubectl kustomize monitoring/configs/staging`.
5. Open a PR and merge it. Flux applies the ConfigMap, and the [Grafana dashboard sidecar](https://github.com/grafana-community/helm-charts/tree/main/charts/grafana#sidecar-for-dashboards) loads it within a minute or so.

## Rotating the Grafana password

```bash
AGE_PUBLIC=age1jxudrgy7x6885yycvahksj5xl6kjhdqpwdqtgznm9r304kv22emqehe8n9
F=monitoring/configs/staging/kube-prometheus-stack/grafana-admin-credentials.yaml
kubectl create secret generic grafana-admin-credentials -n monitoring \
  --from-literal=admin-user=admin \
  --from-file=admin-password=<(openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 32) \
  --dry-run=client -o yaml > "$F"
sops --age="$AGE_PUBLIC" --encrypt --encrypted-regex '^(data|stringData)$' --in-place "$F"
```

Then commit, merge, and restart Grafana with `kubectl --context default -n monitoring rollout restart deploy/kube-prometheus-stack-grafana`. Grafana has no persistent DB here, so it picks up the new admin password on start.

## Still to do

- **Stable node IP for k3s.** kube-state-metrics will keep crash-looping whenever the Wi-Fi drops, because the API server is advertised on the `wlan0` address. The fix is to run k3s with `--node-ip`/`--advertise-address` on an interface that doesn't come and go, such as a wired link or a dummy interface ([k3s server options](https://docs.k3s.io/cli/server)). That's a host change, not GitOps.
- **Free up disk on `/`.** It's ~87% full and image garbage collection can't free enough, which is what caused the evictions ([node-pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)).
- **Grafana DNS.** Point `grafana.d11s.space` at the current address, or put it behind a Cloudflare tunnel like the apps.

## References

- [kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [node_exporter](https://github.com/prometheus/node_exporter)
- [Grafana chart: sidecar for dashboards](https://github.com/grafana-community/helm-charts/tree/main/charts/grafana#sidecar-for-dashboards)
- [Flux HelmRelease](https://fluxcd.io/flux/components/helm/helmreleases/) and [post-renderers](https://fluxcd.io/flux/components/helm/helmreleases/#post-renderers)
- [Manage Kubernetes secrets with SOPS (Flux)](https://fluxcd.io/flux/guides/mozilla-sops/)
- [Prometheus storage and retention](https://prometheus.io/docs/prometheus/latest/storage/), [Prometheus Operator storage](https://prometheus-operator.dev/docs/platform/storage/)
- [k3s storage (local-path)](https://docs.k3s.io/storage)
