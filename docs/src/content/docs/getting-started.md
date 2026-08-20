---
title: Getting started
description: Install the Longhorn Backup Pack and confirm the recurring jobs exist.
---

## Prerequisites

Two things must exist before the chart is useful. Neither is managed by it.

- **Longhorn**, installed in `longhorn-system` (or wherever you point
  `longhornNamespace`). The chart renders Longhorn CRDs and nothing else, so the CRDs must
  be registered.
- **A `BackupTarget`** with a valid S3 URL and credentials Secret. Without one, the hourly
  snapshots still work — they are in-cluster — but the daily backup job fails every night.
  See [Backup target](/backup-target/).

You also need Helm 3.

## Install

```bash
helm repo add nebari-longhorn-backup https://nebari-dev.github.io/nebari-longhorn-backup-pack
helm repo update

helm install backup nebari-longhorn-backup/nebari-longhorn-backup \
  --namespace longhorn-system
```

:::caution[The release namespace must match `longhornNamespace`]
Longhorn only acts on `RecurringJob` resources in its own namespace. Installing into a
different namespace without also setting `longhornNamespace` renders the jobs where
Longhorn will never see them — and nothing errors. The two must agree.
:::

Or from a clone:

```bash
helm install backup . --namespace longhorn-system
```

Or via Argo CD, with a standard `Application` pointing at the published chart.

## Confirm it took

The chart prints a summary on install. To check the cluster itself:

```bash
kubectl -n longhorn-system get recurringjobs.longhorn.io
```

```
NAME                       TASK       CRON        RETAIN   CONCURRENCY
default-daily-backup       backup     0 3 * * *   30       3
default-hourly-snapshot    snapshot   0 * * * *   24       5
```

And the cluster-wide setting:

```bash
kubectl -n longhorn-system get settings.longhorn.io \
  allow-recurring-job-while-volume-detached -o jsonpath='{.value}'
# true
```

## Confirm volumes are actually covered

Rendering the jobs is not the same as protecting anything. Longhorn labels a volume into
the `default` group only when its StorageClass has no `recurringJobSelector` parameter, so
check the label is really there:

```bash
kubectl -n longhorn-system get volumes.longhorn.io \
  -l recurring-job-group.longhorn.io/default=enabled
```

Every volume you expect to protect should be listed. If one is missing, its StorageClass
sets its own selector — see [Coverage model](/coverage/).

## Wait for the first run

The snapshot job fires at the top of the hour, the backup job at 03:00. After the first
tick:

```bash
# Snapshots on a volume
kubectl -n longhorn-system get snapshots.longhorn.io

# Backups that reached the bucket
kubectl -n longhorn-system get backups.longhorn.io
kubectl -n longhorn-system get backupvolumes.longhorn.io
```

To avoid waiting for 03:00 on a first install, temporarily set `backup.cron` to a few
minutes out, confirm a `Backup` object appears and reaches `Completed`, then set it back.

## Tune the schedule

```bash
helm upgrade backup nebari-longhorn-backup/nebari-longhorn-backup \
  --namespace longhorn-system \
  --set snapshot.cron='*/30 * * * *' \
  --set snapshot.retain=48 \
  --set backup.retain=90
```

Invalid input is caught at render time rather than becoming a job that never fires — a
malformed cron or a non-positive retention fails `helm template` with a clear message. See
[Configuration](/configuration/).

## Uninstall

```bash
helm uninstall backup -n longhorn-system
```

This removes the two `RecurringJob`s, so scheduling stops. It does **not** delete existing
snapshots or anything already in the backup bucket — those are Longhorn objects with their
own lifecycle, and your restore history survives.

The cluster-wide `Setting` object is also removed. Longhorn falls back to its own default
(`false`) for `allow-recurring-job-while-volume-detached`, which affects every other
RecurringJob on the cluster too — see [Configuration](/configuration/#cluster-settings).
