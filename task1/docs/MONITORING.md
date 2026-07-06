# Monitoring Architecture Guide

Overview of how Argo CD is monitored: Prometheus scrapes it, Alertmanager routes alerts on it, Grafana visualizes it.

## Architecture

Four layers, bottom to top:

1. **Metrics exposure (Argo CD)** — each component (`server`, `repo-server`, `controller`, `redis`, `notifications`) exposes a `/metrics` endpoint and a `ServiceMonitor`, enabled via Helm values in `apps/argocd-app.yaml`.
2. **Metrics collection (Prometheus)** — discovers those `ServiceMonitor`s and scrapes them.
3. **Alerting (Prometheus + Alertmanager)** — evaluates `PrometheusRule`s, routes firing alerts to Alertmanager.
4. **Visualization (Grafana)** — queries Prometheus, auto-loads the dashboard from a ConfigMap.

## File structure

```text
apps/
├── argocd-app.yaml             # wave -5: Argo CD itself, metrics enabled
├── monitoring-app.yaml         # wave  0: kube-prometheus-stack (Operator, Prometheus, Grafana, Alertmanager)
├── monitoring-app-config.yaml  # wave  5: deploys monitoring/ below
└── monitoring/
    ├── prometheusrule-argocd.yaml     # alert rules
    └── grafana-dashboard-argocd.yaml  # dashboard ConfigMap
```

## Cross-namespace discovery

Prometheus only picks up a `ServiceMonitor`/`PrometheusRule` if both conditions hold:

- it's labeled `prometheus: kube-prometheus`
- it lives in a namespace labeled `monitoring: "true"`

Both the `argocd` and `monitoring` namespaces get that label imperatively in `Taskfile.yaml`'s `02-bootstrap-argocd` task (`kubectl label namespace ... monitoring=true`) — it's not GitOps-managed, so if the label is ever removed manually there's no auto-recovery.

Grafana dashboards work the same way: its sidecar container watches all namespaces for ConfigMaps labeled `grafana_dashboard: "1"` and imports them automatically (`apps/monitoring/grafana-dashboard-argocd.yaml`).

## Alerts

`apps/monitoring/prometheusrule-argocd.yaml` defines two groups:

- **Critical** (`argocd.critical`, 2 min threshold): server / repo-server / application-controller down
- **Warning** (`argocd.warnings`): app health degraded, sync failed, out-of-sync too long, slow reconciliation, high memory

Alertmanager is enabled with the chart's default (null) receiver — alerts are visible in its UI but not routed anywhere external. That's intentional for this demo; wiring a real receiver (Slack/PagerDuty) is a production follow-up, not a functional gap here.

The dashboard is the [an example from the Argo CD repo](https://github.com/argoproj/argo-cd/blob/master/examples/dashboard.json).

## Quick reference

```bash
# UIs
kubectl port-forward -n argocd svc/argocd-server 8443:443                       # https://localhost:8443
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 8082:80    # http://localhost:8082
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Verify discovery
kubectl get servicemonitor -n argocd
kubectl get prometheusrule -n monitoring
kubectl get configmap -n monitoring grafana-dashboard-argocd

# Useful PromQL
up{job=~"argocd.*"}                                    # target health
count by(sync_status) (argocd_app_info)                # apps by sync status
histogram_quantile(0.95, sum by(le) (rate(argocd_app_reconcile_bucket[5m])))  # reconcile p95
```

See `README.md`'s troubleshooting guide for step-by-step debugging of missing metrics/dashboards/alerts.

## Production considerations

This setup is intentionally minimal for a local demo. Several changes would be needed to use this setup in a production environment. Some ideas:

- Move namespace labels (`monitoring: "true"`) into declarative manifests instead of the imperative `kubectl label` in the Taskfile.
- Add persistent storage and HA (2+ Prometheus/Alertmanager replicas) — current setup uses ephemeral storage with 7-day retention, single replica.
- Route Alertmanager to a real receiver (Slack/PagerDuty) instead of the default null receiver.
- Put Grafana behind SSO instead of a Secret-based admin password.
