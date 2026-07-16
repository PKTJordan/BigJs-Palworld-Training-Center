# BigJ's Unsolicited Palworld Dedicated Server Training Center

> *Welcome, Trainer.* Grab your starter and a terminal.

You want a server. Just yours. Yours and your friends'. No shared hardware, no month-to-month pricing tricks, no admin panel that hides the real config behind three menus. A patch of hillside you fence off, put a Palbox on, and invite people to.

That's what this repo teaches. On a cheap VPS, running the same Docker image the pros run, driven by Claude Code so you don't have to memorize which SteamCMD flag does what.

## What you get

A working Palworld server for your friend group. Sized to your peak player count, not someone else's default. You own the box, you own the config, and nothing is hidden behind a managed-panel abstraction. Beyond the server itself, this guide brings a few things most other setups skip.

### Free offsite backups on GitHub Releases

Not "the server writes tarballs to a folder and calls it a day." A full pipeline. The container writes a save tarball every 12 hours. A systemd timer pushes each new tarball up to a private GitHub Release, tagged with the save filename. A nightly job prunes old releases down to the 14 most recent, which works out to 7 days rolling. One command restores from any point in that window. Zero dollars, off-box, versioned, portable. Most guides skip this or point you at S3 with a credit card.

### Every trap we hit, already flagged

WorldOption.sav silently overriding your ini, the one that costs everyone two hours. The PlM save format killing old migration tools. `PalWorldSettings.ini` being regenerated from compose env vars on every container start, which means direct edits vanish on restart. `IS_MULTIPLAY` defaulting to false and hiding your server from the community browser. PS5 clients that can't type an IP. `BaseCampWorkerMaxNum` being ignored by the game even when set correctly. `S_API FAIL` in the logs looking like a crash but being harmless startup noise. Each one has a documented fix in [when-things-break](docs/09-when-things-break.md).

### The right update runbook

Palworld ships patches without warning. The version string REST returns is sticky and lies to you. This guide teaches the [buildid comparison](docs/07-updates.md) that actually works. It also covers the graceful REST-based announce/save/shutdown flow so you can update without kicking anyone, and the SteamCMD `-beta` rollback path for when a patch breaks something you care about.

### REST-first, RCON-out

Pocketpair announced RCON is deprecated and will stop working. Most guides you'll find still use it. This one moved to the REST API and shows you the endpoints, the auth pattern, and how to script against them.

### Two skills for the price of one hobby

You'll pick up working Claude Code fluency and enough Linux + Docker to be useful long after Palworld. The commands here transfer directly to hosting Minecraft, Factorio, Valheim, or anything else that ships as a Linux binary. The Claude Code workflows transfer to any coding or ops task you throw at it.

## What it costs

The VPS bill depends on how many friends you want on at once (see sizing below). Add Claude Code at about $20/month on Anthropic's Pro plan; it reads Pocketpair's docs so you don't have to memorize which SteamCMD flag does what. You don't need a domain name. Friends connect by IP.

Managed alternatives run $20 to $40 a month per server, for a slice of shared hardware sized to whoever isn't playing right now. DIY gets you a whole dedicated box, plus skills that carry to the next game you want to run.

## How much VPS

Start with the official requirements page: [docs.palworldgame.com/getting-started/requirements](https://docs.palworldgame.com/getting-started/requirements).

Rule: **16 GB is the floor. Add 1 GB per player. Cap is 32 GB at 32 players (game max).** Anything below 16 GB is asking for trouble.

| Peak players | RAM to buy | vCPU | Disk |
|---|---|---|---|
| 1–16 | **16 GB (floor)** | 4+ | 100 GB |
| 20 | 20 GB | 4+ | 100 GB |
| 24 | 24 GB | 6+ | 150 GB |
| 32 (game max) | 32 GB | 6+ | 200 GB |

**Push higher if your play style is server-intense.** Raid events, sprawling multi-guild bases, heavy Palbox activity, big guild churn all spike RAM and CPU. If your friends run frequent raids, size a tier up.

Single-threaded work dominates CPU load, so clock speed matters more than core count. 4 modern vCPUs handle up to about 16 players.

Disk is mostly the server binary (about 5 GB), your save file (grows slowly), and backup tarballs. 100 GB covers a small group. Bigger worlds want more headroom.

Pick your VPS from the tiers below to hit the RAM your peak needs.

## Who this is for

- You like knowing how the sausage is made.
- You've used a terminal before, or you're willing to. If `ssh` isn't in your vocabulary yet, that's fine. Claude Code will type it for you and explain what it did.
- You want your Pals fed by *your* servers.

## Who this is not for

- You want one button to click and be playing in ten minutes. Go managed. Real list below, no judgment.
- You need to host 30+ strangers. A small dedicated box won't cut it. A bigger VPS or a proper managed host will treat you better.
- You never want to touch the terminal. Managed it is.

## What this is not

Not affiliated with Pocketpair, Anthropic, Contabo, or any hosting provider mentioned. Not a mod, not a cheat, not a redistribution of any Palworld file. It's a guide. All outbound links point at the actual maintainers.

## The stack you'll end up with

| Piece | Pick | Why |
|---|---|---|
| VPS | Any Ubuntu box sized for your peak player count (see above). Contabo, Vultr, DigitalOcean, Hetzner all work. | You want root and enough RAM. That's it. |
| OS | Ubuntu 24.04 LTS | Docker installs cleanly. Long support tail. Boring on purpose. |
| Container | `docker` + `docker compose` | Reproducible. One file describes your whole server. |
| Palworld image | [thijsvanloef/palworld-server-docker](https://github.com/thijsvanloef/palworld-server-docker) (community) or [pocketpairjp/palworld-dedicated-server-docker](https://github.com/pocketpairjp/palworld-dedicated-server-docker) (official) | The community image adds auto-pause, scheduled backups, REST wrapper scripts, Discord webhooks. The official one is the pure Pocketpair build. This guide leans on the community image for the QoL features. |
| Backups | GitHub Releases | Free. 2 GB per file. Retention by simple deletion. You already have GitHub. |
| Copilot | [Claude Code](https://www.anthropic.com/claude-code) | Reads Pocketpair's docs so you don't have to memorize them. |

Ubuntu, Docker, and the images aren't provider-specific. Any Linux VPS runs this.

## Guides

Numbers reflect the intended order once the whole set lands. Ones with links are done.

1. Choose a VPS. Sizing, region, provider notes.
2. Provision Ubuntu. SSH keys, hardening, Docker, ufw.
3. Deploy Palworld. The compose file, first boot, port opens.
4. Invite friends. Server browser, direct IP, PS5/Xbox crossplay.
5. Bring an old save with you. The PlM/Kraken save-format saga and the WorldOption.sav trap.
6. [Backups.](docs/06-backups.md) GitHub Releases as your free vault, retention.
7. [Updates.](docs/07-updates.md) Check what Steam is shipping, apply without kicking anyone.
8. Tuning. The ini editor for gameplay settings.
9. [When something breaks.](docs/09-when-things-break.md) The checklist.
10. What it actually costs. Real bills, real bandwidth.

## Gotchas we hit so you don't have to

These are the ones that cost real hours to figure out. Every full doc will link back here.

### WorldOption.sav overrides your ini

Palworld writes a snapshot of server settings into `Pal/Saved/SaveGames/0/<worldhash>/WorldOption.sav` when a world is generated. On every subsequent load, that file overrides matching values in `PalWorldSettings.ini`. Confirmed for `AdminPassword`, `ServerName`, `ServerPlayerMaxNum`, and probably more.

It bites hardest when you migrate a single-player save. The SP copy of WorldOption.sav has empty AdminPassword, "Default Palworld Server" as the name, and max 32 players. Every restart those defaults win over whatever you set in the ini. You edit the ini, restart, nothing changes. REST auth returns `401 "Unauthorized (AdminPassword is empty)"` no matter what password you send.

The fix is one command: back up and delete `WorldOption.sav`, then restart the server. It regenerates a fresh copy from the current ini on next boot. `Level.sav` (bases, Pals, characters) is untouched.

Any ini value that refuses to apply, suspect WorldOption.sav before the image, before the docs, before yourself.

### Bringing a single-player save to a dedicated server

Two things move separately: the world and your character.

For the world, copy `Level.sav`, `LevelMeta.sav`, `WorldOption.sav`, `GlobalPalStorage.sav`, and the `Players/` folder from your local save at `%LOCALAPPDATA%\Pal\Saved\SaveGames\<SteamID>\<worldhash>\` on Windows into the dedicated server's world folder. Then wipe WorldOption.sav per the section above. Server boots, your bases and Pals are there.

Your character is where the pain lives. Single-player saves store the host character under a placeholder GUID, or under a SteamID-derived one that doesn't match what the dedicated server assigns you. Join the dedicated server and you'll get a brand-new level-1 character in your old world, while your real character sits orphaned in `Level.sav`.

The only tool that reliably works on Palworld 1.0+ (PlM/Kraken save format) is [deafdudecomputers/PalWorldSaveTools](https://github.com/deafdudecomputers/PalWorldSaveTools), a Windows GUI. Its "Fix Host Save" menu rewrites GUID references in `Level.sav` and renames the player.sav. Older CLI tools like [xNul/palworld-host-save-fix](https://github.com/xNul/palworld-host-save-fix) are pinned to the older PlZ format and fail with `not a compressed Palworld save, found b'PlM' instead of b'PlZ'`.

Ballpark workflow: join dedicated once and level to 2 (this creates your target player.sav), stop the container, pull `Level.sav` + `Players/` to your Windows PC, run the tool, push back, restart, rejoin.

### PlayStation players can't type in an IP

PS5 and PS4 Palworld clients don't have a "connect by IP" box. Xbox and PC have it, PlayStation doesn't. Your PlayStation friends have to find the server in the in-game Community Server browser.

For your server to appear there, all of these have to be true:

- `bIsMultiplay=True` in the ini (image env `IS_MULTIPLAY=true`, which defaults to false in some images and must be set explicitly)
- Server launched with the `-publiclobby` flag (image env `COMMUNITY=true`)
- `PublicIP="<your VPS IP>"` in the ini (image env `PUBLIC_IP=...`)
- `PublicPort=8211`
- `bUseAuth=True`
- `CrossplayPlatforms=(Steam,Xbox,PS5,Mac)`, with PS5 in the list

Miss any of these and the server never registers with Palworld's master list. It stays reachable by direct IP for Steam and Xbox players, but PlayStation friends won't see it at all.

To check it worked, search [Battlemetrics](https://www.battlemetrics.com/servers/palworld) for your VPS IP or server name. If your server shows up there, PlayStation clients can see it. If it doesn't, you're missing one of the above. Battlemetrics polls infrequently for low-rank servers, so give it 15 to 60 minutes after a config change. The [Battlemetrics API](https://api.battlemetrics.com/) with a browser User-Agent header lets you script the check.

## Resource map

Everything below is a real, checked link. **Official** means Pocketpair themselves. **Community** means maintained by someone volunteering. **Provider blog** means a paid host's help article that happens to be right. **⚠** means broken or dying, listed so you don't waste an afternoon on it.

### Palworld, official (Pocketpair)

| Resource | What it is |
|---|---|
| [docs.palworldgame.com](https://docs.palworldgame.com/) | The canonical source. Terse. Bookmark. |
| [Deploy dedicated server (Linux, SteamCMD)](https://docs.palworldgame.com/getting-started/deploy-dedicated-server/) | The Pocketpair walkthrough. Assumes basic Linux. |
| [Requirements](https://docs.palworldgame.com/getting-started/requirements) | 32 GB RAM is the ceiling, not the floor. |
| [PalWorldSettings.ini reference](https://docs.palworldgame.com/settings-and-operation/configuration/) | Every one of the 118 server settings, with an official description for most. |
| [Command-line arguments](https://docs.palworldgame.com/settings-and-operation/arguments) | `-publiclobby`, `-publicip`, `-publicport`, `-players`, threading flags. |
| [REST API](https://docs.palworldgame.com/category/rest-api) | 13 endpoints (info, players, metrics, settings, save, announce, kick, ban, unban, shutdown, stop, game-data). Basic auth, `admin:<AdminPassword>`. This is how you manage a live server. |
| [RCON](https://docs.palworldgame.com/api/rcon/) | ⚠ Deprecated by Pocketpair. Scheduled to stop working. Migrate to REST. |
| [Steam store: Palworld Dedicated Server](https://store.steampowered.com/app/2394010/Palworld_Dedicated_Server/) | App ID `2394010`. Free via SteamCMD. |
| [pocketpairjp/palworld-dedicated-server-docker](https://github.com/pocketpairjp/palworld-dedicated-server-docker) | Pocketpair's own Docker image and compose files. Barebones. No auto-pause, no scheduled backups, no REST helpers. Use this if you want to build your own operational layer on top of the pure official binary. |

### Community: Docker + save tools

| Resource | What it is |
|---|---|
| [thijsvanloef/palworld-server-docker](https://github.com/thijsvanloef/palworld-server-docker) | The image this whole guide is built on. Handles SteamCMD auto-update, auto-pause (drops idle RAM to ~500 MB), scheduled backups, REST wrapper scripts, Discord webhooks. Actively maintained. Pocketpair also publishes an [official image](https://github.com/pocketpairjp/palworld-dedicated-server-docker), which is barebones by comparison. |
| [jammsen/docker-palworld-dedicated-server](https://github.com/jammsen/docker-palworld-dedicated-server) | Third major community image. Similar feature set to thijsvanloef: SteamCMD auto-update, in-container `restapicli` and `backup` commands, Discord webhook events, Helm chart. Removed RCON entirely from container tooling in favor of REST. Good alternative if you like its FAQ style or want a second opinion on env-var naming. |
| [deafdudecomputers/PalWorldSaveTools](https://github.com/deafdudecomputers/PalWorldSaveTools) | Windows GUI save editor. Ships its own Kraken decoder for the modern PlM save format. The reliable path for migrating a single-player character to a dedicated server. |
| [cheahjs/palworld-save-tools](https://github.com/cheahjs/palworld-save-tools) | Python library. Reads PlM. Character schema lags the current game, so it's fine as a dependency but not runnable for host-fix as-is. |
| [xNul/palworld-host-save-fix](https://github.com/xNul/palworld-host-save-fix) | ⚠ Dead on Palworld 1.0+. Pinned to a save-format library that only handles old PlZ (zlib) files. Errors with `not a compressed Palworld save, found b'PlM' instead of b'PlZ'`. Was the community standard before the format changed. |
| [NFZ-441/Palworld-Co-op-to-Dedicated-Server-Migration-Tool](https://github.com/NFZ-441/Palworld-Co-op-to-Dedicated-Server-Migration-Tool) | Fork of xNul with `pyooz` for PlM decompression. Decoder works, schema doesn't. Worth watching. |

### VPS providers

| Provider | Notes |
|---|---|
| [Contabo](https://contabo.com/en/vps/) | Cheapest per-GB-RAM. US and EU regions. Free network firewall in the panel. Cloud VPS tiers scale by RAM: 8, 12, 16, 32 GB, and up. Confirmed to run this stack. |
| [Vultr](https://www.vultr.com/) | Cleaner control panel. Best if you want a specific US-East data center. Roughly 3× Contabo pricing at similar RAM. |
| [DigitalOcean](https://www.digitalocean.com/) | Friendliest first-VPS experience. Roughly 2× Contabo pricing. |
| [Hetzner](https://www.hetzner.com/cloud) | Unbeatable EU pricing. US regions often waitlisted. |

Anything else that gives you Ubuntu root works too. Match RAM to the sizing table above.

### Steam infrastructure

| Resource | What it is |
|---|---|
| [SteamCMD](https://developer.valvesoftware.com/wiki/SteamCMD) | Valve's tool for downloading and updating dedicated servers. The Docker image runs this for you. Also documents `+app_update`, the `-beta` flag for rollback branches, `validate`, and the `appmanifest_*.acf` files your update check reads. |
| [SteamDB: Palworld Dedicated Server](https://steamdb.info/app/2394010/) | Every published buildid, branch, depot, and recent change to App ID 2394010. The [config page](https://steamdb.info/app/2394010/config/) is where you check for rollback branches if a patch breaks something. |
| [steamcmd.net public app info](https://api.steamcmd.net/v1/info/2394010) | Third-party JSON mirror of Steam's app-info feed. Perfect for a one-line "did Palworld ship a new build?" check without running SteamCMD locally. |

### Server discovery + monitoring

| Resource | What it is |
|---|---|
| [Battlemetrics: Palworld](https://www.battlemetrics.com/servers/palworld) | Every Palworld server that publishes to the master list. Cloudflare fronts the web UI, so use the API with a browser User-Agent. |
| [Battlemetrics API](https://api.battlemetrics.com/) | Rate-limited but free. Great for "is my server actually visible to strangers?" checks. |

### Provider blogs worth reading

Not affiliated with these providers. Their help articles happen to be accurate on some tricky parts.

| Resource | What it is |
|---|---|
| [Game Host Bros: Host Character Transfer](https://www.gamehostbros.com/guides/games/palworld/host-character-transfer) | The "level to 2 before exiting, force save via console" tip for character migration. Some steps are their-platform-specific. Extract the transferable idea. |
| [DatHost: Edit Palworld Server Settings](https://dathost.com/blog/how-to-edit-palworld-server-settings) | Documents the WorldOption.sav-overrides-ini trap. Solves a bug that will otherwise cost you half a day. |
| [Sparked Host: Change Palworld Server Settings](https://help.sparkedhost.com/en/article/how-to-change-palworld-server-settings-1fvvpgi/) | Same story, different words. |
| [Supercraft Host: Edit PalworldSettings.ini](https://supercraft.host/wiki/palworld/server_settings/) | Same again. Three independent providers agreeing means you can trust the fix. |

### Tools that make setup less painful

| Resource | What it is |
|---|---|
| [Claude Code](https://www.anthropic.com/claude-code) | Reads the docs, drives the terminal, remembers what you already told it. The whole guide leans on this. |
| [TigerVNC](https://tigervnc.org/) | Console client for when SSH breaks and you need to log in through your VPS provider's KVM. |
| [ClickPaste](https://github.com/Collective-Software/ClickPaste) | Types your clipboard as keypresses. The only sane way to paste into some noVNC consoles, Contabo's included. |

### Going managed instead

If any of this reads like homework, skip it. Any of these will host Palworld for you and handle updates.

- [Game Host Bros](https://www.gamehostbros.com/)
- [DatHost](https://dathost.com/)
- [Sparked Host](https://help.sparkedhost.com/)
- [Supercraft Host](https://supercraft.host/)
- [Bisect Hosting](https://www.bisecthosting.com/)
- [Nitrado](https://server.nitrado.net/)
- [Shockbyte](https://shockbyte.com/)
- [GTXGaming](https://www.gtxgaming.co.uk/)
- [Rocket Node](https://rocketnode.com/)

**Read the fine print before you buy.** Most managed Palworld tiers sell RAM that sits below the 16 GB floor above (some as low as 4 or 8 GB), because they know most groups won't hit peak use. Their plans rarely specify the exact CPU model, vCPU count, or whether your resources are shared with other tenants (they usually are). What looks like a $15/mo bargain often turns into a laggy weekend session, a "please upgrade to the next tier" prompt, and a $30/mo bill for what you could have run yourself on a dedicated 16 GB box.

You get what you pay for. Compare their per-month cost at a tier that actually meets the 16 GB floor, not their loss-leader tier. Then compare that to your VPS bill plus Claude Code. Pick the one you'll enjoy more.

## License

MIT. See [LICENSE](LICENSE).

## Thanks

To the maintainers of every project linked above. Especially thijsvanloef for the Docker image and deafdudecomputers for the only save tool that actually works on the modern format. Standing on their shoulders.
