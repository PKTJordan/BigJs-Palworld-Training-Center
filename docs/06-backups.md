# 06. Backups

**TL;DR:** the Docker image writes save tarballs to disk on a schedule. A systemd timer on the host uploads each new tarball to GitHub Releases as your off-box vault. A second timer prunes old releases so the repo doesn't grow forever. Restore is a one-liner from a fresh box.

## Why GitHub Releases

Palworld saves are binary blobs that get overwritten every few minutes. Committing them to git bloats the repo forever. Releases are GitHub's built-in "big files" side door. They don't count against repo size. Free. 2 GB per asset. Unlimited count. You already have GitHub.

Git LFS is the wrong tool here. Its free tier is 1 GB storage and 1 GB egress per month. You'd blow through that in a day.

## Two stages

**Stage 1: the container writes tarballs to disk.** The `thijsvanloef` image handles this via env vars in your compose file:

```yaml
environment:
  BACKUP_ENABLED: "true"
  BACKUP_CRON_EXPRESSION: "0 */12 * * *"     # every 12h at 00:00 and 12:00 UTC
  DELETE_OLD_BACKUPS: "true"
  OLD_BACKUP_DAYS: "7"                        # local disk retention
```

Tarballs land in `./palworld/backups/` inside the bind mount, so on the host they're at `~/palworld-server/palworld/backups/`. Container prunes anything older than 7 days on its own.

If your friend group is chaotic, tighten the cron. `0 */6 * * *` for every 6 hours. Every tarball is roughly your save size (grows with base count and Pal storage), typically 1 to 10 MB early on, larger later.

**Stage 2: a systemd timer uploads to GitHub Releases.** This runs on the host, not in the container.

## Setup

### Fine-grained PAT

At https://github.com/settings/tokens?type=beta, create a token:

- Repository access: only this repo (whatever you named your private-or-public backup destination)
- Permissions: Contents = Read + Write, Metadata = Read
- Expiration: whatever you can live with (12 months is generous)

Copy it once. GitHub only shows it that time.

### Secrets file

On the VPS, out of the container:

```bash
sudo mkdir -p /etc/palworld-backup
sudo tee /etc/palworld-backup/.env <<'EOF'
GH_TOKEN=github_pat_XXXXXXXXXXXXXXXXXXXX
REPO=<your-github-user>/<your-backup-repo>
BACKUP_DIR=/home/<your-user>/palworld-server/palworld/backups
KEEP_RELEASES=14
EOF
sudo chmod 640 /etc/palworld-backup/.env
sudo chown root:<your-group> /etc/palworld-backup/.env
```

`KEEP_RELEASES=14` at a 12-hour backup cadence gives you 7 days of rolling history. Adjust to taste.

### Push script

`/usr/local/bin/palworld-backup-push.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/palworld-backup/.env

[ -d "$BACKUP_DIR" ] || { echo "backup dir not found: $BACKUP_DIR" >&2; exit 1; }

LATEST=$(find "$BACKUP_DIR" -maxdepth 1 -name '*.tar.gz' -printf '%T@ %p\n' \
  | sort -rn | head -n1 | cut -d' ' -f2- || true)
[ -n "${LATEST:-}" ] || { echo "no tarball yet"; exit 0; }

NAME=$(basename "$LATEST")
TAG="save-${NAME%.tar.gz}"

# skip if already pushed
if GH_TOKEN="$GH_TOKEN" gh release view "$TAG" --repo "$REPO" >/dev/null 2>&1; then
  echo "already pushed: $TAG"
  exit 0
fi

SIZE=$(stat -c%s "$LATEST" | numfmt --to=iec)
echo "pushing $NAME ($SIZE) as $TAG"

GH_TOKEN="$GH_TOKEN" gh release create "$TAG" "$LATEST" \
  --repo "$REPO" \
  --title "$TAG" \
  --notes "auto backup · $NAME · $SIZE · $(hostname)"
```

Make it executable: `sudo chmod +x /usr/local/bin/palworld-backup-push.sh`.

### Retention script

`/usr/local/bin/palworld-backup-retention.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/palworld-backup/.env

TAGS=$(GH_TOKEN="$GH_TOKEN" gh release list --repo "$REPO" -L 500 \
  --json tagName,createdAt \
  --jq "sort_by(.createdAt) | reverse | .[${KEEP_RELEASES}:] | .[].tagName")

for T in $TAGS; do
  echo "deleting $T"
  GH_TOKEN="$GH_TOKEN" gh release delete "$T" --repo "$REPO" --yes --cleanup-tag
done
```

`sudo chmod +x /usr/local/bin/palworld-backup-retention.sh`.

### Systemd timers

`/etc/systemd/system/palworld-backup.service`:

```ini
[Unit]
Description=Push newest Palworld backup to GitHub Releases

[Service]
Type=oneshot
ExecStart=/usr/local/bin/palworld-backup-push.sh
```

`/etc/systemd/system/palworld-backup.timer`:

```ini
[Unit]
Description=Palworld backup push (every 30 min)

[Timer]
OnBootSec=5min
OnUnitActiveSec=30min
Persistent=true

[Install]
WantedBy=timers.target
```

Same pattern for retention, but daily:

`/etc/systemd/system/palworld-backup-retention.service`:

```ini
[Unit]
Description=Palworld backup retention prune

[Service]
Type=oneshot
ExecStart=/usr/local/bin/palworld-backup-retention.sh
```

`/etc/systemd/system/palworld-backup-retention.timer`:

```ini
[Unit]
Description=Palworld backup retention (nightly)

[Timer]
OnCalendar=daily
RandomizedDelaySec=30min
Persistent=true

[Install]
WantedBy=timers.target
```

Enable both:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now palworld-backup.timer
sudo systemctl enable --now palworld-backup-retention.timer
```

## Verify it works

Wait for the first backup cron inside the container to fire (up to 12 hours). Then either wait 30 minutes for the push timer or trigger it manually:

```bash
sudo systemctl start palworld-backup.service
sudo journalctl -u palworld-backup.service --since '5 minutes ago'
```

You should see `pushing <name> as save-<...>`. Then on GitHub: your repo's Releases tab lists the new tag with the tarball attached.

## Does this kick players

No. Both stages are pure I/O.

- Stage 1: the image calls REST `/save` (no restart), then tars `Pal/Saved/` in place.
- Stage 2: `gh release create` uploads an already-written file from the host. Docker never sees the call.

Both run during active play with zero impact.

## Restore on a fresh box

Assuming a fresh VPS with Docker and gh CLI installed, `gh auth login` done or `GH_TOKEN` exported:

```bash
source /etc/palworld-backup/.env
export GH_TOKEN
LATEST_TAG=$(gh release list --repo "$REPO" -L 1 --json tagName -q '.[0].tagName')
gh release download "$LATEST_TAG" --repo "$REPO" --pattern '*.tar.gz' --dir /tmp

cd ~/palworld-server && docker compose down
cd palworld/Pal && sudo rm -rf Saved
sudo tar xzf /tmp/*.tar.gz
sudo chown -R 1000:1000 ~/palworld-server/palworld/
cd ~/palworld-server && docker compose up -d
docker compose logs -f
```

Restore to a specific point in time:

```bash
gh release list --repo "$REPO" -L 50    # pick a tag by timestamp
gh release download <tag> --repo "$REPO" --pattern '*.tar.gz' --dir /tmp
# then same tar / chown / up dance
```

Palworld boots, reads the world, you're back where the backup was taken.

## What can go wrong

- **`gh: command not found`.** Install it: `sudo apt install gh`. The systemd unit runs as root, so gh needs to be on root's PATH.
- **`HTTP 401` from gh.** PAT expired or missing scopes. Regenerate with Contents R+W, update `/etc/palworld-backup/.env`.
- **No tarballs appearing in `BACKUP_DIR`.** Check the container: `docker logs palworld-server 2>&1 | grep -i backup`. Common causes: `BACKUP_ENABLED` unset in compose, cron string is bad, or the container hasn't been up long enough to hit the schedule.
- **Push script prints `no tarball yet` forever.** Same as above. The push timer only runs if the container-side step wrote something.
- **Retention deletes the wrong ones.** The script sorts by `createdAt`. Manually-uploaded releases with weird timestamps can throw the sort. If in doubt, back up `~/palworld-server/palworld/backups/` to a local disk before running retention the first time.
- **Repo bloat from having committed tarballs to git in an earlier attempt.** Git's history keeps them. Use `git filter-repo` or start with a fresh repo. Releases are the way.
- **Rate limits.** GitHub's REST API allows plenty for one server's backup cadence. If you're pushing hourly across many servers, watch for 403s.
