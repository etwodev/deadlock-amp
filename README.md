# Deadlock AMP Template

An [AMP](https://cubecoders.com/AMP) Generic Module template for running a Deadlock dedicated server on Linux.

Valve does not ship a Linux server build, so this template downloads the Windows game files via SteamCMD and runs `deadlock.exe` under [Proton GE](https://github.com/GloriousEggroll/proton-ge-custom). The server is wrapped in `xvfb-run`, because in dedicated mode the game still opens a console window, and it hangs if no display is available.

> **Status:** early work in progress. The server console does not accept input under Proton, so AMP manages the server over RCON instead.

## Requirements

| | Recommended |
|---|---|
| OS | Linux x86_64 (e.g. Debian 13) |
| CPU | 4+ cores, high clock speed preferred |
| RAM | 8 GB+ (16 GB comfortable) |
| Disk | 60 GB+ free. There is no separate server app, so the full game client is installed |
| Steam | An account that has Deadlock in its library. Anonymous download is not possible |
| AMP | 2.8.0.4 or newer, with Docker available for instances (recommended) |

## 1. Prepare the host

Source 2 under Proton needs more memory mappings than Linux allows by default. Run this on the **host**, since a container cannot change it:

```bash
echo 'vm.max_map_count=1048576' | sudo tee /etc/sysctl.d/99-deadlock.conf
sudo sysctl --system
```

Open the game port (default `27067`) for both protocols. UDP carries game traffic and TCP carries RCON:

```bash
sudo ufw allow 27067/udp
sudo ufw allow 27067/tcp
```

If you enable SourceTV, also open its port (default `27020`, UDP):

```bash
sudo ufw allow 27020/udp
```

If your provider has its own firewall or security group, open the ports there as well.

## 2. Add the template to AMP

1. In the main AMP panel (ADS), go to **Configuration → Instance Deployment → Configuration Repositories**.
2. Click **Add** and enter:
   ```
   etwodev/deadlock-amp:main
   ```
3. Click **Fetch Latest**.

## 3. Create the instance

1. Click **Create Instance** and choose **Deadlock** from the application list.
2. Enable the **Docker / container** option. The template uses the `cubecoders/ampbase:debian` image, which includes the tools Proton needs.
   If you run without Docker, install `python3`, `jq` and `wget` on the host.
3. Create the instance and open it.

## 4. First update

1. Click **Update**.
2. AMP will ask for your Steam username and password, then a Steam Guard code if the account has it enabled.
3. The update runs four stages:
   - **SteamCMD Download:** the Windows Deadlock files (about 38 GB; this takes a while).
   - **Steamworks SDK Redist Download:** Valve's Windows `steamclient64.dll`, which the server needs to register with Steam.
   - **Proton GE Download:** the latest Proton GE release, unless a specific version is set under **Updates → Runtime Configuration**.
   - **Copy Steamworks Client Into Proton Prefix:** places `steamclient64.dll` and its dependencies where the game looks for them (`C:\Program Files (x86)\Steam\`). Without a Steam client installed, Proton cannot provide this file itself.

## 5. Configure the server

Settings are under **Configuration → Deadlock**, split into tabs:

| Tab | What it covers |
|---|---|
| Server | Name, password, map, Enable Cheats, LAN mode, pausing, hibernation, voice and text chat, extra console variables |
| Match | Street Brawl and 1v1 modes, duplicate heroes, lanes and lane swapping, pre-game, intro and post-game timings, item drafting, pause timers |
| Gameplay | Damage multipliers, ability cooldown cap, infinite ammo, shop anywhere, forced enhanced or corrupted items, movement, regen zones, gravity, game speed |
| Troopers & Spawns | Troopers, neutrals, NPCs, mid boss, powerups, breakables, healing pickups, and the crate and capture objectives, each with spawn timing overrides |
| Economy | How trooper souls are awarded, orb drops, bounty sharing, objective rewards, the corrupted item shop |
| Sandbox | Hero testing mode, infinite souls, target dummy stats, Buddha mode |
| Bots | Practice bots, bot takeover for disconnected players, bot difficulty |
| Network | Public IP, bandwidth and update rates, lag compensation |
| SourceTV | Spectator relay, delay, password, demo recording |
| Logging | Server log files |

Almost every setting defaults to **Game default** (or is left blank), and is only added to the command line once you change it.

Settings marked **Requires Enable Cheats** need **Enable Cheats** turned on; the game ignores them otherwise. Enabling cheats lets **every player** use cheat commands.

Valve strips `developmentonly` console variables from the public game, so some settings you might expect cannot be changed on a dedicated server. These include starting souls, respawn times and the trooper cap per lane.

The **crate** and **capture (koth)** objectives are listed under their internal names, because Valve does not document which in-game event each one is.

### No-creeps practice mode

Tick **Enable Cheats**, then set **Spawn Troopers** and **Spawn Neutrals** to **Disabled** in the **Troopers & Spawns** tab.

There is no known setting that stops a match from ending. With troopers off, lanes are not pushed, so the match only ends if players destroy the objectives. If that happens, you can restore them over RCON:

```
citadel_respawn_boss_t1
citadel_respawn_boss_t2
citadel_respawn_boss_barracks
citadel_respawn_boss_shrines
```

## 6. Start and connect

1. Click **Start**. AMP marks the server as ready once its RCON connection succeeds.
2. Players open the in-game console and run:
   ```
   password <server password>
   connect <server ip>:27067
   ```
   Skip the `password` line if no password is set.

Commands typed in the AMP console are sent over RCON. To confirm the spawn settings applied, run:

```
citadel_trooper_spawn_enabled
```

## Troubleshooting

The server's console output does not appear in AMP under Proton. It is written to `deadlock/1422450/game/citadel/console.log` inside the instance folder instead.

| Symptom | Things to check |
|---|---|
| Stuck on "Starting" | AMP waits for RCON to connect. Check `console.log` (above) and that a `deadlock.exe` process is running. A process using only ~50 MB of RAM at 0% CPU means it is hung waiting for a display; confirm the instance launches via `xvfb-run` |
| `Failed to load module '...steamclient64.dll'` in `console.log` | Run **Update** again so the Steamworks redist is downloaded and copied into the Proton prefix |
| Many `Failed loading resource` lines in `console.log` | Harmless: sound and model files the server does not need |
| Update fails at SteamCMD | The Steam account must own Deadlock. Re-enter credentials and the Steam Guard code |
| Server crashes shortly after start | Confirm `vm.max_map_count` is set on the host (`sysctl vm.max_map_count`) |
| Server never becomes ready | Check the TCP port is open and nothing else is using it. Look at the instance log for Proton errors |
| Troopers still spawn | Enable Cheats must be ticked. Check the value over RCON as shown above |
| Proton issues after an update | Pin a known-good version, e.g. `GE-Proton9-5`, under **Updates → Runtime Configuration** |

## Files

| File | Purpose |
|---|---|
| `deadlock.kvp` | Main template: executable, command line, RCON, Steam login |
| `deadlockconfig.json` | Settings shown in the AMP UI |
| `deadlockports.json` | Port definitions |
| `deadlockupdates.json` | Update stages (SteamCMD, Proton GE) |
| `deadlockmetaconfig.json` | Config file mappings (none currently) |
| `manifest.json` | Repository manifest AMP reads when fetching templates |
