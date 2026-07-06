# ArgoCD challenge - task 1

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Description](#description)
- [Assumptions](#assumptions)
- [Repo structure](#repo-structure)
- [Verification and testing](#verification-and-testing)
- [Troubleshooting guide](#troubleshooting-guide)
- [Real-World Production Considerations](#real-world-production-considerations)

## Prerequisites

### To execute the test

Ensure you have the following installed locally:

- Docker Desktop or equivalent running
- `kind`
- `kubectl`
- [`task`](https://taskfile.dev/)
- `helm`

### To develop and commit to the repo

To commit to the repo, you also need:

- `pre-commit`
- `detect-secrets`

Execute the following command to initialize the environment for `pre-commit`:

```shell
task init-hooks
```

## Quick start

1. Clone the repository.

2. Create the `kind` cluster:

   ```bash
   task 01-create-cluster
   ```

   - check that the cluster is running: `kubectl get nodes`
   - in a new terminal, run this command to follow the changes: `watch -n 5 "kubectl get pods -A --sort-by=.status.startTime --no-headers| tac"`

3. Run the bootstrap command:

   ```bash
   task 02-bootstrap-argocd
   ```

   - check that ArgoCD is running and check the version of Helm inside the `repo-server`; expect something like `v3.19.4+g7cfb6e4`

   ```bash
   kubectl get pods -n argocd
   task check-helm-version-inside-repo-server
   ```

   - open the ArgoCD UI:

      - get the ArgoCD admin password:

      ```bash
      task extract-admin-password
      ```

      - Run this command in one terminal (or send it in the background):

      ```bash
      kubectl port-forward svc/argocd-server -n argocd 8443:443
      ```

      - open [https://localhost:8443](https://localhost:8443) - accept the risk of self-signed certificate, and login as `admin` using the password from the previous step

4. Apply the manifest for the app-of-apps:

   ```bash
   task 03-apply-root-app
   ```

   **WARNING**
   This will cause a restart of `argocd-server` which will kill any `port-forward` command. Remember to restart the `port-forward`

5. Wait for all applications to sync (this may take 5-10 minutes for Prometheus to fully deploy). **The previous `port-forward` will close and you will need to run it once more.**

6. Verify the deployments:

   ```bash
   # Check Argo CD applications
   kubectl get applications -n argocd

   # Check Argo CD pods
   kubectl get pods -n argocd

   # Check Prometheus stack pods
   kubectl get pods -n monitoring
   ```

7. Open the Prometheus and the Grafana UI

   ```bash
   # Port forward to Grafana
   kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 8082:80
   Forwarding from 127.0.0.1:8082 -> 3000

   # extract password
   task extract-grafana-password

   # Access at: http://localhost:3000
   # Username: admin
   ```

   ```bash
   kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

   # Access at: http://localhost:9090
   ```

8. Check the Helm version inside the `repo-server` (see above); this time you should see `v3.7.2+g663a896`:

   ```bash
   task check-helm-version-inside-repo-server
   task: [check-helm-version-inside-repo-server] echo "Checking Helm version inside Argo CD repo-server"
   Checking Helm version inside Argo CD repo-server
   task: [check-helm-version-inside-repo-server] kubectl exec -n argocd deploy/argocd-repo-server -c repo-server -- helm version --short
   v3.7.2+g663a896
   ```

## Description

Deploy Argo CD and prepare a GitOps repo that makes Argo CD do the following:

1. Manage its own configurations/lifecycle

2. Deploy Prometheus and any required configurations to make it monitor Argo CD (with dashboards and alerts)

3. Replace the Helm binary included with Argo CD with a different version of Helm and specifically use Helm version 3.7.2

## Assumptions

- There is an initial manual bootstrap to install ArgoCD via `helm template` + `kubectl apply`. `helm` could not be used to install ArgoCD because it would have created ownership problems for ArgoCD itself.

- Use `app-of-apps` to describe the components to ArgoCD

- `Applications` #1 is the ArgoCD instance itself. To try to avoid a `suicide loop`, we use the following strategies:

  - `ServerSideApply`: helps with CRDs size problem and with unintentional drift caused by slight changes in the manifest by the `kubectl apply`.

  - `Prune=false` in the `sync-options`, so that in case of an `out of sync`, we try to prevent ArgoCD from killing itself.

  ```YAML
  metadata:
     annotations:
        argocd.argoproj.io/sync-options: Prune=false
  ```

- The bootstrap installs vanilla ArgoCD while the ArgoCD Application manifests contain the replacement of `helm` with version 3.7.2 via `initContainer`, which only happens when ArgoCD takes control of itself; this causes a restart of all services with loss of connectivity to the UI but overall ArgoCD should come back online.

### Monitoring

See [MONITORING.md](docs/MONITORING.md)

## Repo structure

```text
.
├── apps
│   ├── argocd-app.yaml
│   ├── monitoring
│   │   ├── grafana-dashboard-argocd.yaml
│   │   └── prometheusrule-argocd.yaml
│   ├── monitoring-app-config.yaml
│   └── monitoring-app.yaml
├── bootstrap
│   ├── root-app.yaml
│   └── values-bootstrap.yaml
├── docs
│   ├── argocdscreenshot01.png
│   ├── design.png
│   ├── grafana.png
│   ├── MONITORING.md
│   ├── prom-alert-firing.png
│   ├── prom-alerts-overview.png
│   ├── prom-rules.png
│   ├── prom-targets.png
│   └── TESTING.md
├── kind-config.yaml
├── README.md
└── Taskfile.yaml
```

## Verification and testing

See [TESTING.md](docs/TESTING.md)

## **Troubleshooting Guide**

This section covers common issues you might encounter and how to resolve them.

### Generic ArgoCD troubleshooting

   1. Check pod status:

      ```bash
      kubectl get pods -n argocd
      ```

   2. Look for pods in `CrashLoopBackOff`, `Error`, or `Pending` state

   3. Check pod logs:

      ```bash
      kubectl logs -n argocd deploy/argocd-server
      kubectl logs -n argocd deploy/argocd-repo-server
      kubectl logs -n argocd deploy/argocd-application-controller
      ```

   4. Common fixes:
      - Wait longer (initial image pulls can take 2-3 minutes)
      - Check resource availability: `kubectl describe node`
      - Verify kind cluster is healthy: `kind get clusters`

### `kind` cluster issues

- `kubectl` commands are failing:
  - check with `docker` or `crictl` if the `kind` containers are running

- `argocd` CLI is not working or `port-forward` for ArgoCD fails:
  - check if Pods are running: `kubectl get pods -n argocd`

### Problem: Root application not syncing child applications

- Root app shows "Synced" but no child applications visible
- No applications appear in Argo CD UI

- Troubleshooting Steps:

1. Verify root application exists:

   ```bash
   kubectl get application root-app -n argocd -o yaml
   ```

2. Check application status:

   ```bash
   kubectl get application -n argocd
   ```

3. Check Argo CD application controller logs:

   ```bash
   kubectl logs -n argocd deploy/argocd-application-controller | grep -i error
   ```

4. Common causes:
   - Incorrect `repoURL` in root-app.yaml (must match your fork/repo)
   - Network connectivity issues

5. Fix:

   ```bash
   # Update repoURL if needed - apply to the manifests in Github

   # Force refresh
   kubectl patch application root-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
   ```

### Helm Version Verification Issues

#### Problem: Helm version still shows default version (not 3.7.x)

- Troubleshooting Steps:

1. Check if init container ran successfully:

   ```bash
   kubectl describe pod -n argocd -l app.kubernetes.io/component=repo-server
   ```

2. Check init container logs:

   ```bash
   kubectl logs -n argocd argocd-repo-server-xxxx-xxxx -c install-helm-3-7
   ```

3. Common issues:
   - Volume mount path conflict
   - Init container failed to pull `helm` image

### Prometheus Monitoring Issues

#### Problem: ServiceMonitors not being discovered by Prometheus

- Argo CD targets not visible in Prometheus UI under Status → Targets
- No Argo CD metrics available in Prometheus

- Troubleshooting Steps:**

1. Verify ServiceMonitors exist:

   ```bash
   kubectl get servicemonitor -n argocd
   ```

2. Check ServiceMonitor labels match Prometheus selector:

   ```bash
   kubectl get servicemonitor -n argocd -o yaml | grep -A 10 labels
   ```

   Should include: `prometheus: kube-prometheus`

3. Verify namespace has monitoring label:

   ```bash
   kubectl get namespace argocd -o yaml | grep labels -A 5
   ```

   Should include: `monitoring: "true"`

4. Check Prometheus configuration:

   ```bash
   kubectl get prometheus -n monitoring -o yaml | grep -A 10 serviceMonitorSelector
   ```

5. Check Prometheus operator logs:

   ```bash
   kubectl get pods -n monitoring | grep operator
   # find Pod name
   kubectl logs -n monitoring kube-prometheus-stack-operator-xxxxx-xxxxx
   ```

6. Force Prometheus reload:

   ```bash
   kubectl delete pod -n monitoring -l app.kubernetes.io/name=prometheus
   ```

#### Problem: Grafana dashboard not showing Argo CD data

- Dashboard exists but panels show "No data"
- Queries return empty results

- Troubleshooting Steps

1. Verify Prometheus is scraping Argo CD metrics:
   - Open Prometheus UI: `http://localhost:9090`
   - Try query: `up{job=~"argocd.*"}`
   - Should return results with `value=1`

2. Check if metrics endpoints are accessible:

   ```bash
   kubectl port-forward -n argocd svc/argocd-server-metrics 9999:8083
   curl http://localhost:9999/metrics
   ```

   Should return Prometheus-format metrics

3. Verify Grafana data source configuration:
   - Open Grafana UI -> Left menu -> Connections -> Data Sources
   - Check Prometheus URL is correct
   - Test connection (should show green checkmark)

4. Check dashboard ConfigMap:

   ```bash
   kubectl get cm -n monitoring grafana-dashboard-argocd -o yaml
   ```

5. Verify Grafana sidecar is loading dashboards:

   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
   ```

#### Problem: Prometheus alerts not firing

- PrometheusRule exists but no alerts visible in Prometheus UI or Alertmanager

- Troubleshooting Steps:

1. Verify PrometheusRule exists and is valid:

   ```bash
   kubectl get prometheusrule -n monitoring argocd-alerts -o yaml
   ```

2. Check PrometheusRule labels:

   ```bash
   kubectl get prometheusrule -n monitoring argocd-alerts -o jsonpath='{.metadata.labels}'
   ```

   Should include: `prometheus: kube-prometheus`

3. Check Prometheus rule configuration in UI:
   - Open Prometheus: `http://localhost:9090`
   - Navigate to Status → Rules health
   - Look for `argocd.critical` and `argocd.warnings` groups

4. Check Prometheus operator logs for rule loading errors:

   ```bash
   kubectl logs -n monitoring -l  app.kubernetes.io/component=prometheus-operator| grep -i "prometheusrule"
   ```

5. Manually trigger an alert for testing:

   ```bash
   # Scale down Argo CD server to trigger alert
   kubectl scale deployment argocd-server -n argocd --replicas=0

   # Wait 2-3 minutes, then check Prometheus alerts
   # Remember to scale back up:
   kubectl scale deployment argocd-server -n argocd --replicas=1
   ```

### Self-Management Issues

#### Problem: Argo CD application shows "OutOfSync" constantly

- Argo CD application in UI shows red "OutOfSync" status
- Diff shows changes that shouldn't be there

- Troubleshooting Steps:**

1. Check what's out of sync:

   ```bash
   kubectl get application argocd -n argocd -o yaml | grep -A 50 status
   ```

2. Review diff in Argo CD UI to identify divergent fields

3. Common causes:
   - Server-side defaults added by Kubernetes API
   - Helm chart changes between versions
   - CRD field modifications

4. Solutions:
   - Add field to `ignoreDifferences` in Application spec
   - Use `ServerSideApply=true` (already configured)
   - Update values.yaml to match actual state

5. Force sync if safe:

   ```bash
   # Review changes first in UI!
   kubectl patch application argocd -n argocd --type merge -p '{"operation":{"sync":{"syncStrategy":{"hook":{}}}}}'
   ```

#### Problem: Argo CD suicide loop (deleting its own pods)

- Argo CD pods constantly restarting
- Application shows sync in progress repeatedly
- Events show pods being deleted and recreated

- Troubleshooting Steps:

1. Immediately disable auto-sync:

   ```bash
   kubectl patch application argocd -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'
   ```

2. Check if `Prune=false` annotation is set:

   ```bash
   kubectl get application argocd -n argocd -o jsonpath='{.metadata.annotations}'
   ```

   Should include: `argocd.argoproj.io/sync-options: Prune=false`

3. Identify what's causing the loop:
   - Check Application events: `kubectl describe application argocd -n argocd`
   - Review sync logs in Argo CD UI

4. Fix by ensuring proper sync options:

   ```bash
   kubectl annotate application argocd -n argocd \
     argocd.argoproj.io/sync-options=Prune=false --overwrite
   ```

### 5. General Debugging Commands

#### View all resources in Argo CD namespace

```bash
kubectl get all -n argocd
```

#### Check Argo CD application controller logs for sync issues

```bash
kubectl logs -n argocd -l app.kubernetes.io/component=application-controller --tail=100 -f
```

#### Check repo-server logs for Helm/Git issues

```bash
kubectl logs -n argocd -l app.kubernetes.io/component=repo-server --tail=100 -f
```

#### Get detailed application status

```bash
kubectl describe application <app-name> -n argocd
```

#### Force application refresh

```bash
argocd app get <app-name> --hard-refresh
# or via kubectl:
kubectl patch application <app-name> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

#### Clean up

```bash
# Delete everything
task clean
```

## Real-World Production Considerations

While this solution is designed for local testing, it lacks completely any feature that would be expected in a real world production scenario, like HA, performance features, DR, secure hardening and more.
