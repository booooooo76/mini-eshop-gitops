# mini-eshop-gitops

Desired state of the [mini-eshop](https://github.com/booooooo76/mini-eshop) deployment on a local minikube cluster.
Argo CD watches this repository and keeps the cluster in sync with it.

## Layout

| Path | Contents |
|---|---|
| `bootstrap/root.yaml` | Root Application (App of Apps). The only object applied by hand |
| `argocd-apps/` | Child Applications. The root Application syncs everything in this folder |
| `argocd-apps/argo-rollouts.yaml` | Argo Rollouts controller (Helm chart `argo-rollouts` 2.43.8), sync-wave -1, no finalizer so its CRDs survive |
| `argocd-apps/mini-eshop.yaml` | Application for the chart: namespace `eshop-gitops`, automated sync with prune and selfHeal |
| `argocd-apps/kube-prometheus-stack.yaml` | Prometheus, Alertmanager and Grafana (chart 92.18.0) in `monitoring`, server-side apply for the large CRDs |
| `argocd-apps/loki.yaml` | Loki 3.7 in monolithic mode with filesystem storage (chart `grafana-community/loki` 20.4.1) |
| `argocd-apps/alloy.yaml` | Grafana Alloy (chart 1.15.0): reads the logs of every pod through the Kubernetes API and pushes them to Loki |
| `charts/mini-eshop/` | Helm chart of the application (copied from `mini-eshop/helm/mini-eshop`, now 0.8.x) |
| `charts/mini-eshop/dashboards/mini-eshop.json` | Grafana dashboard shipped as a ConfigMap |
| `charts/mini-eshop/values-minikube.yaml` | Values for the minikube cluster: image tag, replicas, rollout deadline, feature switches |
| `loadgen/k6.yaml` | k6 load generator, applied by hand (not watched by Argo CD) |

## Bootstrap

1. Create the namespace and the database Secret (see [Secrets](#secrets)).
2. Apply the root Application once:

   ```bash
   kubectl apply -f bootstrap/root.yaml
   ```

Argo CD then reads `argocd-apps/`, creates the `mini-eshop` Application and deploys the chart.
Every later change goes through a commit to `main`.

## Secrets

No passwords are stored in Git. Grafana reads its admin login from the `grafana-admin` Secret
(keys `admin-user`, `admin-password`) in `monitoring`, created by hand the same way. The chart
reads the PostgreSQL credentials from a Secret that is created by hand once per cluster
(`secrets.existingSecret: mini-eshop-db`):

```powershell
kubectl create namespace eshop-gitops
kubectl create secret generic mini-eshop-db -n eshop-gitops `
  --from-literal=POSTGRES_USER=postgres `
  --from-literal=POSTGRES_PASSWORD=$pw
```

The services build their connection strings from `POSTGRES_USER` and `POSTGRES_PASSWORD`
(`$(VAR)` references in the Deployment), so the Secret needs only these two keys.

## Differences from the Lab 4 chart

- `secrets.existingSecret`: use a Secret created outside the chart instead of `--set secrets.postgresPassword=...`.
- The Secret holds only the user and password; connection strings are assembled in the Deployments.
- `app.progressDeadlineSeconds` (default 600, minikube 60): how long a rollout may take before Kubernetes reports `ProgressDeadlineExceeded` and Argo CD shows the app as Degraded.
- `values-minikube.yaml` disables the `pre-upgrade` hook and the `helm test` pod. Argo CD runs
  `pre-upgrade` hooks as PreSync before every sync, including the first one, when PostgreSQL
  does not exist yet, and it does not run `helm test` at all.

## Render locally

```bash
helm lint charts/mini-eshop -f charts/mini-eshop/values-minikube.yaml
helm template mini-eshop charts/mini-eshop -n eshop-gitops -f charts/mini-eshop/values-minikube.yaml
```

## Day-to-day changes

- Release: change `image.tag` in `values-minikube.yaml`, commit, push. Argo CD polls Git every ~3 minutes; `argocd app get mini-eshop --refresh` checks at once.
- Roll back: `git revert <commit>` and push. Argo CD's own Rollback is disabled while automated sync is on, so Git stays the source of truth.
- Manual changes in the cluster (`kubectl scale`, `kubectl edit`) are reverted by selfHeal within seconds.

## Progressive delivery (Argo Rollouts)

With `rollout.enabled: true` the five app services render as Argo `Rollout` objects instead of Deployments
(helpers `mini-eshop.workloadKind` and `mini-eshop.workloadStrategy`). `values-minikube.yaml` uses a canary
on 3 replicas: 20%, 30 s pause, an analysis step, then 60% and 80% with 30 s pauses.

- Release: change `image.tag`, commit, push, `argocd app get mini-eshop --refresh`. A healthy release passes the
  analysis and promotes itself; a bad one is aborted (Argo CD: Synced + Degraded) and is finished with `git revert`.
- Manual control still works: `kubectl argo rollouts promote|abort <rollout> -n eshop-gitops`.
- Pod templates carry checksums of the ConfigMap `data` only (helper `mini-eshop.dataChecksum`), so bumping the
  chart version does not start a new rollout.
- Check the split from inside the cluster, one request per connection:
  `kubectl -n eshop-gitops run curl --rm -it --image=curlimages/curl:8.11.1 -- sh -c 'while true; do curl -s http://mini-eshop-gateway:8080/version; echo; sleep 0.5; done'`

## Observability

The app images from 1.7.0 expose `/metrics` (`http_requests_total` and `http_request_duration_seconds`,
labels `method`, `route`, `status`), write one JSON log line per request to stdout and honour `FAIL_RATE`.
Chart switches in `values-minikube.yaml`:

| Value | What it adds |
|---|---|
| `metrics.serviceMonitor.enabled` | ServiceMonitor for the five services (port `http`, `/metrics`, 15 s), keeps the `rollouts-pod-template-hash` pod label |
| `metrics.dashboard.enabled` | ConfigMap with `grafana_dashboard: "1"`; the Grafana sidecar loads it from any namespace |
| `metrics.prometheusRule.enabled` | Alert `MiniEshopHighErrorRate`: more than 5% 5xx for 30 s, per service |
| `rollout.analysis.enabled` | AnalysisTemplate `mini-eshop-success-rate`: share of non-5xx answers of the canary pods, at least 95% |
| `app.failRate` | `FAIL_RATE` env var on every app pod; `"0.3"` makes a release fail the analysis |

Access (each in its own terminal):

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80        # http://localhost:3000, dashboard "mini-eshop"
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090   # Status -> Target health
kubectl apply -f loadgen/k6.yaml                                                    # load on; kubectl scale deploy/k6 -n loadgen --replicas=0 to stop
```

Useful queries:

```
sum by (service) (rate(http_requests_total{namespace="eshop-gitops", route!="/health"}[1m]))
histogram_quantile(0.95, sum by (service, le) (rate(http_request_duration_seconds_bucket{namespace="eshop-gitops"}[1m])))
{namespace="eshop-gitops", app=~"gateway|.+-api"} | json | status >= 500
sum by (app, pod_template_hash) (count_over_time({namespace="eshop-gitops"} | json | status >= 500 [1m]))
```

The monitoring stack needs about 2.5 GB on top of the application; with 6 GB minikube ran out of memory
(heavy IO wait, failing probes). The node runs with 8 GB (`docker update --memory 8g --memory-swap 8g minikube`).
