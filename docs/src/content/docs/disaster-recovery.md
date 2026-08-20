---
title: Disaster recovery
description: Rebuilding onto a fresh cluster from the same Longhorn backup bucket.
---

The reason the nightly backup exists: the cluster is gone, and the data has to come back
somewhere else.

## What survives

Only what reached the bucket. Snapshots live on Longhorn replicas in the cluster that no
longer exists, so the recovery point is the **last completed nightly backup** — up to 24
hours old with the default schedule.

Volume data survives. Cluster state does not: the Hub database, Keycloak's Postgres, and
any Kubernetes resources are outside this pack's scope. Users get their files back and a
reset server list.

## Procedure

### 1. Stand up Longhorn on the new cluster

Install Longhorn as normal, then create the `BackupTarget` pointing at the **same**
`backupTargetURL` with the same credentials. See [Backup target](/backup-target/).

### 2. Wait for discovery

Longhorn re-reads the bucket every `pollInterval` (default 5 minutes) and auto-discovers
backups it did not create:

```bash
kubectl -n longhorn-system get backupvolumes.longhorn.io
```

Every volume from the old cluster should appear, named `pvc-<UUID>`. Nothing appearing
after two poll intervals means the target is misconfigured — check
`kubectl -n longhorn-system get backuptargets.longhorn.io default -o yaml` for
`status.available` and conditions.

### 3. Restore each volume

Follow the RWO or RWX runbook in [Restoring a volume](/restore/) for each one. On a fresh
cluster there is no PVC to delete first, so you start at the Longhorn UI restore step.

**Reuse the original PVC name and namespace.** That is what lets workloads bind without
manual intervention — deploy the charts as normal and their PVCs bind to the restored PVs.

Identifying which `pvc-<UUID>` was which claim is the tedious part. Longhorn records the
original PVC in the backup volume's labels:

```bash
kubectl -n longhorn-system get backupvolumes.longhorn.io -o custom-columns=\
'BACKUP:.metadata.name,PVC:.status.labels.KubernetesStatus'
```

If you still have the old cluster's manifests or `kubectl get pv -o yaml` output, the
`claimRef` on each PV is the more direct mapping — worth exporting *before* you need it.

### 4. Bring workloads up

Deploy the platform charts as normal. For JupyterHub, spawn user servers; file data is
intact, and the hub DB resets unless it was backed up separately.

### 5. Re-install this chart

```bash
helm install backup nebari-longhorn-backup/nebari-longhorn-backup \
  --namespace longhorn-system
```

Easy to forget in the rush, and the new cluster is unprotected until you do.

:::caution[Rotate the bucket prefix first]
Do not point the new cluster's scheduled backups at the same prefix while there is any
chance the old cluster is still alive. See below.
:::

## The split-brain hazard

Two live clusters writing to the same `backupstore/` prefix corrupt each other's metadata.
This is not theoretical during a DR: a "destroyed" cluster is often only unreachable, and
it may come back — nodes rebooting, a network partition healing, a hypervisor restored — and
resume its 03:00 backup into the bucket the new cluster is now also writing to.

Before declaring the new cluster authoritative, do one of:

- point the new cluster's `BackupTarget` at a **new prefix** in the same bucket, or
- **rotate the credential** the old cluster holds so its writes fail.

Restoring *from* the old prefix is read-only and safe; it is concurrent *writes* that
destroy the block store.

## Stale locks

Longhorn coordinates bucket access with lock objects under `backupstore/lock/`. A cluster
that died mid-backup can leave one behind. Reads — including every restore — keep working;
new writes complain until the lock is removed with `mc rm` or the equivalent.

## Practice it

The whole procedure depends on details that are cheap to verify in advance and expensive to
discover during an outage:

- Are backups actually completing? `kubectl -n longhorn-system get backups.longhorn.io`
  after a nightly run. See [Operations](/operations/).
- Can you map `pvc-<UUID>` back to a claim? Export `kubectl get pv -o yaml` somewhere
  outside the cluster.
- Do you know the bucket credentials independently of the cluster that holds them?
- Have you restored one volume end to end, into a scratch namespace, at least once?

A restore that has never been run is a backup policy, not a recovery plan.
