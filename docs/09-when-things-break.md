# 09. When something breaks

**TL;DR:** work outside in. Network first (can you SSH?), then container (is it running?), then game config (is the ini right?), then Palworld quirks (WorldOption.sav override). Most Palworld weirdness is one of five things below.

## Order to check

1. Can you SSH to the box?
2. Is the container running and healthy?
3. Is the port actually open from the outside?
4. Are your ini settings actually being read?
5. Is Palworld itself misbehaving (upstream bug or format drift)?

Every symptom below maps to one of these layers.

## 1. SSH won't connect

Symptoms: `ssh <user>@<VPS_IP>` hangs, times out, or "connection refused" after previously working.

Order:

1. **Your home IP changed.** Most likely cause. Check: `curl -s https://ifconfig.me`. If different from what your VPS provider's network firewall (Contabo, Vultr, DO all offer these) allows, the panel is now blocking you. Update the SSH rule in the provider panel with your new IP `/32`. Provider firewalls apply within seconds.
2. **VPN active.** Same problem, different cause. Turn it off and retry.
3. **Provider network firewall attached but no rule for you.** Log in via the provider's KVM/console (Contabo noVNC, DO console, Vultr view), fix the firewall from the panel.
4. **`ufw` on the box changed.** Console in, `sudo ufw status`, confirm `22/tcp ALLOW`.
5. **sshd itself is broken.** Console in, `sudo systemctl status ssh`, `sudo journalctl -u ssh -e --since '1 hour ago'`.

If the provider firewall is attached and blocks you, the VPS console is the only way in. Not ufw. Not a reboot. Console.

## 2. Container won't start

```bash
docker compose logs --tail 200
```

Common causes:

- **Bad `.env` values.** Missing `SERVER_PASSWORD` or `ADMIN_PASSWORD`. Image refuses to start.
- **Wrong ownership on `./palworld/`.** SteamCMD runs as UID 1000 inside the container. `sudo chown -R 1000:1000 ~/palworld-server/palworld/`.
- **SteamCMD download interrupted.** `docker compose down && docker compose up -d`. Image resumes the download.
- **Disk full.** `df -h`. Palworld binaries plus backups can pile up.
- **Config syntax error.** Compose won't parse. `docker compose config` (dry-run) shows the exact line.

## 3. Port shows closed from outside

Two firewalls in play, both must ACCEPT:

- **Provider network firewall** (Contabo, DO, Vultr, etc). UDP 8211 = ACCEPT from Any. UDP 27015 = ACCEPT from Any (if you want Battlemetrics to scrape live). TCP 22 = ACCEPT from your IP `/32`.
- **`ufw` on the VPS.** `sudo ufw status`. Same rules, same ports.

To open a new port: add to BOTH. The provider firewall drops upstream of ufw.

Test from home (UDP has no ACK, so a positive result doesn't prove much, but a negative rules the port out):

```bash
nc -u -z -v <VPS_IP> 8211
```

The real test is trying to join from a Palworld client.

## 4. Ini changes don't apply (the WorldOption.sav trap)

Symptoms:

- You edit `ServerName`, `AdminPassword`, or `ServerPlayerMaxNum` in the ini, restart the container, and the running server acts on the OLD value.
- REST `/info` shows the wrong servername.
- REST auth returns `401 "Unauthorized (AdminPassword is empty)"` no matter what password you send.

Cause: `Pal/Saved/SaveGames/0/<worldhash>/WorldOption.sav`. Palworld writes a snapshot of world settings into this file when the world is generated. On every subsequent load, that snapshot **overrides** the ini for `AdminPassword`, `ServerName`, `ServerPlayerMaxNum`, and likely more.

This bites hardest after migrating a single-player save to dedicated. The SP save's WorldOption.sav has empty AdminPassword, default server name, max 32 players. Those defaults win every restart until you get rid of the file.

Fix, one-shot wipe, PalServer regenerates from current ini:

```bash
WORLD=~/palworld-server/palworld/Pal/Saved/SaveGames/0/<worldhash>
sudo cp -a "$WORLD/WorldOption.sav" "$WORLD/WorldOption.sav.bak-$(date -u +%Y%m%dT%H%M%SZ)"
sudo rm "$WORLD/WorldOption.sav"
cd ~/palworld-server && sudo docker compose up -d --force-recreate
```

Verify:

```bash
ADM=$(grep '^ADMIN_PASSWORD=' ~/palworld-server/.env | cut -d= -f2)
sudo docker exec palworld-server curl -sS -u admin:$ADM \
  http://127.0.0.1:8212/v1/api/info
```

Servername should match your ini. Auth should return 200 instead of 401.

Bases, Pals, characters are in `Level.sav`. WorldOption.sav is settings-snapshot only. Deleting it does not touch gameplay state.

Rule going forward: any ini value that refuses to apply, suspect WorldOption.sav before the image, before the docs, before yourself.

### The other face of WorldOption.sav: a legitimate workaround

WorldOption.sav has a second job. Some Palworld ini keys are silently ignored by the running server regardless of what you set. The best-documented one is `BaseCampWorkerMaxNum`. This is an upstream bug confirmed by the [jammsen image FAQ](https://github.com/jammsen/docker-palworld-dedicated-server#i-changed-the-basecampworkermaxnum-setting-why-didnt-this-update-the-server). Editing the ini or setting the compose env var does nothing. The only path that works is editing WorldOption.sav directly.

Tool for that: [legoduded/palworld-worldoptions](https://github.com/legoduded/palworld-worldoptions). Read + write WorldOption.sav's binary format.

So the same file is both a foot-gun (when a stale copy hides your ini changes) and the required tool (when the ini path is broken). Which face you're looking at depends on the setting. When in doubt, wipe it first and let it regenerate. Only reach for the editor when a specific setting is known to be ini-ignored.

## 5. Container reports `unhealthy` but game works

Symptom: `docker compose ps` shows `Up N minutes (unhealthy)`, but joining works fine.

Cause: someone (probably a copy-pasted compose template) added a custom `healthcheck:` block pointing at a script that isn't shipped in the image. The `thijsvanloef` image already has its own healthcheck (`pgrep PalServer-Linux`), which is reliable.

Fix: remove any custom `healthcheck:` block from your compose file. `docker compose up -d --force-recreate`.

## 6. Config changes via compose aren't taking effect

Symptom: you edit `docker-compose.yml`, run `docker compose up -d`, but the running server acts on the old value.

Cause: `docker compose up -d` is a no-op if it thinks nothing changed. And on the flip side, direct edits to `PalWorldSettings.ini` inside `palworld/Pal/Saved/Config/LinuxServer/` DON'T persist across restarts. The image's `compile-settings.sh` regenerates the ini from compose env vars on EVERY container start (unconditional overwrite). So the only reliable path is: edit compose env → force-recreate the container.

Fix: change env in compose, then explicitly force-recreate.

```bash
sudo nano ~/palworld-server/docker-compose.yml   # add/change the env you want
cd ~/palworld-server && sudo docker compose up -d --force-recreate
```

Confirm the container has the new env:

```bash
sudo docker exec palworld-server env | grep <YOUR_ENV_VAR>
```

Confirm the ini reflects it:

```bash
sudo docker exec palworld-server grep <INI_KEY> \
  /palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

**Warning about direct ini edits:** they work for one server-process lifetime, then the next `--force-recreate` (which happens nightly at 06:30 UTC via `AUTO_REBOOT`) wipes them. Only use direct edits for temporary A/B tests. Anything you want permanent goes in compose.

For the ini-key → env-var mapping (there's one for nearly every ini key), inspect `/home/steam/server/compile-settings.sh` inside the container. The image README also lists common ones.

## 7. Fresh character in migrated world

Symptom: you moved a single-player save onto your dedicated server, joined with your Steam account, got a brand-new level-1 character. Your real character sits orphaned in `Level.sav`.

Cause: the SP save stored your host character under a placeholder GUID or a SteamID-derived GUID that doesn't match what the dedicated server assigns you. Level.sav's references point at the wrong player entry.

Fix: use [deafdudecomputers/PalWorldSaveTools](https://github.com/deafdudecomputers/PalWorldSaveTools) on Windows. It's the only tool that reliably handles the modern PlM (Oodle/Kraken) save format. The `Fix Host Save` menu rewrites GUID references in `Level.sav` and renames the player.sav.

Workflow:

1. Join dedicated once and level to 2. This creates your fresh target `player.sav`.
2. Stop the container. Pull `Level.sav` + `Players/` down to your Windows PC.
3. Run the tool, use `Fix Host Save`, follow the migrator dialog. Look at levels and names to know which slot is the real character. Don't trust the folder-name mapping alone.
4. Push the modified files back. Restart. Rejoin.

## 8. Save tool errors: `PlM instead of PlZ` or `Warning: EOF not reached`

You're on Palworld 1.0+ using an older tool. The save format changed to PlM (Oodle/Kraken compression). Tools pinned to `palworld-save-tools==0.17.x` only handle the older PlZ (zlib).

Only working path is `deafdudecomputers/PalWorldSaveTools`. See section 7.

## 9. RCON returns `authentication failed`

RCON is deprecated by Pocketpair. Scheduled to stop working entirely. Do not spend time debugging it. Migrate the caller to REST.

REST auth uses HTTP Basic, user `admin`, password from `AdminPassword` in the ini. If REST also returns 401, jump to section 4 (WorldOption.sav).

## 10. RAM keeps climbing past what you provisioned

Expected on long-lived worlds with big bases and lots of Pals. Mitigations already available if you're on the `thijsvanloef` image:

- `AUTO_PAUSE_ENABLED=true`: drops RAM to about 500 MB when 0 players.
- `AUTO_REBOOT_ENABLED=true`: nightly reboot at 06:30 UTC clears leaks.
- Swap file on the host: `sudo fallocate -l 4G /swapfile; sudo chmod 600 /swapfile; sudo mkswap /swapfile; sudo swapon /swapfile`. Add to `/etc/fstab` to persist.

If it still hits OOM, resize up a tier at your VPS provider. Most support live upgrades that only need a reboot.

## 11. Server won't show in community browser (PlayStation friends can't find it)

See the "PlayStation players can't type in an IP" section in the main [README](../README.md#playstation-players-cant-type-in-an-ip). All six of these must be true in the ini:

- `bIsMultiplay=True`
- `-publiclobby` CLI flag
- `PublicIP="<your VPS IP>"`
- `PublicPort=8211`
- `bUseAuth=True`
- `CrossplayPlatforms=(Steam,Xbox,PS5,Mac)`

Verify from outside by searching [Battlemetrics](https://www.battlemetrics.com/servers/palworld) for your IP. If Battlemetrics doesn't have you, PlayStation clients won't either.

## 12. Log noise that looks alarming but isn't

Common startup lines that trip up first-time server operators. All benign.

- `[S_API FAIL] Tried to access Steam interface SteamUser021 before SteamAPI_Init succeeded.` This is printed by SteamCMD's dedicated-server bootstrap. The server proceeds and works normally. Documented as safe in the [jammsen image FAQ](https://github.com/jammsen/docker-palworld-dedicated-server#im-seeing-s_api-errors-in-my-logs-when-i-start-the-container). If the server also fails to accept players, treat that as a separate problem. The S_API line is not the cause.
- `Setting breakpad minidump AppID = 2394010`. SteamCMD wiring up its crash reporter. Not a crash. Normal.
- `Update state (0x5) verifying install ...` sitting for a few minutes on boot. SteamCMD is checking every file against Steam's manifest. Give it 5 to 10 minutes. Only intervene if it's still hung after that.

## 13. Backup timer failing

```bash
sudo systemctl status palworld-backup.service
sudo journalctl -u palworld-backup.service -e --since '24 hours ago'
```

Common causes:

- PAT expired or missing scopes. Regenerate, update `/etc/palworld-backup/.env`.
- `gh` CLI not installed. `sudo apt install gh`.
- `BACKUP_DIR` in env doesn't match where the container writes.
- Retention script deleting the wrong tags (rare, but check ordering).

## When you're really stuck

Two escape hatches:

- **Container recreate is cheap.** `docker compose down && docker compose up -d`. As long as `~/palworld-server/palworld/` is intact (bind mount), your world is safe. SteamCMD re-verifies the binary on boot.
- **Restore from GitHub Releases.** See [06-backups.md](06-backups.md). Any tagged release brings your world back to that point.

Neither of these fixes are graceful. Player check first.
