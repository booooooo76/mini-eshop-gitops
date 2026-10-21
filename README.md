# mini-eshop-gitops

Desired state of the [mini-eshop](https://github.com/booooooo76/mini-eshop) deployment on a local minikube cluster.
Argo CD watches this repository and keeps the cluster in sync with it.

## Layout

| Path | Contents |
|---|---|
| `charts/mini-eshop/` | Helm chart of the application (copied from `mini-eshop/helm/mini-eshop`, version 0.2.0) |
| `charts/mini-eshop/values-minikube.yaml` | Values for the minikube cluster: image tag, replicas, feature switches |

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
- `values-minikube.yaml` disables the `pre-upgrade` hook and the `helm test` pod. Argo CD runs
  `pre-upgrade` hooks as PreSync before every sync, including the first one, when PostgreSQL
  does not exist yet, and it does not run `helm test` at all.

## Render locally

```bash
helm lint charts/mini-eshop -f charts/mini-eshop/values-minikube.yaml
helm template mini-eshop charts/mini-eshop -n eshop-gitops -f charts/mini-eshop/values-minikube.yaml
```
