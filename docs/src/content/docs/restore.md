---
title: Restoring a volume
description: Step-by-step restore runbooks for RWO and RWX Longhorn volumes.
---

This pack schedules backups; restore is a Longhorn operation. Both runbooks below use the
Longhorn UI, which is the practical path for a one-off recovery.

## Reaching the Longhorn UI

On a Nebari cluster the UI is exposed through the gateway at
`https://longhorn.<your-domain>/`, behind Keycloak OIDC.

Access is restricted to members of the `longhorn-admins` group in the `nebari` realm;
everyone else is denied. The seeded `admin` user is already a member — add yourself to that
group, then sign in through Keycloak.

## Before you start

Three rules govern every restore. Getting any of them wrong is the usual cause of a restore
that "worked" but left the workload broken.

1. **Stop the workload first.** Deleting a PVC while a pod still mounts it hangs on
   `pvc-protection` until the consumer is gone.
2. **PVC name and namespace must match the original exactly.** Kubernetes binds PVCs to PVs
   by `(name, namespace)`, not by UUID. Any mismatch orphans the new PV and the workload
   comes up with empty storage.
3. **Watch out for Argo CD `selfHeal`.** If the PVC is declared in a chart Argo CD manages
   with `selfHeal: true`, it will be recreated blank within about three minutes. Disable
   auto-sync on that Application for the duration, or complete the restore quickly.

## Restoring an RWO volume

Worked example: `claim-tpotts`, a user home directory in the `jupyterhub` namespace.

### 1. Stop the workload

```bash
kubectl -n jupyterhub get pod -l hub.jupyter.org/username=tpotts -o name | xargs -r kubectl delete
```

### 2. Delete the PVC

```bash
kubectl -n jupyterhub delete pvc claim-tpotts
```

With the default `Delete` reclaim policy on the `longhorn` StorageClass, this cascades to
the PV and the underlying Longhorn volume. That is intended — the restore creates a new
one.

### 3. Restore in the Longhorn UI

- **Backup** in the left sidebar → find the row for the deleted source volume, named
  `pvc-<UUID>` → expand it.
- Click the **timestamp** of the backup you want.
- Click **Restore Latest Backup**, or **Restore** from the dropdown on an older row.
- Fill in:
  - **Name** — any new identifier, e.g. `claim-tpotts-restored`. This becomes the new
    Longhorn volume name, not the PVC name.
  - **Number of Replicas** — typically `3`.
  - **Access Mode** — **`ReadWriteOnce`**.
  - Leave the rest at defaults.
- **OK**, then wait for the new volume to reach **`Detached`** in the **Volume** tab.
  Seconds to minutes, depending on backup size.

### 4. Re-expose it as a PV and PVC

- Open the restored volume's detail page (click its name under **Volume**).
- **Create PV/PVC**, top right.
- **PV Name** — any identifier, e.g. `pv-claim-tpotts-restored`.
- **PVC Name** — `claim-tpotts`. **Must match the original exactly.**
- **Namespace** — `jupyterhub`. **Must match the original exactly.**
- **Access Mode** — ReadWriteOnce.
- **OK**.

### 5. Verify and restart

```bash
kubectl -n jupyterhub get pvc claim-tpotts    # STATUS should be Bound
```

Spawn the user's server from the JupyterHub UI. KubeSpawner re-attaches the restored PVC.

## Restoring an RWX volume

Same flow, with two differences: the access mode is `ReadWriteMany` in both dialogs, and a
Longhorn **share-manager** pod must reach `Ready` before any consumer can mount the volume.

Worked example: the `shared-storage` PVC in `jupyterhub`.

### 1. Stop every consumer

All singleuser pods that mount the shared volume, not just one.

### 2. Delete the PVC

```bash
kubectl -n jupyterhub delete pvc shared-storage
```

### 3. Restore in the UI

As in the RWO flow, but set **Access Mode = `ReadWriteMany`**.

### 4. Create PV/PVC

- **PVC Name** — `shared-storage`
- **Namespace** — `jupyterhub`
- **Access Mode** — ReadWriteMany

### 5. Wait for the share-manager

Typically 30–60 seconds:

```bash
kubectl -n longhorn-system get sharemanagers.longhorn.io
# wait until the entry for the restored volume shows STATE=running
```

### 6. Verify from a consumer pod

```bash
mount | grep nfs    # should show the NFS mount on /shared/<...>
```

:::caution[RWX consumers need an NFS client on the node]
Every node running a pod that mounts an RWX volume needs `nfs-common` (or your distro's
equivalent) installed, with the `nfs`/`nfsv4` kernel modules loadable. The symptom of a
missing client is a pod event reading `mount: bad option; for several filesystems (e.g.
nfs, cifs) you might need a /sbin/mount.<type> helper program`.
:::

## Rolling back to a snapshot instead

If the data loss happened within the snapshot retention window and the cluster is healthy,
a snapshot revert is much faster than a backup restore — no S3 round trip, no PVC surgery.

The volume must be **detached** first (scale the workload to zero), then in the Longhorn UI
open the volume, pick a snapshot, and **Revert**. Re-attach by restarting the workload.

Reverting discards everything written after that snapshot. There is no undo.

## Gotchas

- **Stop the workload first** — otherwise the PVC delete hangs on `pvc-protection`.
- **PVC name + namespace must match exactly** — binding is by name and namespace, not UUID.
- **Argo CD `selfHeal` recreates the PVC blank** within ~3 minutes. Disable auto-sync during
  the restore.
- **In-flight backup locks.** A source cluster that died mid-backup can leave a stale lock in
  `backupstore/lock/`. Reads work; new writes complain until it is removed.
- **Two clusters, one bucket, corrupted metadata.** See
  [Disaster recovery](/disaster-recovery/).
