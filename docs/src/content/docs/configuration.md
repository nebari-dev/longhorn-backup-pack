---
title: Configuration
description: Every value the Longhorn Backup Pack chart accepts, and the render-time validation guards.
---

The whole surface is small enough to list. Defaults are in
[`values.yaml`](https://github.com/nebari-dev/longhorn-backup-pack/blob/main/values.yaml).

## Placement

| Value | Default | Purpose |
|---|---|---|
| `longhornNamespace` | `longhorn-system` | Namespace the `RecurringJob`s and `Setting` are rendered into. |

:::caution[This must be Longhorn's own namespace]
Longhorn only acts on `RecurringJob` resources in the namespace where it runs. Rendering
them elsewhere produces valid objects that are silently ignored.
:::

## Snapshot job

| Value | Default | Purpose |
|---|---|---|
| `snapshot.enabled` | `true` | Render `RecurringJob/default-hourly-snapshot`. |
| `snapshot.cron` | `0 * * * *` | Hourly, on the hour. |
| `snapshot.retain` | `24` | Snapshots kept per volume — a rolling 24 hours. |
| `snapshot.concurrency` | `5` | Volumes snapshotted at once. |

Snapshots are copy-on-write and in-cluster, so concurrency is cheap here relative to the
backup job.

## Backup job

| Value | Default | Purpose |
|---|---|---|
| `backup.enabled` | `true` | Render `RecurringJob/default-daily-backup`. |
| `backup.cron` | `0 3 * * *` | Nightly at 03:00. |
| `backup.retain` | `30` | Backups kept per volume — roughly a month. |
| `backup.concurrency` | `3` | Volumes streaming to S3 at once. |

`concurrency: 3` is deliberately lower than the snapshot job's: backups stream blocks off
the node, and a higher number saturates the uplink.

Both jobs target Longhorn's `default` recurring-job-group. That is not configurable — see
[Coverage model](/coverage/).

## Cluster settings

| Value | Default | Purpose |
|---|---|---|
| `clusterSettings.allowRecurringJobWhileVolumeDetached` | `true` | Let RecurringJobs auto-attach detached volumes long enough to run. |

Longhorn's own default is `false`, which silently skips detached volumes. The chart flips
it because JupyterHub user PVCs are routinely detached when servers idle out, and skipping
them defeats the point.

Set it to `null` to leave Longhorn's existing value untouched — the `Setting` resource is
then not rendered at all:

```yaml
clusterSettings:
  allowRecurringJobWhileVolumeDetached: null
```

:::caution[Cluster-wide, not chart-scoped]
This setting affects every RecurringJob on the cluster, including ones other charts or
operators created. `null` is the right choice if something else already manages it.
:::

## Metadata

| Value | Default | Purpose |
|---|---|---|
| `commonLabels` | `{}` | Extra labels merged onto every rendered resource. |
| `commonAnnotations` | `{}` | Extra annotations merged onto every rendered resource. |

Useful for cost attribution or Argo CD sync-wave hints:

```yaml
commonLabels:
  team: platform
commonAnnotations:
  argocd.argoproj.io/sync-wave: "1"
```

## Validation

Guards run at render time, included from every resource template, so bad input fails
`helm template` instead of producing a job that never fires:

| Guard | Message on failure |
|---|---|
| `snapshot.cron` is 5 whitespace-separated fields | `snapshot.cron is not a valid 5-field cron expression: "..."` |
| `backup.cron` is 5 whitespace-separated fields | `backup.cron is not a valid 5-field cron expression: "..."` |
| `snapshot.retain` > 0 | `snapshot.retain must be > 0 (got N)` |
| `backup.retain` > 0 | `backup.retain must be > 0 (got N)` |

Try it:

```bash
helm template test . --set 'snapshot.cron=not-a-cron'
helm template test . --set 'backup.retain=0'
```

The cron check is structural — five fields — not semantic. `99 * * * *` passes the guard and
is rejected later by Longhorn, so check the `RecurringJob`'s status after changing a
schedule to something unusual.

## Rendered output

Default values produce exactly three objects:

```bash
helm template backup . | grep -E '^(kind|  name):'
```

```
kind: Setting
  name: allow-recurring-job-while-volume-detached
kind: RecurringJob
  name: default-daily-backup
kind: RecurringJob
  name: default-hourly-snapshot
```

No StorageClass, no Deployment, no ServiceAccount. If you see anything else, you are
rendering a different chart.

## A worked override

```yaml
# Tighter RPO, longer history, gentler on the uplink
snapshot:
  cron: "*/15 * * * *"
  retain: 96          # still 24 hours of history
  concurrency: 5

backup:
  cron: "0 2 * * *"
  retain: 90          # a quarter
  concurrency: 2

commonLabels:
  team: platform
```
