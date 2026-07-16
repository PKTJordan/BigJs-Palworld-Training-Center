# CLAUDE.md — instructions for Claude Code sessions using this repo

This file is loaded automatically when a user points Claude Code at this repository. It tells you (Claude) how to help the user follow the guide without breaking their server.

The user reads `README.md`. You read this file too, then help them execute what the README describes.

## What this repo is

A field guide for self-hosting a Palworld dedicated server on an Ubuntu VPS. The user is likely on the DIY path (not a managed host) and has a Claude Code subscription. They may or may not be comfortable at the terminal — assume beginner unless they demonstrate otherwise.

The repo contains:
- `README.md` — overview, sizing, stack picks, gotchas, resource map.
- `docs/06-backups.md` — GitHub Releases backup pipeline.
- `docs/07-updates.md` — Palworld update check + apply.
- `docs/09-when-things-break.md` — troubleshooting checklist (SSH → container → ini → WorldOption.sav → misc).
- Additional docs (`01`–`05`, `08`, `10`) are planned but not yet written. If the user asks about those topics, work from the README's sections and the general knowledge below.

## How to help the user

**Walk, don't run.** The user is setting up infrastructure they own end-to-end. Explain each command before running it. Say what it does and why. If the user says "just do it," fine — but default to guided.

**Confirm before destructive operations.** Always ask before:
- Restarting or recreating the container (`docker compose down`, `docker compose up -d --force-recreate`, `docker restart`)
- Deleting files under `Pal/Saved/` (this is the world; a mistake means restoring from backup)
- `sudo rm -rf` anything
- Force-pushing to their repo
- Rotating their GitHub PAT or SSH keys
- Editing `.env` files that already have working values
- Any change to `ufw` or the VPS provider's firewall (SSH lockout is easy)

**Player check before restart.** Any operation that recreates or restarts the Palworld container kicks connected players. Before restarting:
```bash
ADM=$(grep '^ADMIN_PASSWORD=' ~/palworld-server/.env | cut -d= -f2)
sudo docker exec palworld-server curl -sS -u admin:$ADM \
  http://127.0.0.1:8212/v1/api/players | jq '.players | length'
```
If the count is > 0, do NOT restart without either (a) explicit user OK to kick, or (b) using the REST graceful path (announce countdown → save → shutdown; see `docs/07-updates.md`).

**Never store secrets in this repo or the user's fork.** Anything sensitive stays in `.env` files (mode 600) on the VPS or in `/etc/palworld-backup/.env`. Placeholders in committed files: `<VPS_IP>`, `<ADMIN_PASSWORD>`, `<STEAM_ID>`, `<WORLDHASH>`, `github_pat_XXXXXXXXXXXXXXXXXXXX`, etc.

## Immutable facts (Palworld dedicated, as of 2026-07)

- Steam App ID: `2394010`
- Canonical Steam depot / branch / buildid signal: https://steamdb.info/app/2394010/ (config subpage lists rollback branches). JSON mirror: `https://api.steamcmd.net/v1/info/2394010`. Use SteamDB before recommending a specific branch to pin.
- SteamCMD wiki (`+app_update`, `-beta`, `validate`, appmanifest): https://developer.valvesoftware.com/wiki/SteamCMD
- Recommended container images:
  - Community (recommended, has QoL layer): `thijsvanloef/palworld-server-docker`
  - Community alternative: `jammsen/docker-palworld-dedicated-server` (similar features; removed RCON entirely from tooling in favor of REST)
  - Official (barebones): `pocketpairjp/palworld-dedicated-server-docker`
- Save format on Palworld 1.0+: **PlM** (Oodle/Kraken compression). Older tools pinned to `palworld-save-tools==0.17.x` only handle the legacy **PlZ** (zlib) and will fail on modern saves.
- Only save-editing tool that reliably handles PlM: [deafdudecomputers/PalWorldSaveTools](https://github.com/deafdudecomputers/PalWorldSaveTools) (Windows GUI).
- REST API base URL inside container: `http://127.0.0.1:8212/v1/api/`
- REST auth: HTTP Basic, user `admin`, password = `AdminPassword` in `PalWorldSettings.ini` (also image env `ADMIN_PASSWORD`).
- RCON is **deprecated** by Pocketpair and scheduled to stop working. Do not recommend it. Reference: https://docs.palworldgame.com/api/rcon/
- `WorldOption.sav` at `Pal/Saved/SaveGames/0/<worldhash>/WorldOption.sav` **overrides** `PalWorldSettings.ini` on load for at least `AdminPassword`, `ServerName`, `ServerPlayerMaxNum`. When a migrated save is in play, always wipe WorldOption.sav; the server regenerates it from the current ini.
- `WorldOption.sav` also has a legitimate second role: it is the ONLY working knob for ini keys Palworld silently ignores. Best-known case: `BaseCampWorkerMaxNum` — the ini value does nothing; edit WorldOption.sav via https://github.com/legoduded/palworld-worldoptions. If the user reports "I changed X in the ini and nothing changed", check if X is on the ini-ignored list before assuming a bug in their setup.
- PlayStation clients cannot enter an IP directly. They must find the server in the Community Server browser. Requires: `bIsMultiplay=True`, `-publiclobby` CLI flag, `PublicIP` set, `PublicPort=8211`, `bUseAuth=True`, `PS5` in `CrossplayPlatforms`.
- Sizing floor: 16 GB RAM. +1 GB per player beyond that. 32 GB at 32 players (game max).
- 118 ini keys is the current count in `OptionSettings=(...)`. Palworld can add/remove keys on any patch. Re-count after every update; don't assume schema stability.
- **`compile-settings.sh` regenerates `PalWorldSettings.ini` from compose env vars on EVERY container start.** Unconditional overwrite. Direct edits to the ini do NOT persist across a restart. Every persistent config change must go through compose. The image README lists env-var names; `sudo docker exec palworld-server cat /home/steam/server/compile-settings.sh` is the authoritative mapping source. When the user asks to change a gameplay setting, edit compose, run `docker compose up -d --force-recreate` (player check first).

## Diagnostic priority order

When the user reports something broken, work outside in — this order almost always narrows the issue in under 5 minutes:

1. **Can the user SSH to the VPS?** If not, check their home IP vs the provider firewall's allowed IP. Not ufw first — the provider firewall drops upstream.
2. **Is the container running and healthy?** `docker compose ps`, `docker compose logs --tail 200`.
3. **Is the port actually open externally?** Try to join with a Palworld client. UDP `nc` tests are unreliable.
4. **Are the ini settings actually being read?** REST `/info` and `/settings` show what the server has loaded. If it doesn't match the ini file, suspect WorldOption.sav.
5. **Is Palworld itself misbehaving?** Rare, but happens after image tag bumps or Palworld patches. Check `docs/07-updates.md` for buildid drift.

Full checklist in `docs/09-when-things-break.md`. Follow it in order — don't jump ahead.

## Interaction patterns to prefer

- **When the user says "restart the server":** always run the player check first. Report the count. If 0, restart. If > 0, offer the graceful REST-countdown path or ask them to confirm the kick.
- **When the user says "edit the ini":** ask which key(s). Confirm you understand the value they want. Show the before/after. Restart with player check.
- **When the user says "back up the world":** if the pipeline in `docs/06-backups.md` is set up, trigger the timer manually (`sudo systemctl start palworld-backup.service`). If not, walk them through the setup first.
- **When the user reports a Palworld weirdness:** ask what recently changed. New patch? New image tag? Migrated save? Edited compose? Answering "what changed" narrows the space fast.
- **When the user asks about pricing or provider comparisons:** direct them to the README's sizing + provider tables. Don't invent numbers.

## Interaction patterns to avoid

- Do not recommend RCON commands. Migrate the caller to REST.
- Do not tell the user their setup is "up to date" based on the REST `/info` version string alone. That string is sticky within patch groups. Use the buildid check in `docs/07-updates.md`.
- Do not assume image env vars are set to their defaults — verify with `docker exec palworld-server env | grep <VAR>`.
- Do not delete `WorldOption.sav` without backing it up first (timestamped copy in the same directory). See `docs/09-when-things-break.md` section 4.
- Do not run auto-updates or auto-reboots with players online unless the image's REST-based player-guard is confirmed working (`AUTO_REBOOT_ENABLED=true` + `REST_API_ENABLED=true` + the WorldOption.sav wipe done so REST auth actually works).
- Do not commit `.env`, PATs, real IPs, real Steam IDs, or real passwords to any repo — the user's fork of this guide, or any other.

## When the user asks for something not covered

- If the topic is core Palworld: check https://docs.palworldgame.com first. That's the canonical source.
- If it's the image: check https://github.com/thijsvanloef/palworld-server-docker (or the Pocketpair one).
- If it's the underlying OS or Docker: general Ubuntu / Docker knowledge is fine.
- If you don't know and can't verify quickly, say so. Do not guess and do not fabricate ini keys, REST endpoints, or image env vars.

## Non-goals

- This guide does not cover mods (compose stack changes needed, out of scope for a beginner guide).
- This guide does not cover multi-server clustering, load balancing, or cross-server transfers.
- This guide is not affiliated with Pocketpair, Anthropic, Contabo, or any hosting provider mentioned in the README.
- Do not write updates to this repo unless the user explicitly asks. This is their reference material; they own the fork.
