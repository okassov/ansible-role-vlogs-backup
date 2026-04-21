# ansible-role-vlogs-backup

Ansible role to back up [VictoriaLogs](https://docs.victoriametrics.com/victorialogs/) per-day partitions to any S3-compatible object storage via [`rclone`](https://rclone.org/). Designed for long-term archival (e.g. 5-year retention managed via S3 lifecycle expiration).

## What it does

- Installs `rclone` and `curl`.
- Renders `/etc/rclone/rclone.conf` with S3 credentials (mode `0600`).
- Drops two bash scripts in `/usr/local/sbin/`:
  - `vlogs-backup-daily.sh` — `rclone sync`s yesterday's partition from `<data_dir>/partitions/YYYYMMDD/` to `s3://<bucket>/<prefix>/YYYYMMDD/`. Closed (non-current) partitions are immutable, so syncing them directly from disk is safe.
  - `vlogs-backup-reconcile.sh` — walks all closed partitions on disk, checks if each one is present in S3, and back-fills any missing partition.
- Installs and enables two systemd timers:
  - `vlogs-backup-daily.timer` — daily at 03:00 (UTC of the host).
  - `vlogs-backup-reconcile.timer` — monthly on the 1st at 04:00.

## Requirements

- Ubuntu 22.04+ / Debian 12+ (`apt` package `rclone` available).
- VictoriaLogs reachable on the same host (default `http://127.0.0.1:9428`).
- S3 bucket already created. Lifecycle expiration (e.g. 1825 days for 5-year retention) configured by you on the bucket — the role does not manage bucket lifecycle.

## Role variables

See [`defaults/main.yaml`](defaults/main.yaml) for the full list. Most important:

| Variable | Default | Description |
|---|---|---|
| `vlogs_backup_data_dir` | `/var/lib/victoria-logs` | VictoriaLogs `storageDataPath`. |
| `vlogs_backup_s3_endpoint` | `https://object.pscloud.io` | S3 endpoint. |
| `vlogs_backup_s3_region` | `kz-ala-1` | S3 region. |
| `vlogs_backup_s3_bucket` | `mycar-vlogs-backup` | Bucket name. |
| `vlogs_backup_s3_prefix` | `vlogs/partitions` | Key prefix inside the bucket. |
| `vlogs_backup_s3_access_key` | `""` | **Required.** S3 access key. |
| `vlogs_backup_s3_secret_key` | `""` | **Required.** S3 secret key. |
| `vlogs_backup_daily_schedule` | `*-*-* 03:00:00` | systemd `OnCalendar` for daily timer. |
| `vlogs_backup_reconcile_schedule` | `*-*-01 04:00:00` | systemd `OnCalendar` for reconcile timer. |
| `vlogs_backup_daily_lag_days` | `1` | Which day to back up (1 = yesterday). |
| `vlogs_backup_telegram_bot_token` | `""` | Telegram bot token. Empty = notifications disabled. |
| `vlogs_backup_telegram_chat_id` | `""` | Telegram chat / channel id (numeric, can be negative for groups). |
| `vlogs_backup_telegram_notify_on_success` | `true` | If `false`, only failures send a message. |

## Example usage

`requirements.yaml`:

```yaml
roles:
  - name: vlogs_backup
    scm: git
    src: https://github.com/okassov/ansible-role-vlogs-backup.git
    version: v0.1.0
```

Playbook:

```yaml
- hosts: vlogs
  roles:
    - role: vlogs_backup
```

`group_vars/vlogs.yaml`:

```yaml
vlogs_backup_s3_access_key: "..."
vlogs_backup_s3_secret_key: "..."
```

## Restore

To restore an old partition:

```bash
sudo systemctl stop victoria-logs-single
sudo rclone copy --config=/etc/rclone/rclone.conf \
  pscloud:<bucket>/vlogs/partitions/YYYYMMDD/ \
  /var/lib/victorialogs/partitions/YYYYMMDD/
sudo chown -R victorialogs:victorialogs /var/lib/victorialogs/partitions/
sudo systemctl start victoria-logs-single
```

## License

MIT
