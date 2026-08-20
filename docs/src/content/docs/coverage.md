---
title: Coverage model
description: Which volumes this chart protects, how Longhorn decides, and what is deliberately out of scope.
---

## The rule

The two `RecurringJob`s target Longhorn's built-in **`default`** recurring-job-group:

```yaml
spec:
  groups:
    - default
```

Longhorn auto-labels a volume `recurring-job-group.longhorn.io/default=enabled` whenever
the volume's StorageClass has **no `recurringJobSelector` parameter**. That covers the
cluster's default `longhorn` StorageClass and any other class that does not set its own
selector.

The practical result: install the chart and every volume on the cluster's default
StorageClass is protected, with no per-workload opt-in and no annotations to remember on
new PVCs.

## Checking coverage

```bash
# Volumes Longhorn considers part of the default group
kubectl -n longhorn-system get volumes.longhorn.io \
  -l recurring-job-group.longhorn.io/default=enabled

# Every Longhorn volume, for comparison
kubectl -n longhorn-system get volumes.longhorn.io
```

A volume in the second list but not the first is not being protected.

## When a volume is missed

The single cause is a StorageClass that sets its own `recurringJobSelector`:

```yaml
parameters:
  recurringJobSelector: '[{"name":"my-group","isGroup":true}]'
```

That opts every volume of the class out of `default` and into `my-group` instead. If the
chart's jobs are the only ones on the cluster, those volumes get nothing.

Check which class a PVC actually used:

```bash
kubectl -n <namespace> get pvc <name> -o jsonpath='{.spec.storageClassName}'
kubectl get sc <name> -o jsonpath='{.parameters}'
```

:::caution[Two default StorageClasses is a real failure mode]
If a cluster has more than one class marked `storageclass.kubernetes.io/is-default-class:
"true"`, which class a PVC lands on depends on admission ordering, and some volumes may not
be Longhorn volumes at all. Confirm with `kubectl get sc` that exactly one class is
default.
:::

## Detached volumes

Coverage is only half the problem. Longhorn's stock
`allow-recurring-job-while-volume-detached` default is `false`, which makes RecurringJobs
**silently skip** any volume that is not attached when the cron fires.

On a JupyterHub cluster this is the difference between a working backup policy and a
decorative one. User PVCs are detached whenever a server idles out, so the volumes most
likely to be skipped are exactly the ones belonging to users who have been away — the users
whose data is least likely to be reproducible.

The chart therefore sets it to `true`. Longhorn briefly auto-attaches the volume, takes the
snapshot or backup, and detaches it again.

:::caution[This setting is cluster-wide]
It affects every RecurringJob on the cluster, not just the two this chart renders. Set
`clusterSettings.allowRecurringJobWhileVolumeDetached: null` to leave Longhorn's existing
value untouched.
:::

## What is out of scope

Deliberately, so the chart stays small:

- **Per-workload group targeting.** Protecting a subset of workloads on a different
  schedule means a StorageClass with its own `recurringJobSelector` and a second release
  targeting that group. That requires changes outside this chart's surface.
- **Dedicated paired StorageClasses.** An earlier design shipped a `longhorn-jhub` class
  paired with a `jhub` group. Targeting `default` instead means new workloads are covered
  without anyone remembering to pin a class.
- **Non-Longhorn volumes.** Anything on hostPath, a cloud CSI driver, or another provisioner
  is invisible to this chart.
- **Non-volume state.** The Hub database, Keycloak's Postgres, MLflow's Postgres, object
  storage contents, and Kubernetes resources like ConfigMaps and Secrets are all outside
  the scope. Losing the Hub DB resets named servers and last-active timestamps; file data
  is unaffected.

## What "protected" means

For each covered volume:

| | Snapshot | Backup |
|---|---|---|
| Where it lives | Longhorn replicas, same cluster | S3 bucket |
| Default frequency | hourly | nightly at 03:00 |
| Default retention | 24 (a rolling day) | 30 (a rolling month) |
| Recovery point | ≈ 1 hour | ≈ 24 hours |
| Survives cluster loss | no | yes |
| Costs bandwidth | no | yes |

Both are per-volume: retention counts snapshots or backups of *that volume*, not across the
cluster.

## RWX volumes

Read-write-many volumes are backed by a Longhorn share-manager pod exporting an underlying
RWO volume over NFS. The recurring jobs snapshot the **underlying RWO volume**, not the NFS
export.

Active writes through NFS are subject to write-back caching, so a snapshot taken mid-write
can miss the last few hundred milliseconds of buffered writes. For most workloads this is
acceptable; if you need stricter consistency, quiesce the share-manager before the snapshot
window. Restoring an RWX volume has its own steps — see
[Restoring a volume](/restore/#restoring-an-rwx-volume).
