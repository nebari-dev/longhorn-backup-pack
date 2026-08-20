---
title: Introduction
description: "Hourly Longhorn snapshots and daily S3 backups for every volume on the cluster's default StorageClass."
---

The Nebari Longhorn Backup Pack schedules Longhorn-native snapshots and backups
for the cluster's default StorageClass. It renders two `RecurringJob`s and one
cluster-wide `Setting` that lets snapshots fire on detached volumes — nothing
else.

:::note[Documentation in progress]
This site is the scaffolding for the pack's documentation. Content is being
written; until it lands here, the
[repository README](https://github.com/nebari-dev/longhorn-backup-pack#readme)
is the reference for installing and configuring the pack.
:::

## Contributing to these docs

Pages live in `docs/src/content/docs/`. See the
[docs README](https://github.com/nebari-dev/longhorn-backup-pack/blob/main/docs/README.md)
for how to run the site locally and add a page.
