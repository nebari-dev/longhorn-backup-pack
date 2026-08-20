---
title: Introduction
description: "Hourly Longhorn snapshots and daily S3 backups for every volume on the cluster's default StorageClass."
---

The Nebari Longhorn Backup Pack schedules Longhorn-native snapshots and backups for the
cluster's default StorageClass. It renders two `RecurringJob`s and one cluster-wide
`Setting` — nothing else.

Small on purpose. The chart owns *when* volumes are protected; where the backups land is a
cluster concern configured outside it. See
[Backup target](/backup-target/) for that boundary.

```
  every volume on the default StorageClass
  (auto-labeled recurring-job-group.longhorn.io/default=enabled)
                        │
        ┌───────────────┴───────────────┐
        │                               │
  hourly snapshot                 daily backup
  0 * * * *  retain 24            0 3 * * *  retain 30
  in-cluster CoW                  streamed to the S3 BackupTarget
  RPO ≈ 1h, no S3 traffic         durable, survives the cluster
```

## What it renders

| Resource | Purpose | Default schedule | Retention |
|---|---|---|---|
| `RecurringJob/default-hourly-snapshot` | In-cluster copy-on-write snapshot. Fast, no S3 traffic. | `0 * * * *` | 24 (rolling 24h) |
| `RecurringJob/default-daily-backup` | Snapshot, then stream blocks to the S3 BackupTarget. | `0 3 * * *` | 30 (rolling 30 days) |
| `Setting/allow-recurring-job-while-volume-detached` | Lets RecurringJobs auto-attach detached volumes long enough to run. | — | — |

That third resource is the one that makes the other two useful on a JupyterHub cluster.
Longhorn's stock default silently skips detached volumes, and user PVCs are detached
whenever a server idles out — so without it, the users who most need protection are exactly
the ones who do not get it. See [Coverage model](/coverage/).

## Two layers of protection

Snapshots and backups answer different questions, which is why the chart schedules both at
different rates.

- A **snapshot** is a copy-on-write point in time held on the same Longhorn replicas as the
  live volume. Cheap and fast, so it runs hourly — but it dies with the cluster.
- A **backup** streams the changed blocks to object storage. Slower and bandwidth-bound, so
  it runs nightly — but it is the only thing that survives losing the cluster.

## In this guide

- **[Getting started](/getting-started/)** — install the chart and confirm the jobs exist
- **[Backup target](/backup-target/)** — the S3 configuration the chart deliberately does
  not manage
- **[Coverage model](/coverage/)** — which volumes are protected, and which are not

## Guides

- **[Restoring a volume](/restore/)** — the RWO and RWX runbooks, step by step
- **[Disaster recovery](/disaster-recovery/)** — rebuilding onto a fresh cluster from the
  same bucket
- **[Operations](/operations/)** — verifying jobs ran, and what to check when they did not

## Reference

- **[Configuration](/configuration/)** — every value, and the render-time validation guards
- **[Architecture](/architecture/)** — what the chart owns and where its boundaries are
