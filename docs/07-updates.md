# 07. Updates

**TL;DR:** Palworld ships patches without warning. Check Steam's public buildid against what your container installed. If they differ, safe-restart triggers SteamCMD to pull the new build on boot. The image's cron does this nightly on its own. This doc is for the "friends can't join right now" moments.

## Why buildid, not version string

The version string that REST `/info` returns (like `v1.0.1.100619`) is for humans. It stays sticky within patch groups. If you rely on it to decide whether to update, you'll be one patch behind and not know.

Steam publishes a **buildid** for every dedicated-server release. That's the authoritative signal. Your container has one baked into its `appmanifest_2394010.acf` when SteamCMD downloads the game. Compare the two.

## The 30-second check

Bash, from any Linux with `jq` (works fine from the VPS itself):

```bash
STEAM=$(curl -sS "https://api.steamcmd.net/v1/info/2394010" \
  | jq -r '.data."2394010".depots.branches.public.buildid')

OURS=$(ssh <user>@<VPS_IP> \
  'sudo docker exec palworld-server grep -oE "\"buildid\"[[:space:]]+\"[0-9]+\"" \
   /palworld/steamapps/appmanifest_2394010.acf' \
  | grep -oE '[0-9]+$')

echo "steam=$STEAM  ours=$OURS"
[ "$STEAM" = "$OURS" ] && echo current || echo "UPDATE AVAILABLE"
```

PowerShell one-liner from a Windows workstation:

```powershell
$steam = (Invoke-RestMethod "https://api.steamcmd.net/v1/info/2394010").data.'2394010'.depots.branches.public.buildid
$ours  = (ssh <user>@<VPS_IP> 'sudo docker exec palworld-server grep -oE ''"buildid"[[:space:]]+"[0-9]+"'' /palworld/steamapps/appmanifest_2394010.acf') -replace '.*"([0-9]+)".*','$1'
"steam=$steam  ours=$ours  " + $(if($steam -eq $ours){"current"}else{"UPDATE AVAILABLE"})
```

## Sources of truth

| Signal | Where | Use for |
|---|---|---|
| **buildid** | Steam: `steamcmd.net/v1/info/2394010`. Ours: `appmanifest_2394010.acf` inside the container. | Reliable currency comparison. |
| `timeupdated` | Same Steam response | Human-readable "when did Steam last publish". |
| `version` string | REST `/v1/api/info` | Human display. Tell friends "we're on v1.0.1.x". Do not use for currency checks. |
| `Downloading update` log lines | `docker logs palworld-server` | Whether the last recreate actually pulled bytes. Absence means nothing was new that boot. |

## Apply

When the check says `UPDATE AVAILABLE`:

### 1. Player check

Never skip. Use REST `/players` for the authoritative list:

```bash
ADM=$(grep '^ADMIN_PASSWORD=' ~/palworld-server/.env | cut -d= -f2)
ssh <user>@<VPS_IP> "sudo docker exec palworld-server curl -sS \
  -u admin:$ADM http://127.0.0.1:8212/v1/api/players | jq '.players | length'"
```

If the count is 0, proceed. If not, either wait for the empty window or use the graceful path below.

### 2. Recreate the container

`docker compose up -d --force-recreate` triggers SteamCMD on boot, which pulls the new buildid.

```bash
ssh <user>@<VPS_IP> 'cd ~/palworld-server && sudo docker compose up -d --force-recreate'
```

Downloads take 1 to 3 minutes. The container's `health: starting` state sits longer than a normal restart. That extra time is the update.

### 3. Graceful path if players are online

If friends are on and you can't wait, use REST to announce, save, and shut down cleanly. The container's `unless-stopped` restart policy brings it back, SteamCMD pulls on the way up.

```bash
ADM=$(grep '^ADMIN_PASSWORD=' ~/palworld-server/.env | cut -d= -f2)
ssh <user>@<VPS_IP> "sudo docker exec palworld-server curl -sS -X POST \
  -u admin:$ADM -H 'Content-Type: application/json' \
  -d '{\"message\":\"Server restarting in 60 seconds for update.\"}' \
  http://127.0.0.1:8212/v1/api/announce"

sleep 55

ssh <user>@<VPS_IP> "sudo docker exec palworld-server curl -sS -X POST \
  -u admin:$ADM http://127.0.0.1:8212/v1/api/save"

ssh <user>@<VPS_IP> "sudo docker exec palworld-server curl -sS -X POST \
  -u admin:$ADM -H 'Content-Type: application/json' \
  -d '{\"waittime\":5,\"message\":\"Restarting now.\"}' \
  http://127.0.0.1:8212/v1/api/shutdown"
```

Container restarts, players see a countdown, save happens before the door closes.

### 4. Wait for healthy

```powershell
for ($i=0; $i -lt 30; $i++) {
  $s = ssh <user>@<VPS_IP> 'sudo docker inspect --format "{{.State.Health.Status}}" palworld-server'
  Write-Host ("t={0,3}s {1}" -f ($i*10), $s)
  if ($s -eq 'healthy') { break }
  Start-Sleep 10
}
```

### 5. Verify

REST `/info` returns the new version and the buildid check now agrees:

```bash
ssh <user>@<VPS_IP> 'sudo docker exec palworld-server bash /home/steam/server/rest_api.sh info'
```

## After every update: check for schema drift

Palworld can add, remove, or rename ini keys silently. Any tool that bakes in a fixed list (a config editor, a settings validator) can silently drop unknown keys and corrupt your config.

Quick key-count check:

```powershell
$out  = ssh <user>@<VPS_IP> 'sudo cat ~/palworld-server/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini'
$blob = ($out -join "`n") -replace '(?s).*OptionSettings=\((.*)\).*', '$1'
$parts = New-Object System.Collections.Generic.List[string]
$d = 0; $sb = New-Object System.Text.StringBuilder
foreach ($ch in $blob.ToCharArray()) {
  if ($ch -eq '(') { $d++; [void]$sb.Append($ch) }
  elseif ($ch -eq ')') { $d--; [void]$sb.Append($ch) }
  elseif ($ch -eq ',' -and $d -eq 0) { $parts.Add($sb.ToString()); [void]$sb.Clear() }
  else { [void]$sb.Append($ch) }
}
if ($sb.Length -gt 0) { $parts.Add($sb.ToString()) }
"key count: $($parts.Count)"
```

Compare to your last known count (118 as of Palworld v1.0.1.100619). If it changed, note what's new before applying any prior ini edits blindly.

## Rolling back a bad patch

Sometimes a Palworld patch breaks something you care about. Save files stop loading, a mod breaks, framerate tanks. SteamCMD supports pinning to a named branch other than `public`, which is Palworld's rollback surface.

Check which branches Steam publishes for App ID `2394010`:

- [SteamDB config page for Palworld Dedicated Server](https://steamdb.info/app/2394010/config/) → "Branches" section. Shows every named branch, its current buildid, and the last time it changed.
- Or via the JSON mirror: `curl -sS "https://api.steamcmd.net/v1/info/2394010" | jq '.data."2394010".depots.branches | keys'`.

If Pocketpair has published a rollback branch (they sometimes tag one as `previous` or a dated name), pin your container to it by adding these to `docker-compose.yml` under the palworld service's `environment:` block:

```yaml
STEAMCMD_APP_BRANCH: "<branch-name>"
STEAMCMD_APP_BRANCH_PASSWORD: ""     # only if the branch is password-gated
```

Then force-recreate. On boot SteamCMD runs `+app_update 2394010 -beta <branch-name> validate` and pulls that branch's buildid instead of `public`. Reverse by removing the env vars and force-recreating again.

`thijsvanloef/palworld-server-docker` and `jammsen/palworld-dedicated-server` both respect these env vars, but read your image's README before relying on this in an outage. Env var names have shifted before.

Reference: the SteamCMD wiki's [`+app_update` docs](https://developer.valvesoftware.com/wiki/SteamCMD#Downloading_an_App) cover the `-beta` flag in general.

## Automation status

The `thijsvanloef` image runs this cron inside the container:

```
0 */12 * * *  bash /usr/local/bin/backup
0 6 * * *     bash /usr/local/bin/update
30 6 * * *    bash /home/steam/server/auto_reboot.sh
```

- **06:00 UTC daily**: update check via SteamCMD. If a new build exists and both `UPDATE_ON_BOOT=true` and `REST_API_ENABLED=true` are set (image defaults), broadcasts a countdown via REST `/announce`, calls `/save`, calls `/shutdown`. `unless-stopped` policy restarts, SteamCMD pulls on boot.
- **06:30 UTC daily**: safety-net reboot. Player-guard uses REST `/players`, so it skips if anyone is online.
- **Every 12 hours**: local backup tarball (see [06-backups.md](06-backups.md)).

Timing choice: 06:00 UTC catches Pocketpair's typical evening-Tokyo release window. If a patch drops just after your check window, you might miss it by up to 24 hours. Manual runbook above closes that gap.

## What can go wrong

- **`Update state (0x5) verifying install` sits forever.** SteamCMD is verifying every file. Give it 5 to 10 minutes. If it's still hung, `docker compose down` + `docker compose up -d --force-recreate` restarts the check cleanly.
- **Post-update, players say "server version mismatch".** They need to update their client on Steam / Xbox / PlayStation. Palworld doesn't do backward compatibility between patch groups.
- **REST auth returns 401 after an update.** Usually the WorldOption.sav override again. See [09-when-things-break.md](09-when-things-break.md).
- **Ini keys you were editing disappeared.** Palworld deprecated them. Grep for the key in the fresh ini. If it's gone, check the official [configuration reference](https://docs.palworldgame.com/settings-and-operation/configuration/) for whatever replaced it.
- **Update cron never fires.** Verify the daemon is running: `sudo docker exec palworld-server ps -eo pid,cmd | grep supercronic`. If missing, the image failed to start supercronic; recreate the container.
