
# postgres-kubernetes

Kubernetes manifests for running PostgreSQL as an internal database for SUNBOYS services.

## What It Deploys

- Namespace `infra`
- Secret with PostgreSQL credentials and database list
- ConfigMap with init script
- PersistentVolumeClaim for database data
- StatefulSet with PostgreSQL
- ClusterIP Service available inside the cluster

## Ready-To-Use Targets

| Target | Command | Service DNS |
| --- | --- | --- |
| Shared dev database | `kubectl apply -k k8s/overlays/dev` | `postgres.infra.svc.cluster.local:5432` |
| Auth service database | `kubectl apply -k k8s/overlays/auth-service` | `auth-postgres.infra.svc.cluster.local:5432` |
| Main service database | `kubectl apply -k k8s/overlays/main-service` | `main-postgres.infra.svc.cluster.local:5432` |
| CI/test database | `kubectl apply -k k8s/overlays/ci` | `ci-postgres.infra.svc.cluster.local:5432` |

## Deploy

```bash
kubectl apply -k k8s/overlays/dev
kubectl -n infra rollout status statefulset/postgres
```

For a prefixed target, wait for the prefixed StatefulSet:

```bash
kubectl -n infra rollout status statefulset/auth-postgres
```

## Automatic Deploy

GitHub Actions deploys both `auth-postgres` and `main-postgres` on every push to
`master` or `main`. The workflow can also be started manually.

Configure these GitHub Actions repository secrets before the first deployment:

- `KUBECONFIG_B64`: base64-encoded kubeconfig, as used by the service repositories
- `POSTGRES_ADMIN_PASSWORD`: password for the PostgreSQL administrator
- `AUTH_DB_PASSWORD`: password for the `auth_service` user
- `MAIN_DB_PASSWORD`: password for the `main_service` user

Create `KUBECONFIG_B64` from a working kubeconfig on Linux/macOS:

```bash
base64 < ~/.kube/config | tr -d '\n'
```

On PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$HOME/.kube/config"))
```

The workflow renders both Kustomize overlays, applies them to the `infra`
namespace, and waits for both StatefulSets to become ready.

## Create A New Target

Copy the template overlay:

```bash
cp -r k8s/overlays/template k8s/overlays/my-purpose
mv k8s/overlays/my-purpose/secret.env.example k8s/overlays/my-purpose/secret.env
```

Edit:

- `namePrefix` in `k8s/overlays/my-purpose/kustomization.yaml`
- `SERVICE_DATABASES` in `k8s/overlays/my-purpose/secret.env`
- `APP_DB_USER` and `APP_DB_PASSWORD`
- PVC storage size patch if the target needs more or less disk

Then deploy:

```bash
kubectl apply -k k8s/overlays/my-purpose
```

## Database List

`SERVICE_DATABASES` is a comma-separated list created on first PostgreSQL startup:

```text
SERVICE_DATABASES=auth_service,package_service,notification_service
```

The init script runs when PostgreSQL initializes an empty data directory. The
deployment workflow also runs it after every rollout, so missing roles and
databases are created for existing PVCs. Application roles are kept as login-only
roles without `CREATEDB`, `CREATEROLE`, or superuser privileges.

## Connection String

Shared dev target:

```text
Host=postgres.infra.svc.cluster.local;Port=5432;Database=auth_service;Username=sunboys_app;Password=sunboys_app_password
```

Auth service target:

```text
Host=auth-postgres.infra.svc.cluster.local;Port=5432;Database=auth_service;Username=auth_service;Password=change_me_auth
```

Main service target:

```text
Host=main-postgres.infra.svc.cluster.local;Port=5432;Database=main_service;Username=main_service;Password=change_me_main
```

## Configuration

Target defaults are in each overlay's `secret.env`.

Before using this in production, replace the passwords and avoid committing real secrets. For production, prefer Sealed Secrets, External Secrets Operator, SOPS, or your cloud secret manager.

## Check Status

```bash
kubectl -n infra get pods
kubectl -n infra get svc postgres
kubectl -n infra logs statefulset/postgres
```

## Remove

This removes the workload but keeps data if the PVC is retained by your storage class:

```bash
kubectl delete -k k8s/overlays/dev
```

To delete database data too:

```bash
kubectl -n infra delete pvc postgres-data
```
