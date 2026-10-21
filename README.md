# mini-eshop-gitops

Desired state of the [mini-eshop](https://github.com/booooooo76/mini-eshop) deployment on a local minikube cluster.
Argo CD watches this repository and keeps the cluster in sync with it.

## Layout

| Path | Contents |
|---|---|
| `bootstrap/root.yaml` | Root Application (App of Apps). The only object applied by hand |
| `argocd-apps/` | Child Applications. The root Application syncs everything in this folder |
| `argocd-apps/mini-eshop.yaml` | Application for the chart: namespace `eshop-gitops`, automated sync with prune and selfHeal |
| `charts/mini-eshop/` | Helm chart of the application (copied from `mini-eshop/helm/mini-eshop`, version 0.3.0) |
| `charts/mini-eshop/values-minikube.yaml` | Values for the minikube cluster: image tag, replicas, rollout deadline, feature switches |

## Bootstrap

1. Create the namespace and the database Secret (see [Secrets](#secrets)).
2. Apply the root Application once:

   ```bash
   kubectl apply -f bootstrap/root.yaml
   ```

Argo CD then reads `argocd-apps/`, creates the `mini-eshop` Application and deploys the chart.
Every later change goes through a commit to `main`.

## Secrets

No passwords are stored in Git. The chart reads the PostgreSQL credentials from a Secret
that is created by hand once per cluster (`secrets.existingSecret: mini-eshop-db`):

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
