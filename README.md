# Backup Servers → Cloudflare R2 (n8n workflow)

Weekly automated backup pipeline: two source servers → a Docker host running
n8n and rclone → Cloudflare R2. Each host's backup runs independently, with
Telegram notifications on success (including bytes uploaded and total size
in R2) and on failure.

## How it works

For each host:

1. **Mount** — SSHes into the *source* host and mounts its local backup
   drive read-only by UUID at `/mnt/backup-sync` (idempotent: skips if
   already mounted, so a re-run never double-mounts).
2. **Start sync** — SSHes into the *Docker host* and launches a detached
   (`docker run -d`) rclone container that syncs the source host's
   `/mnt/backup-sync/...` folder (read over SFTP) to an R2 bucket. Detached
   on purpose: once launched, nothing stays attached to the container, so
   restarting n8n (e.g. an unattended Watchtower update) can't interrupt an
   in-progress transfer.
3. **Wait for sync** — `docker wait`s for that container, dumps its logs,
   removes it, and exits with its real exit code — so the rest of the
   workflow sees the transfer's actual result even though it ran detached.
4. **Unmount** — unmounts the source host's backup drive.
5. **Check Size** — queries the R2 bucket's total size for this host via
   `rclone size --json`.
6. **Combine Result** — a Code node that determines success/failure and
   formats human-readable byte counts (uploaded this run, total in R2).
7. **IF → Notify / Alert** — sends a Telegram message either way.

## Setup

### 1. n8n credentials (create these first, in n8n's credential store)

| Credential | Type | Used by |
|---|---|---|
| SSH: host-a | SSH Private Key | Mount/Unmount nodes for host A |
| SSH: host-b | SSH Private Key | Mount/Unmount nodes for host B |
| SSH: docker host | SSH Private Key | Start sync / Wait for sync / Check Size nodes (all run on the Docker host) |
| Telegram account | Telegram API | Notify/Alert nodes |

n8n never stores these secret values in the workflow JSON — only an internal
credential id + label. That's why this file is safe to publish, but you
still need to create the four credentials above yourself and re-point each
placeholder node at them after import (`REPLACE_ME_1`, `REPLACE_ME_2`,
`REPLACE_ME_3`, `REPLACE_ME_TG`).

### 2. Import

Import `startos-backup-to-r2-TEMPLATE.json` into n8n, then:

- Re-point every SSH node's credential (see table above).
- Re-point both Telegram nodes' credential.
- In both `Notify:` and `ALERT:` nodes, replace
  `REPLACE_WITH_YOUR_TELEGRAM_CHAT_ID` with your chat ID (message
  [@userinfobot](https://t.me/userinfobot) on Telegram to get yours).
- In both `Mount:` nodes, replace `REPLACE_WITH_DISK_UUID_HOST_A` /
  `_HOST_B` with the real UUID of each source host's backup drive
  (`lsblk -f` on that host, or `blkid`).
- In `build_workflow_public.py`, `R2_BUCKET`, the two `sftp_remote` values,
  and `dest_folder` values are placeholders (`host-a`, `host-b`,
  `REPLACE_WITH_YOUR_R2_BUCKET`) — either edit the script and regenerate,
  or just edit the relevant SSH node commands directly in the n8n UI to
  match your own rclone remote names and bucket.

### 3. On the Docker host (outside n8n — do NOT commit any of this)

- `/opt/rclone/rclone.conf` — holds your R2 access key/secret (as the `r2`
  remote) and your SFTP remotes' pinned host keys. **Never commit this
  file.**
- `/opt/rclone/.env` — env vars passed to the rclone containers (also holds
  R2 credentials if you split them out this way). **Never commit this
  file either.** If you want an example in the repo, commit a
  `.env.example` with placeholder values and add the real `.env` to
  `.gitignore`.
- Pin each SFTP remote's host key once (silences rclone's "no host key
  validation" notice and makes future connections actually verify the
  host, instead of just trusting it — worth doing before your first real
  run):

  ```bash
  docker run --rm --env-file /opt/rclone/.env -v /opt/rclone:/config/rclone \
    rclone/rclone:latest --sftp-pin-host-key lsd <your-remote-name>-sftp:
  ```

### 4. `.gitignore` suggestion

```
/opt/rclone/.env
/opt/rclone/rclone.conf
*.env
```

(Adjust paths if you're committing this from inside `/opt/rclone` itself
rather than a separate repo — the point is: the workflow JSON is safe to
publish, the rclone config/env files next to it on the Docker host are not.)
