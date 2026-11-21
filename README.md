# beholder

Monitoring for neptune, the home k3s cluster: Prometheus and Grafana from the
kube-prometheus-stack chart, applied by Flux. Why and how: [ADR 0001](docs/adr/0001-prometheus-and-grafana-on-neptune.md).

```
 tailnet ──► Tailscale proxy ──► Grafana ──► Prometheus ──scrapes──► node-exporter (the host)
             (sets Tailscale-                               │         kube-state-metrics
              User-Login)                                   │         kubelet / cAdvisor, API server, CoreDNS
                                                            └───────► each app's ServiceMonitors (from its own repo)
```

## Layout
```
k8s/kustomization.yaml          what Flux applies (path ./k8s)
k8s/namespace.yaml              monitoring (privileged: node-exporter)
k8s/kube-prometheus-stack.yaml  HelmRepository + HelmRelease, values inline
k8s/grafana.yaml                Tailscale Ingress + NetworkPolicy for Grafana
docs/adr/                       decisions
```

## Using it
- **Grafana:** https://grafana.tail0adfbf.ts.net. You are signed in by your
  tailnet identity. If you get a login form instead, the identity header did
  not arrive; the fallback is `admin` with the password in the
  `kube-prometheus-stack-grafana` Secret in `monitoring`.
- **Prometheus UI:** no Ingress;
  `sudo k3s kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090`.
- **Adding an app's metrics:** in the app's repository, a `ServiceMonitor`
  (or `PodMonitor`) next to the app, plus a NetworkPolicy rule admitting the
  `monitoring` namespace to the metrics port. Prometheus selects monitors in
  every namespace. Dashboards go in a ConfigMap labelled
  `grafana_dashboard: "1"`, in any namespace.

## Changing it
Commit to `main` and push; Flux reconciles within a minute (the GitRepository
interval), or nudge it:
`kubectl -n flux-system annotate gitrepository beholder reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite`.
Check the release with `kubectl -n flux-system get helmrelease kube-prometheus-stack`.

Render locally before pushing: `kubectl kustomize k8s`.

## Wiring (one-time)
Flux reads this repository with its own read-only deploy key; GitHub deploy
keys are per repository, so Plutus's cannot be reused.

```sh
ssh-keygen -t ed25519 -N '' -C flux-beholder -f /tmp/beholder-deploy
# GitHub → tcamick/beholder → Settings → Deploy keys → add /tmp/beholder-deploy.pub (read-only)
sudo k3s kubectl -n flux-system create secret generic beholder-git-credentials \
  --from-file=identity=/tmp/beholder-deploy \
  --from-literal=known_hosts="github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl"
shred -u /tmp/beholder-deploy
```

The `GitRepository` and `Kustomization` that point here are in the Plutus
repository (`deploy/k8s/clusters/neptune/beholder.yaml`) until the cluster
gets its own repository (Plutus ADR 0015).
