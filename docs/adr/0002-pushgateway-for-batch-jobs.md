# ADR 0002 — A Pushgateway for batch jobs' counters

**Date:** 2026-11-21 · **Status:** accepted (owner, 2026-11-21) · **Relates to:** ADR 0001

## Context
Prometheus scrapes running pods. A batch job's pod (Plutus's nightly CronJob, the freshness controller's catch-up Jobs) lives minutes, between scrapes, and its counters end with it: how many price, EDGAR, FRED, news or YNAB requests the night made, and how many failed, were never seen. Plutus's BH2 already exports each job's outcome from its SQLite history at scrape time; what is missing is the per-source detail.

## Decision
- **The Prometheus Pushgateway** (`prometheus-pushgateway` chart, pinned), in `monitoring`, scraped by a ServiceMonitor with the pushed labels kept (`honorLabels`).
- **A batch run pushes once, at its end, with PUT,** so its group holds that run's counts and the next run of the same group replaces them: "last night's numbers", not a running total. A pushed group carries `push_time_seconds`, so a dashboard can say how old they are.
- **Grouping:** each app chooses its own `job` and grouping labels (Plutus: `job="plutus-batch"`, `batch` = `nightly` or the single job's name). A grouping label overwrites a label of the same name in the pushed body, so an app does not push families that carry its own `job` label.
- **Persistence:** a 1 Gi volume (`--persistence.file`), so a gateway restart does not lose the last run's counts.
- **Who may push:** the gateway has no authentication. A `NetworkPolicy` admits namespaces labelled `beholder.neptune/push-metrics: "true"`, and Prometheus; each app's own egress policy limits which of its pods send.

## Rejected
- **Recording each source's counts in the app's own store** and exporting them at scrape time, as BH2 does for job outcomes: no new component, but every app writes its own bookkeeping for what the Pushgateway does generically.
- **Prometheus's remote-write receiver:** the batch job would need a remote-write client, and the samples would be raw time series rather than a group replaced per run.
- **Running the nightly jobs inside the long-lived server:** scraped like everything else, but it undoes the separation between the server and the batch work (Plutus ADR 0009) for a metrics concern.

## Consequences
- Pushed groups stay until replaced or deleted; a batch that stops running leaves its last counts, aging in `push_time_seconds`.
- `rate()` over pushed counters is meaningless across runs (each push restarts them); dashboards show the values as they stand.
