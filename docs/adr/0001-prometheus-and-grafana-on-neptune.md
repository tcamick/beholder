# ADR 0001 — Prometheus and Grafana on neptune, from their own repository

**Date:** 2026-11-21 · **Status:** accepted (owner, 2026-11-21) · **Relates to:** Plutus ADR 0010 (health report), 0011 (hosting on neptune), 0015 (neptune as a shared cluster)

## Context
Neptune (Ryzen 7 2700X, 125 GiB, single-node k3s with Flux, Tailscale and the External Secrets Operator) runs Plutus, and will run more apps (Plutus ADR 0015). Plutus's Go app, freshness controller and health reporter already expose Prometheus metrics, under bounded-label conventions, but nothing scrapes them. Plutus ADR 0010 rejected Prometheus and Grafana because of their memory on an 8 GB VM. That VM is gone; neptune uses about 2.4 GB of its 125 GiB. ADR 0010 also left memory and CPU history open, which a time-series store answers directly.

## Decision
- **A separate repository, `beholder`, owns the monitoring stack.** It is cluster infrastructure, used by every app on neptune, so it does not belong in Plutus (ADR 0015's direction). Until ADR 0015's cluster repository exists, the Flux `GitRepository` and `Kustomization` that point at beholder live in Plutus's `deploy/k8s/clusters/neptune/beholder.yaml`; they move with the rest of the cluster's Flux config.
- **kube-prometheus-stack**, by a Flux `HelmRelease` with a pinned chart version, in namespace `monitoring`: the Prometheus operator, Prometheus, Grafana, node-exporter and kube-state-metrics. It is the standard choice and brings Kubernetes and node dashboards with it; each app adds a `ServiceMonitor` or `PodMonitor` in its own repository and Prometheus selects them across the cluster.
- **k3s adjustments:** the controller manager, scheduler, proxy and etcd scrapes and their rules are off (k3s embeds them, bound to localhost, and uses kine, not etcd).
- **Dashboards only at first:** Alertmanager is off; the default recording and alerting rules are kept, so Prometheus's own Alerts page and Grafana show what would fire.
- **Storage:** Prometheus on a 50 Gi local-path volume, 30 days and at most 40 GB retained, scraping every 30 s. Grafana on a 2 Gi volume. Dashboards meant to last are ConfigMaps in Git (`grafana_dashboard: "1"`); the sidecar reads every namespace, so an app ships its dashboards beside its code.
- **Access: the tailnet identity alone.** Grafana is at `https://grafana.<tailnet>.ts.net` through the Tailscale operator's Ingress. Grafana's auth proxy trusts the `Tailscale-User-Login` header the Tailscale proxy sets and signs the person in (auto sign-up, Admin, one user). A `NetworkPolicy` admits only the `tailscale` namespace to Grafana's pod, so no other pod can forge the header. Prometheus has no Ingress; Grafana is the way in (`kubectl port-forward` for the Prometheus UI).
- **Pod security:** `monitoring` enforces privileged because node-exporter needs the host (network, PID namespace, read-only `/`), with baseline warnings and audit.

## Phases
1. **BH1 — The stack.** This repository, the Flux wiring in Plutus, the deploy key. Done when Grafana opens on the tailnet signed in as the owner, and the node and Kubernetes dashboards show neptune.
2. **BH2 — Plutus scraped** (in Plutus): `/metrics` moves from the dashboard port (8080, reachable from the tailnet without the password) to the internal port (8081); the job history in SQLite is exported at scrape time, because the nightly CronJob's own metrics die with its pod; `ServiceMonitor`s for the server, the freshness controller and the health reporter, and network policies admitting `monitoring` to their metrics ports; a Plutus dashboard in Git. The kustomization applying them depends on beholder's (the `ServiceMonitor` CRD).
3. **BH3 — Alerts**, when wanted: Alertmanager and a receiver.

## Rejected
- **The stack in the Plutus repository:** the fastest, but an investing knowledge base would own the cluster's monitoring, against ADR 0015.
- **Plain Prometheus and Grafana charts without the operator:** fewer CRDs, but hand-written scrape configs and dashboards, and no per-app `ServiceMonitor`.
- **VictoriaMetrics:** lighter and compatible with the same query language, but less standard, and memory is not the constraint here.
- **Grafana's own login, or anonymous access:** a second password for one person, or no record of who signed in. The tailnet already decides who can reach it.

## Consequences
- About 1.5–2 GB more memory on neptune and a few percent of a core.
- Plutus ADR 0010's "Memory and CPU history" can read from Prometheus instead of storing samples itself.
- A chart upgrade is a version bump here; the chart's CRDs are replaced on upgrade (`crds: CreateReplace`).
