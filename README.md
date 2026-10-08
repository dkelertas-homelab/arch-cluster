k8s

GitOps repo for my single-node k3s homelab on yogi, reconciled by Flux.

- `apps/`: apps (audiobs, linkding)
- `infrastructure/`: n8n, renovate
- `monitoring/`: kube-prometheus-stack, plus a Grafana dashboard for yogi's host metrics. See [monitoring/README.md](monitoring/README.md).
