---
title: Operations
description: Verifying the schedule ran, and what to check when it did not.
---

## Daily checks

```bash
# The jobs exist and carry the schedule you expect
kubectl -n longhorn-system get recurringjobs.longhorn.io

# Backups completed
kubectl -n longhorn-system get backups.longhorn.io \
  --sort-by=.metadata.creationTimestamp | tail -20

# Snapshots exist for a specific volume
kubectl -n longhorn-system get snapshots.longhorn.io \
  -l longhornvolume=<volume-name>
```

A `Backup` in state `Error` carries the reason in `.status.error`:

```bash
kubectl -n longhorn-system get backups.longhorn.io <name> -o jsonpath='{.status}' | jq
```

## The check that actually matters

Counting jobs proves scheduling. It does not prove coverage. The question worth asking on a
schedule is whether every volume you care about has a *recent* backup:

```bash
kubectl -n longhorn-system get backupvolumes.longhorn.io \
  -o custom-columns='VOLUME:.metadata.name,LAST:.status.lastBackupAt'
```

Anything with a stale or empty `lastBackupAt` is a volume you believe is protected and is
not.

Cross-check against the covered set:

```bash
kubectl -n longhorn-system get volumes.longhorn.io \
  -l recurring-job-group.longhorn.io/default=enabled -o name | wc -l
```

## Symptoms and causes

### No snapshots for a volume

The volume is probably not in the `default` group. Check the label:

```bash
kubectl -n longhorn-system get volumes.longhorn.io <name> \
  -o jsonpath='{.metadata.labels}'
```

Missing `recurring-job-group.longhorn.io/default=enabled` means its StorageClass sets its
own `recurringJobSelector`. See [Coverage model](/coverage/).

### Snapshots only for volumes that happen to be attached

The cluster-wide setting is off:

```bash
kubectl -n longhorn-system get settings.longhorn.io \
  allow-recurring-job-while-volume-detached -o jsonpath='{.value}'
```

If it reads `false`, either the chart was installed with
`clusterSettings.allowRecurringJobWhileVolumeDetached: null`, or something else on the
cluster overwrote it. This is the most consequential misconfiguration in the pack, because
it fails silently and skips exactly the idle user volumes you most want covered.

### Snapshots work, backups fail

Almost always the `BackupTarget`:

```bash
kubectl -n longhorn-system get backuptargets.longhorn.io default -o yaml
```

Check `status.available` and `status.conditions`. Bad credentials, an unreachable endpoint,
and a missing bucket all surface here rather than on the RecurringJob. See
[Backup target](/backup-target/).

### Backups run long or saturate the uplink

Lower `backup.concurrency` (default 3) or move `backup.cron` to a quieter hour. The first
night is the heaviest — backups are block-incremental afterwards.

### The jobs do not exist at all

The most likely cause is a release namespace that does not match `longhornNamespace`.
Longhorn only sees `RecurringJob` resources in its own namespace, and nothing errors when
they are rendered elsewhere:

```bash
kubectl get recurringjobs.longhorn.io -A
```

## Retention

Both retentions are **per volume**:

| | Default | Meaning |
|---|---|---|
| `snapshot.retain` | 24 | 24 snapshots of each volume — a rolling day at hourly cadence |
| `backup.retain` | 30 | 30 backups of each volume — a rolling month at nightly cadence |

Longhorn prunes on its own as new ones are taken. Changing `retain` downward does not
immediately delete the excess; the next run trims to the new count.

Never delete backup objects from the bucket by hand — that corrupts the block store. Use
the Longhorn UI or delete the `Backup` CR.

## Changing the schedule

```bash
helm upgrade backup nebari-longhorn-backup/nebari-longhorn-backup \
  --namespace longhorn-system \
  --set snapshot.cron='*/30 * * * *' \
  --set snapshot.retain=48
```

Halving the interval doubles the storage a day of snapshots costs, so raise `retain`
together with the frequency if you want to keep the same window.

Invalid values are rejected at render time rather than becoming a job that never fires:

```bash
helm template test . --set 'snapshot.cron=not-a-cron'
# Error: snapshot.cron is not a valid 5-field cron expression: "not-a-cron"
```

## Pausing protection

To stop one job without uninstalling:

```bash
helm upgrade backup ... --set snapshot.enabled=false
```

Existing snapshots and backups are untouched; only scheduling stops. Remember to turn it
back on — nothing will remind you.

## Verifying a chart change

```bash
helm lint .
helm template test . > /tmp/out.yaml
bash tests/render-test.sh

# Negative case: validation should fail the render
helm template test . --set 'snapshot.cron=not-a-cron'
```
