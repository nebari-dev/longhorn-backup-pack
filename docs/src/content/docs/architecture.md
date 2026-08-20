---
title: Architecture
description: What the chart owns, what it deliberately leaves to the cluster, and why.
---

## Three objects

```
                    Helm release
                         │
     ┌───────────────────┼───────────────────┐
     │                   │                   │
RecurringJob        RecurringJob          Setting
default-hourly-     default-daily-        allow-recurring-job-
snapshot            backup                while-volume-detached
task: snapshot      task: backup          value: "true"
groups: [default]   groups: [default]     (cluster-wide)
```

No Deployment, no ServiceAccount, no StorageClass, no controller. Longhorn already has a
scheduler; the chart's entire job is to declare policy into it.

That is why the chart has no `appVersion` drift to worry about and no images to pin: it
ships CRs, and Longhorn does the work.

## Where the boundaries are

| Concern | Owner |
|---|---|
| *When* volumes are snapshotted and backed up | this chart |
| *Which* volumes are covered | Longhorn's `default` group, driven by StorageClass parameters |
| *Where* backups go — bucket, region, credentials | the cluster's GitOps repo, via `BackupTarget` |
| Installing Longhorn | the cluster |
| Restoring | an operator, through the Longhorn UI |

The split falls where cluster-specificity begins. Scheduling is identical everywhere, so it
belongs in a chart. Bucket URLs and credentials are not, so they belong in the
infrastructure repository — and keeping them out of chart values keeps them out of a
release's rendered manifests.

## Targeting `default`, not a dedicated group

An earlier design paired a `longhorn-jhub` StorageClass with a `jhub` recurring-job-group
and required workloads to pin that class. It worked, but every new chart was one forgotten
`storageClassName` away from unprotected volumes.

Targeting the built-in `default` group inverts the failure mode. Longhorn auto-labels any
volume whose StorageClass has no `recurringJobSelector` parameter — so protection is the
default state, and opting *out* is the deliberate act. New workloads on the cluster's
default StorageClass are covered the moment they are created.

The cost is loss of granularity: one schedule for everything. Per-workload schedules mean a
StorageClass with its own selector plus a second release, which is outside this chart's
surface by design. See [Coverage model](/coverage/).

## Why the cluster-wide `Setting` is in the chart

It sits oddly next to two neatly-scoped `RecurringJob`s — it is cluster-wide state, and the
chart says so in a comment on the template itself.

It is here because without it the other two objects do not do what their names imply.
Longhorn's stock `allow-recurring-job-while-volume-detached: false` makes RecurringJobs skip
detached volumes silently. On a JupyterHub cluster, "detached" describes every user who is
not currently logged in — which is to say, most of them, most of the time.

Shipping the schedule without the setting would be shipping a backup policy that protects
active users and quietly ignores idle ones. So the chart takes the cluster-wide edit,
documents it loudly, and offers `null` for operators who manage the setting elsewhere.

## Snapshots and backups are different tools

| | Snapshot | Backup |
|---|---|---|
| Storage | Longhorn replicas, in-cluster | S3 backupstore |
| Mechanism | copy-on-write | block-incremental upload |
| Cost | near-zero, no network | bandwidth, bucket space |
| Default cadence | hourly | nightly |
| Recovers from | user error, bad write, corruption | losing the cluster |
| Restore speed | seconds, revert in place | minutes to hours |

Two tiers exist because the cheap one cannot survive cluster loss and the durable one is
too expensive to run hourly. The hourly/nightly split is the standard compromise; both
cadences are values.

## Validation as a design choice

The validation helper is included from every resource template rather than living in a
`NOTES.txt` or a CI check, so a malformed cron or a zero retention fails
`helm template` — before Argo CD ever applies it.

The alternative is worse than it sounds: Longhorn accepts a `RecurringJob` with a bad cron,
the object exists, `kubectl get recurringjobs` looks healthy, and nothing ever runs. The
guard converts a silent no-op into a loud render failure.

## Failure modes worth knowing

The pack has three ways to look correct while protecting nothing. Each has a check in
[Operations](/operations/):

1. **Wrong namespace.** Jobs rendered outside Longhorn's namespace are ignored.
2. **Volumes outside the `default` group.** A StorageClass with its own
   `recurringJobSelector` opts its volumes out.
3. **The detached-volume setting off.** Idle volumes are skipped silently.

All three produce a healthy-looking `kubectl get recurringjobs` and no data. Verifying
protection means checking *volumes*, not jobs.

## Design history

The two dated design and implementation notes in the repository's `docs/` directory record
the original Velero-to-Longhorn migration and the `longhorn-jhub` StorageClass approach
that preceded the current one. They are kept as history — the chart has since moved to the
`default` group — and are not published as part of this site.
