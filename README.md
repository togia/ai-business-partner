# ai-business-partner

An AI agent intended to help develop business ideas over months or years, not just a single session.

## What we're trying to achieve

`ai-business-partner` is a **personal AI business-development partner** for conversations that continue over a long time horizon. It should:

- Brainstorm and challenge business ideas rather than just agreeing with them.
- Remember previous discussions and distinguish hypotheses from confirmed decisions.
- Retrieve prior research with provenance (so claims can be traced back to their source).
- Keep a living visual strategy/vision board in **Miro**.

The planned application is built from pieces that can each be swapped out independently:

- A **replaceable agent runtime** (an SDK-based agent or coding harness, e.g. Google ADK, OpenAI Agents SDK, or Claude Agent SDK) behind an application-owned adapter, so the underlying agent framework is never a dead end.
- **Graphiti** (the open-source temporal graph system underlying Zep) for evolving business entities, relationships, and historical facts, with a local **FalkorDB** graph database as a possible low-cost backend.
- **Mem0** — *optional, not yet installed* — for personalized long-term memory such as user preferences and durable goals. Start with Graphiti plus a simple preference store; add Mem0 only if testing shows a real benefit.
- **SQLite** as a framework-independent, append-only conversation/event archive and explicit business-decision record. Raw source history is preserved so any derived memory can be rebuilt from scratch if a memory engine changes.
- **Miro via MCP** for creating and updating the living visual business board.
- **Local files and Git** for documents, research, and configuration templates/source code.

## Why a VPS with backups instead of AWS

The design deliberately avoids AWS and other managed-cloud database costs. Running an agent runtime plus a graph database plus a conversation archive on managed services (RDS, managed graph DBs, EBS snapshots, etc.) adds up quickly for a single-user, long-running personal project — and most of that spend would go toward availability guarantees this project doesn't need yet.

Instead, everything except model APIs, Miro, and off-site storage runs on **one Contabo Ubuntu VPS**. The VPS gets its durability not from expensive managed infrastructure, but from **encrypted, versioned, off-site backups** (Restic snapshots in a private Backblaze B2 bucket, run on a schedule by systemd). This gets most of the practical safety of a managed service — the ability to recover from a lost or corrupted host — at a small fraction of the cost, while keeping agent runtimes and memory engines replaceable without losing the project's data.

**This README currently covers provisioning, access, and backups — not installing the agent, Graphiti, FalkorDB, Mem0, or Miro yet.** The VPS and backup commands below are a reproducible guide, and also a record of what can be inferred from previously supplied shell history. A command appearing in history does *not* prove it completed successfully.

## Backup objective

Protect the business knowledge stored on the VPS with **encrypted off-site Restic snapshots in a private Backblaze B2 bucket**. A systemd timer runs every six hours. Contabo's paid auto-backup, private networking, and object storage add-ons are not necessary for this initial single-host deployment.

> **Scope:** Restic currently backs up files at the specified paths. Until database-consistent SQLite and FalkorDB snapshots are added and tested, a successful Restic run does **not** establish that all future live databases are recoverable.

## Architecture

```text
Your computer --SSH--> Contabo Ubuntu VPS
                            |
                            +-- /srv/ai-business-partner/       application/files
                            +-- /var/backups/ai-business-partner/  database exports (later)
                            |
                            +-- systemd timer (every 6 hours)
                                  -> Restic (encrypt + deduplicate)
                                  -> Backblaze B2 private bucket
```

Restic creates encrypted, incremental, versioned snapshots. Backblaze B2 supplies independent storage. Restic is **not** a scheduler: systemd runs it on a schedule.

## Status inferred from shell history

The supplied history records the following actions, **not a complete verified inventory**:

| Area | Evidence in history | What remains to verify |
|---|---|---|
| Restic configuration | `/etc/restic/env` sourced and inspected; password file read | Correct exported variables, safely stored password, key permissions |
| Repository | `restic init` attempted multiple times; `restic snapshots` run | Repository opens with the saved password; do not reinitialize an existing repository |
| Backup | `/srv/ai-business-partner/test.txt` backed up repeatedly, including a timestamped test with `--tag ai-business-partner` | Confirm the current file matches the specific snapshot using `restic dump` |
| Restore | Snapshot `71561da1` was dumped and restored to `/tmp/ai-business-partner-restore` | Confirm restored bytes matched the timestamped live file; history does not include command output |
| Automation | `/usr/local/sbin/ai-business-partner`, `.service`, `.timer` edited; timer enabled; service started | Inspect contents, service exit code, recent journal, latest successful scheduled snapshot |
| Databases | No installation or database-snapshot commands shown | Add and test consistent exports when SQLite/FalkorDB are deployed |

### Why earlier restore attempts looked stale

Some prior test writes targeted a **different directory** from the directory passed to `restic backup`. Therefore a backup of `/srv/ai-business-partner` would not include those test-file changes. In addition, `restic restore latest` can select a different snapshot than intended when several snapshots exist. Use **one canonical path** and an **explicit snapshot ID** for verification, as shown in section 5.

Avoid treating either an old restored string or a successful `systemctl enable` as proof that the newest data is protected. Do not remove existing snapshots or files until you have tested recovery.

## 1. Provision the Contabo VPS

1. Sign in at [Contabo](https://contabo.com/), choose a VPS with **at least 8 GB RAM** as a starting point for a future agent + Graphiti + graph database. Check the live checkout price, renewal price, billing period, and setup fees.
2. Select an EU location if appropriate, **Ubuntu 24.04 LTS**, and an x86-64 instance if you want the simplest Docker-image compatibility.
3. A public IPv4 address is useful for initial SSH access. You can **skip** paid private networking, object storage, and provider auto-backup for this single-VPS design, *provided independent backups are configured and tested*.
4. Set a strong account password and enable account MFA. Add an SSH public key at provisioning if the interface permits.
5. When the server is ready, save its public IP address and initial access details securely.

No cost shown here is guaranteed; verify current pricing and availability before purchase.

## 2. Connect using SSH

On **your local computer**, generate a key if you don't have one:

```bash
ssh-keygen -t ed25519 -C "ai-business-partner"
```

Copy the contents of `~/.ssh/id_ed25519.pub` (the **public** key, never the private key) into Contabo's provisioning interface if supported. Then connect using the username provided by Contabo; for an initial root account:

```bash
ssh root@YOUR_VPS_IP
```

If the key was not installed during provisioning, first connect using the credentials Contabo supplied, then add your public key using `ssh-copy-id` from your local computer (or follow Contabo's manual key instructions):

```bash
ssh-copy-id root@YOUR_VPS_IP
```

### Basic server security

On the VPS, update packages:

```bash
apt update && apt upgrade -y
apt install -y sudo ufw
```

Create a non-root administrative account (replace `adminuser` with your preferred username):

```bash
adduser adminuser
usermod -aG sudo adminuser
```

From **your local computer**, copy your SSH public key to the new user (while root login still works):

```bash
ssh-copy-id adminuser@YOUR_VPS_IP
ssh adminuser@YOUR_VPS_IP
```

Confirm that a **new terminal session** can log in using the key and run `sudo` before changing SSH settings. On the VPS:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

Only **after testing key-based login for the new user**, disable password and root SSH logins using a drop-in file:

```bash
sudo tee /etc/ssh/sshd_config.d/99-ai-business-partner.conf >/dev/null <<'SSHEOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
SSHEOF
sudo sshd -t
sudo systemctl reload ssh
```

Keep your existing SSH session open until you've confirmed a fresh login works. If your SSH service is on a nonstandard port, allow that port in UFW **before** enabling the firewall.

## 3. Create a Backblaze B2 bucket and application key

1. Sign in to [Backblaze](https://secure.backblaze.com/) and enable **B2 Cloud Storage** if needed.
2. Go to **B2 Cloud Storage → Buckets → Create a Bucket**.
3. Use a globally unique name based on `ai-business-partner-backup` (for example, `ai-business-partner-backup-UNIQUE`). Choose **Private**. For the first setup, leave Object Lock/lifecycle rules unset until you understand how they interact with backup retention.
4. Note the **S3 endpoint** shown for the bucket, for example `s3.eu-central-003.backblazeb2.com`. Use the exact endpoint from Backblaze.
5. Go to **Application Keys → Add a New Application Key**. Name it `ai-business-partner-restic`, restrict it to the backup bucket, grant the access required to read and write Restic repository objects, and enable bucket listing if needed by your tooling.
6. Save both values immediately in your password manager: **`keyID`** (access key ID) and **`applicationKey`** (secret access key). The secret is only shown when created. Do **not** use the master application key.

For stronger protection, later consider separate restricted credentials and an immutable/versioned protection strategy. An attacker who gains the active backup credential may be able to damage backups depending on its permissions.

## 4. Install Restic on the VPS

Log in using your admin account and run:

```bash
sudo apt update
sudo apt install -y restic sqlite3 openssl util-linux
restic version
sudo install -d -m 700 /etc/restic
sudo install -d -m 700 /var/backups/ai-business-partner
sudo install -d -m 755 /srv/ai-business-partner
```

### Configure credentials

Open a file as root:

```bash
sudo nano /etc/restic/env
```

Paste the following, replacing all placeholders. **Never paste the resulting file into a chat or commit it to Git.**

```bash
export AWS_ACCESS_KEY_ID='YOUR_B2_KEY_ID'
export AWS_SECRET_ACCESS_KEY='YOUR_B2_APPLICATION_KEY'
export RESTIC_REPOSITORY='s3:https://YOUR_S3_ENDPOINT/YOUR_BUCKET_NAME'
export RESTIC_PASSWORD_FILE='/etc/restic/password'
```

For example, a repository URL might look like:

```text
s3:https://s3.eu-central-003.backblazeb2.com/ai-business-partner-backup-UNIQUE
```

Restrict access:

```bash
sudo chown root:root /etc/restic/env
sudo chmod 600 /etc/restic/env
```

Generate a **separate** encryption password for Restic:

```bash
sudo sh -c 'umask 077; openssl rand -base64 48 > /etc/restic/password'
sudo chown root:root /etc/restic/password
sudo chmod 600 /etc/restic/password
```

Save the exact contents of `/etc/restic/password` in a secure password manager **outside the VPS**, along with the B2 credentials and repository URL. Without this password, encrypted backups cannot be recovered. Avoid placing secrets on the command line, in screenshots, or in shell history.

### Verify configuration without revealing secrets

```bash
sudo bash -c 'source /etc/restic/env; test -n "$RESTIC_REPOSITORY" && echo "Repository configured"'
```

### Initialize **once**

```bash
sudo bash -c 'source /etc/restic/env; restic init'
```

If Restic reports **`repository master key and config already initialized`**, that repository already exists. **Do not initialize again or delete it.** Verify access instead:

```bash
sudo bash -c 'source /etc/restic/env; restic snapshots'
```

If Restic says the repository location is missing, check that `/etc/restic/env` contains an **exported** `RESTIC_REPOSITORY` and that you used `source /etc/restic/env` in the same shell as the Restic command.

## 5. Make and verify a test backup

Write a timestamped test file:

```bash
printf 'BACKUP TEST %s\n' "$(date -u +%Y%m%dT%H%M%SZ)" | sudo tee /srv/ai-business-partner/test.txt
sudo cat /srv/ai-business-partner/test.txt
```

Back it up:

```bash
sudo bash -c 'source /etc/restic/env; restic backup /srv/ai-business-partner --tag ai-business-partner'
```

Note the **snapshot ID** printed at the end. List snapshots:

```bash
sudo bash -c 'source /etc/restic/env; restic snapshots --path /srv/ai-business-partner'
```

Read the test file **directly from that exact snapshot** (replace `SNAPSHOT_ID`):

```bash
sudo bash -c 'source /etc/restic/env; restic dump SNAPSHOT_ID /srv/ai-business-partner/test.txt'
```

Its contents should match the live file. Use an explicit snapshot ID rather than `latest` if other hosts, backup paths, or earlier tests exist in the same repository.

### Test a restore into a separate directory

```bash
sudo mkdir -p /tmp/ai-business-partner-restore
sudo bash -c 'source /etc/restic/env; restic restore SNAPSHOT_ID --target /tmp/ai-business-partner-restore'
sudo cat /tmp/ai-business-partner-restore/srv/ai-business-partner/test.txt
```

The restored content should match. Do **not** restore over your live application directory during testing.

Run a repository-structure check:

```bash
sudo bash -c 'source /etc/restic/env; restic check'
```

This checks repository structure; a more thorough check can use `restic check --read-data` but downloads and verifies more data.

## 6. Inspect and verify the existing six-hour backup automation

Shell history shows that these exact paths were **already created or edited**, with the corresponding timer/service enabled and started:

- Script: `/usr/local/sbin/ai-business-partner`
- Service: `/etc/systemd/system/ai-business-partner.service`
- Timer: `/etc/systemd/system/ai-business-partner.timer`

**Inspect these first; do not create differently named duplicates.** Their contents cannot be reconstructed from command history alone. In particular, the timer running does not prove its service points to a valid backup script or that the script backs up the intended directory.

```bash
sudo systemctl cat ai-business-partner.service
sudo systemctl cat ai-business-partner.timer
sudo sed -n '1,180p' /usr/local/sbin/ai-business-partner
systemctl list-timers --all ai-business-partner.timer
sudo journalctl -u ai-business-partner.service -n 60 --no-pager
```

Never publish output containing credentials. If you want to standardize the content **after inspecting it**, use the following configuration, keeping the already-used filenames.

### Backup script: `/usr/local/sbin/ai-business-partner`

```bash
sudo nano /usr/local/sbin/ai-business-partner
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
source /etc/restic/env

exec 9>/run/lock/ai-business-partner.lock
flock -n 9 || { echo 'Backup already running' >&2; exit 1; }

# TODO before production: export live SQLite and FalkorDB into
# /var/backups/ai-business-partner using DB-consistent mechanisms.
restic backup \
  /srv/ai-business-partner \
  /var/backups/ai-business-partner \
  --tag ai-business-partner
```

```bash
sudo chown root:root /usr/local/sbin/ai-business-partner
sudo chmod 700 /usr/local/sbin/ai-business-partner
sudo install -d -m 700 /var/backups/ai-business-partner
```

### Service: `/etc/systemd/system/ai-business-partner.service`

```ini
[Unit]
Description=ai-business-partner encrypted off-site backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/ai-business-partner
TimeoutStartSec=2h
```

### Timer: `/etc/systemd/system/ai-business-partner.timer`

```ini
[Unit]
Description=Back up ai-business-partner every six hours

[Timer]
OnCalendar=*-*-* 00,06,12,18:00:00
Persistent=true
RandomizedDelaySec=5min
Unit=ai-business-partner.service

[Install]
WantedBy=timers.target
```

Create/update the service and timer files with `sudo nano` only if their inspected contents need correction. Then test:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ai-business-partner.timer
sudo systemctl start ai-business-partner.service
sudo systemctl status ai-business-partner.service --no-pager
systemctl list-timers --all ai-business-partner.timer
sudo journalctl -u ai-business-partner.service -n 60 --no-pager
sudo bash -c 'source /etc/restic/env; restic snapshots --path /srv/ai-business-partner'
```

The service is a `oneshot`, so **`inactive (dead)` after a successful run is normal**; verify its exit status, journal, and newly created snapshot. A six-hour schedule can lose up to six hours of data, or more if the service fails; add external failure alerts.

## 7. Databases: critical follow-up before real use

Do **not** assume Restic can safely copy the live files of SQLite or FalkorDB while they are changing.

- **SQLite:** Create a consistent database copy using SQLite's online backup API (`sqlite3 SOURCE '.backup DEST'`) or the application's backup mechanism. Write copies into `/var/backups/ai-business-partner/` before invoking Restic. Include all actual databases and test opening the restored copies.
- **FalkorDB:** Enable and verify supported persistence, then use a database-supported snapshot/export procedure. Test restoring the graph, not just finding files in the snapshot. The procedure depends on the deployed version and persistence configuration.
- **Agent workspace:** Confirm every durable data directory and mounted Docker volume is covered. Keep source code and deployment configuration in version control separately from encrypted backups.
- **Secrets:** Back up necessary configuration carefully, but avoid putting active API keys or encryption passwords inside routinely restored application data. Document how credentials will be recreated after a disaster.

The script in section 6 is an **initial files-only backup** until these pre-backup database steps are added.

## 8. Retention, security, and ongoing checks

Start with backups and restore testing. Only then configure a retention policy, for example 30 daily and 6 monthly snapshots. Pruning deletes backup data and should be treated as a privileged operation.

Example **manual** retention command (run only once you understand retention and have confirmed backups are healthy):

```bash
sudo bash -c 'source /etc/restic/env; restic forget --tag ai-business-partner --keep-daily 30 --keep-monthly 6 --prune'
```

If you have multiple independent backup groups, explicitly scope retention to the correct host/paths/tags before running it. If using Backblaze lifecycle rules, make sure they do **not** delete Restic repository objects behind Restic's back.

Suggested routine:

| Frequency | Action |
|---|---|
| Every 6 hours | Scheduled Restic backup |
| Daily | Alert if no successful backup within expected window |
| Weekly | Review snapshots, storage usage, and backup logs |
| Monthly | `restic check` and test restoration of representative files/databases |
| Periodically | Full disaster-recovery drill onto another machine |

Keep the repository encryption password and B2 credentials in a password manager independent of Contabo. Consider a separate backup account, strong MFA, restricted key permissions, and tamper-resistant retention as the project grows.

## 9. Recovery if Contabo is lost

1. Provision any replacement Linux VPS (or use your own computer).
2. Install Restic and recreate `/etc/restic/env` and `/etc/restic/password` using **your independently saved secrets**.
3. Verify access with `restic snapshots`.
4. Restore a selected snapshot into a **temporary** directory:

   ```bash
   sudo bash -c 'source /etc/restic/env; restic restore SNAPSHOT_ID --target /tmp/ai-business-partner-recovery'
   ```

5. Inspect recovered files and validate database snapshots before placing them into production paths.
6. Recreate Docker/app services, reissue credentials as necessary, and reconnect Miro and other tools.

Reproducible deployment configuration plus **tested database backups** is what makes VPS migration feasible — and is the main reason the project can safely stay off AWS: the exit path (recreate on any other host) doesn't depend on any AWS-specific service.

## 10. Troubleshooting

| Symptom | Check |
|---|---|
| `Please specify repository location` | Load `/etc/restic/env` using `source` in the same shell; verify `RESTIC_REPOSITORY` is exported |
| `repository master key and config already initialized` | Repository exists; use `restic snapshots`, do not reinitialize |
| `wrong password` | Check saved Restic password and repository URL; do not delete the repository |
| `AccessDenied` | Bucket restrictions, read/write permissions, key ID, and S3 endpoint |
| S3 signature error | Correct secret, endpoint, and region; avoid accidental whitespace |
| Restoring old text | Choose the exact new snapshot ID, then use `restic dump SNAPSHOT_ID /srv/ai-business-partner/test.txt` |
| Timer not running | `systemctl list-timers --all` and `journalctl -u ai-business-partner.service` |
| Backup succeeds but DB cannot open | Implement and test database-consistent snapshot procedures |

## Official references

- [Contabo: SSH setup](https://help.contabo.com/en/support/solutions/articles/103000271398-how-do-i-setup-a-ssh-connection-)
- [Contabo: Ubuntu server initial setup](https://contabo.com/blog/ubuntu-server-operating-system-setup-and-benefits/)
- [Backblaze: Restic with B2](https://www.backblaze.com/docs/cloud-storage-integrate-restic-with-backblaze-b2)
- [Backblaze: S3-compatible API](https://www.backblaze.com/apidocs/introduction-to-the-s3-compatible-api)
- [Restic: Backup and scheduling documentation](https://restic.readthedocs.io/en/stable/040_backup.html)

---

**Project naming convention:** Use `ai-business-partner` consistently in directory paths, script names, systemd units, tags, and documentation.
