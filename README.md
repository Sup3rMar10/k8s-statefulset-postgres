# k8s-statefulset-postgres

> **Reference configuration.** Rendered with `kubectl kustomize` only. It was **not applied to a cluster**, Postgres was never started, and the backup was never run or restored.

A single-instance PostgreSQL on Kubernetes done the stateful way: StatefulSet, headless Service, per-pod volume, probes, a PodDisruptionBudget and a nightly logical backup.

```
kubectl create namespace data
kubectl -n data create secret generic postgres-credentials --from-literal=password=CHOOSE_ONE
# also create a PVC named postgres-backups in namespace data (the backup target)
kubectl apply -k .
```

## Why a StatefulSet and not a Deployment

Stable pod name, stable DNS through the headless Service, and a PersistentVolumeClaim per pod that follows it across restarts. A Deployment gives none of that.

## Decisions

- **`PGDATA` in a subdirectory.** The volume root can contain `lost+found`, which makes initdb refuse to start.
- **`fsGroup: 999`** so the non-root postgres user can write to the volume.
- **`terminationGracePeriodSeconds: 60`** so shutdown can complete cleanly instead of being killed mid-write.
- **PDB `maxUnavailable: 0`** with one replica: a node drain waits for a person. Deliberate for a data pod; the cost is a blocked drain.
- **Password from a Secret you create**, nothing secret in Git.
- **Probes use `pg_isready`.** Readiness gates traffic; liveness is conservative with a 30 s start delay.

## What this is not

- **Not high availability.** One replica means downtime during node loss or upgrades. Real HA needs an operator (CloudNativePG, Zalando, Crunchy) that handles replication and failover.
- **No restore test.** A `pg_dump` that has never been restored is an unproven backup.
- **Backups sit in the same cluster.** Copy them off-cluster (object storage) for a real recovery plan.
- **Storage class, sizing and the `postgres:16.4` tag** were not checked against a real cluster or registry.
- No TLS, no connection pooling, no monitoring exporter, no WAL archiving or point-in-time recovery.

MIT licensed.
