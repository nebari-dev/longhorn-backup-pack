---
title: Backup target
description: Configuring the S3 destination Longhorn streams backups to — the piece this chart deliberately does not own.
---

The chart schedules backups. It does not configure where they go. That split is
intentional: scheduling is the same on every cluster, while the bucket, region, and
credentials are not.

Without a working `BackupTarget`, hourly snapshots still succeed — they never leave the
cluster — but every nightly backup fails.

## What is needed

A Longhorn `BackupTarget` named `default`, with:

- a **`backupTargetURL`** of the form `s3://<bucket>@<region>/<optional-prefix>`
- a **`credentialSecret`** naming a Secret in the Longhorn namespace

Longhorn's [Set Backup Target](https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/set-backup-target/)
guide is the canonical reference. In outline:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-backup-credentials
  namespace: longhorn-system
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "..."
  AWS_SECRET_ACCESS_KEY: "..."
  # For S3-compatible storage (MinIO, Hetzner, Backblaze):
  AWS_ENDPOINTS: "https://s3.example.com"
---
apiVersion: longhorn.io/v1beta2
kind: BackupTarget
metadata:
  name: default
  namespace: longhorn-system
spec:
  backupTargetURL: s3://my-longhorn-backups@us-east-1/
  credentialSecret: longhorn-backup-credentials
  pollInterval: 5m0s
```

On a GitOps-managed cluster both belong in the infrastructure repository alongside the Argo
CD `Application` that installs this chart — not in this chart's values, where the
credentials would end up in git.

## Verifying it works

```bash
kubectl -n longhorn-system get backuptargets.longhorn.io default -o yaml
```

Look at `status.available` and `status.conditions`. An unreachable bucket or bad
credentials show up here rather than in the RecurringJob.

The Longhorn UI reports the same thing under **Setting → Backup Target**, usually with a
more legible error.

## The poll interval

`pollInterval` (default 5 minutes) controls how often Longhorn re-reads the bucket to
discover backups it did not create. It has no effect on backups *this* cluster takes —
those appear immediately.

It matters in exactly one situation: a fresh Longhorn install pointed at an existing
bucket, where nothing shows up until the first poll completes. See
[Disaster recovery](/disaster-recovery/).

## Bandwidth and the nightly window

The daily backup defaults to `concurrency: 3` — at most three volumes streaming at once —
specifically so the job does not saturate the node uplink. On a cluster with many large
volumes and a modest uplink, the window can still run long.

Two levers:

- Lower `backup.concurrency` to reduce peak bandwidth, at the cost of a longer window.
- Move `backup.cron` to a quieter hour.

Longhorn backups are incremental after the first one, so the initial night is by far the
heaviest.

## Retention and bucket growth

`backup.retain: 30` keeps 30 backups **per volume**. Because backups are block-incremental,
30 days of a slowly-changing volume costs far less than 30 full copies — but the bucket
still grows with volume count and churn.

Longhorn prunes according to `retain` on its own. Deleting objects out of the bucket by
hand corrupts the block store; use the Longhorn UI or the `Backup` CRs instead.

## Locks

Longhorn coordinates concurrent access through lock objects under `backupstore/lock/` in
the bucket. If a cluster dies mid-backup, a stale lock can persist: reads keep working, new
writes complain until it is removed (`mc rm`, or the equivalent for your client).

:::caution[Never point two live clusters at one backupstore]
Two clusters writing to the same `backupstore/` prefix corrupt each other's metadata. This
is the sharpest edge in a disaster-recovery scenario, where the "destroyed" cluster may
come back to life — rotate the bucket prefix or the credential before declaring a new
cluster authoritative. See [Disaster recovery](/disaster-recovery/).
:::
