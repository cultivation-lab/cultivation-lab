---
title: "A cheap, low-maintenance Minecraft server for four friends — and the things that tripped me up"
date: 2026-08-08
draft: false
tags: ["minecraft", "self-hosting", "papermc", "linux"]
---

I wanted a survival server four friends could jump on whenever — cheap to run, and low-maintenance enough that I'd basically forget it was there. Vanilla-ish, no giant modpack, just a persistent world that stays up on its own. This is the full build start to finish, commands and all. And because the stuff that actually cost me an evening was never the stuff I expected, I've left every gotcha in place, right where it bit me.

## The box

I'm on a Contabo Cloud VPS 10 — 8GB RAM, Ubuntu Server. For four players on a vanilla-ish world that's plenty. For a small server the RAM headroom matters more than core count, and this tier is cheap monthly, which was the entire point.

Everything below happens over SSH on that box. I do most of it as root (more on that quirk later), so where you see `sudo`, drop it if you're already root.

## A dedicated user for the server

Don't run a game server as root. Make a user that owns the server and nothing else:

```bash
sudo adduser --disabled-password --gecos "" minecraft
```

That gives me a `minecraft` user with a home at `/home/minecraft` and no password — I get into it from a root shell when I need to, rather than logging in as it directly. The whole server will live at `/home/minecraft/mcserver`:

```bash
mkdir -p /home/minecraft/mcserver
```

## Installing Java 25 (Amazon Corretto)

First real gotcha, and it's a startup-blocker: **the Java version has to match the Minecraft version.** MC 26.x needs Java 25 or newer. If you just install whatever JDK `apt` gives you by default, you can easily land on something older and Paper simply won't boot with a cryptic class-version error.

I used Amazon Corretto 25. The clean way on Ubuntu is to add Amazon's apt repo:

```bash
wget -O - https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/corretto-keyring.gpg] https://apt.corretto.aws stable main" | sudo tee /etc/apt/sources.list.d/corretto.list
```

```bash
sudo apt-get update && sudo apt-get install -y java-25-amazon-corretto-jdk
```

Then confirm it's actually 25 before you go any further:

```bash
java -version
```

You want to see `Corretto-25.x` in that output. If it says anything lower, stop and fix it here — nothing downstream will work otherwise.

## Getting Paper onto the box

The server runs **PaperMC** rather than the vanilla jar. Paper is a drop-in replacement that performs better and takes plugins — no real downside for a survival server.

Here's a trap I hit chasing an old tutorial: **the old `api.papermc.io/v2` download URLs are dead** (they stopped getting builds at the end of 2025). Don't copy a `wget` line from some 2024 blog — it'll 404. Grab the current jar from the official downloads page instead: [papermc.io/downloads](https://papermc.io/downloads). On a headless box, find the 26.2 build, copy the download link, and pull it straight in as `paper.jar`:

```bash
cd /home/minecraft/mcserver
```

```bash
wget "PASTE_THE_26.2_DOWNLOAD_LINK_HERE" -O paper.jar
```

## First boot and the EULA

Run the jar once by hand. It'll generate the config files, then immediately stop and complain that you haven't accepted the EULA:

```bash
java -jar paper.jar --nogui
```

That first run creates `eula.txt`, `server.properties`, and the rest. Accept the EULA:

```bash
echo "eula=true" > eula.txt
```

Now's the moment to hand the whole folder to the `minecraft` user, since I ran that first boot as root and root now owns those fresh files:

```bash
chown -R minecraft: /home/minecraft/mcserver
```

## server.properties: the few things I actually changed

The first run generated `server.properties`. The handful I set for this server:

```properties
online-mode=true
max-players=4
server-port=25565
```

`online-mode=true` means only real, authenticated Minecraft accounts can join — leave that on. `max-players=4` is just us. Port 25565 is the default.

Gotcha worth knowing before you go hunting for it: **`pvp` is no longer in `server.properties`.** It moved to an in-game gamerule back in 1.21.2, so a `pvp=true` line here does nothing on 26.2. If you want to toggle it, it's `/gamerule pvp true|false` now, not this file.

One workflow note that'll save you confusion later: a running server rewrites `server.properties` on shutdown, so it'll clobber hand-edits. **Stop the server first, edit, then start it** — don't edit it live.

## Running it as a systemd service

I want the server to come back after a reboot and restart itself if it crashes, without me babysitting it. That's a systemd service. I also want a real console I can attach to, so the service launches the server inside a `screen` session.

Install screen first:

```bash
sudo apt-get install -y screen
```

Then the unit file at `/etc/systemd/system/minecraft.service`:

```ini
[Unit]
Description=Minecraft Server (Paper)
After=network.target

[Service]
User=minecraft
WorkingDirectory=/home/minecraft/mcserver
ExecStart=/usr/bin/screen -DmS minecraft /usr/bin/java -Xms6G -Xmx6G -jar paper.jar --nogui
ExecStop=/usr/bin/screen -p 0 -S minecraft -X eval 'stuff "stop"\015'
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

A few things going on there. It runs as the `minecraft` user out of the server directory. `screen -DmS minecraft` starts a named screen session (`minecraft`) in the foreground so systemd can supervise it — but I can still attach to it. The `-Xms6G -Xmx6G` gives the JVM 6 GB; on an 8GB box that leaves room for the OS and BlueMap's rendering later — tune to your machine. And `ExecStop` sends the literal `stop` command into the console so the world saves cleanly instead of getting hard-killed.

Enable it (so it survives reboot) and start it:

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable --now minecraft
```

From here on, start/stop/restart is just:

```bash
sudo systemctl restart minecraft
```

## The console lives inside screen

To actually see the live console and type server commands, attach to the screen session as the `minecraft` user:

```bash
su - minecraft -c "screen -r"
```

Detach with **Ctrl+A** then **D**.

Gotcha: **never hit Ctrl+C in that console** — it doesn't detach, it kills the server. Ctrl+A then D every time.

One more thing that confused me early: **server console commands take no leading slash.** In that screen console you type `chunky ...` or `whitelist ...` bare. In-game as an op, the *same* commands take a leading `/`.

## The `sudo -u minecraft` quirk that ate an hour

Here's one that surprised me: `sudo -u minecraft ...` doesn't work in my setup. So whenever I need to drop a file where the server can read it, my workflow is: download or edit it as root, then hand ownership over:

```bash
chown minecraft: server-icon.png
```

Small thing, but I spent real time on "why can't the server see this file" before it clicked that it was an ownership problem, not a path problem. If a file you added is invisible to the server, check who owns it first.

## Opening the firewall

The server won't be reachable until you let the port through:

```bash
sudo ufw allow 25565/tcp
```

(BlueMap adds one more port later — I'll open 8100 when I get to it.)

At this point the server is live and a friend can connect. The rest is making it nice.

## Giving it a face: MOTD and icon

Two cosmetic touches that punch way above their weight in the multiplayer list.

The **MOTD** is the couple of lines shown under the server name. Colors use section-sign codes — and this one got me good: you **can't** put a raw `§` in `server.properties`, it breaks the line. You have to write the color codes as `\u00A7` escapes, and join the two lines with `\n`. So it's a single line in the file that looks like this:

```properties
motd=\u00A79My Survival Server\n\u00A76Under construction — come build
```

`\u00A79` is blue, `\u00A76` is gold.

The **icon** is a small PNG the client shows next to your server. Gotcha: it must be **exactly 64×64**, and the filename is case-sensitive — `server-icon.png`, in the server root. Drop it in, then (per the ownership thing above):

```bash
chown minecraft: server-icon.png
```

Two more traps here. The server only reads the icon **at startup**, so restart to pick up a change. And the **client caches it** — if you swap the icon and don't see the new one, it's not broken, you just need to refresh (or remove and re-add) the server entry in your multiplayer list.

## One tuning change: view-distance

I bumped `view-distance` from the default 10 up to 12 — a bit more world visible, still comfortable for four players on this box. I left `simulation-distance` at 10, because that's the one that actually costs CPU (it's what's being ticked, not just rendered).

## Plugins

This is where a small server earns its keep. Everything's from Hangar (Paper's plugin site) unless I say otherwise.

### BlueMap — a live 3D web map

BlueMap renders the world into a browsable 3D map you open in a browser. It runs its own little webserver on port 8100, which I opened in the firewall:

```bash
sudo ufw allow 8100/tcp
```

You view it at `http://<server-ip>:8100`.

Worth knowing: mine is plain http and publicly reachable — anyone with the address can look at it. It's read-only, but it does reveal builds and base locations. For a trusted friend group that's a fine tradeoff; just go in eyes-open that the map is as public as its address is.

### Chunky — pre-generate the world

New chunks are the laggy part of Minecraft: the game generates terrain the first time someone walks into it, and that's where the stutter comes from. Chunky generates it ahead of time, while nobody's around. I pre-generated a 3000-block radius around spawn (typed in the server console, no slash):

```
chunky radius 3000
```

```
chunky start
```

It's CPU-heavy while it runs, so do it with friends offline. Progress survives restarts — `chunky pause` and `chunky continue` to manage it, `chunky progress` for an ETA. Nice bonus: BlueMap picks up the newly generated chunks automatically, so pre-genning also fills in the map.

### AxGraves — death chests

Vanilla death scatters your stuff on the ground with a timer before it's gone forever, which for a casual group is just misery. AxGraves catches everything — items and XP — in a floating grave right where you died. I tuned it for a family server: graves never expire, only the owner can open theirs, and it keeps your XP on pickup. Nice detail: it's packet-based, so nothing extra actually gets written into the world file.

### Prism — block logging and rollback (grief insurance)

This is the one plugin I wouldn't run a shared world without. It logs who placed and broke what, so if something gets griefed — or you fat-finger a build yourself — you can roll it back.

Two gotchas, back to back:

First, my actual first choice was CoreProtect, and there just wasn't a working free build for 26.2 — current-MC support was paywalled or broken. So I switched to **Prism**, which does the same job.

Second, Prism needs the **NBTAPI** library plugin or it won't even enable — and the copy of NBTAPI on Hangar is stale, stuck on an old MC version. I had to grab NBTAPI from **Modrinth** instead to get a build that works on 26.2.

For future me — always preview before you apply. Rollback params are `r:<n>` for radius, `since:<time>` like `since:2h`, and `p:<player>` to scope it to one person. The flow is preview, then apply or cancel:

```
/prism preview-rollback r:10 since:2h p:<player>
```

```
/prism preview-apply
```

```
/prism preview-cancel
```

`/prism near` and `/prism lookup` inspect history without changing anything. Docs are at docs.prism-mc.org.

## Backups

A nightly cron tars up the world folder at 3am. Nothing fancy — a root crontab entry (`crontab -e`) like:

```
0 3 * * * tar -czf /home/minecraft/backups/world-$(date +\%F).tar.gz -C /home/minecraft/mcserver world
```

Gotcha in that line: inside a crontab, `%` is special and has to be escaped as `\%`, or cron silently mangles the command. Make the `backups` directory first, obviously. It means a bad day is a restore instead of a rebuild. Prism runs its own record purge at midnight, deliberately clear of the backup so they don't collide.

## The meta-lesson: pin your versions

If there's one thing I took away from this build, it's that Paper and its plugins move fast and don't always agree with each other across versions. Two of my three real headaches were pure compatibility problems — CoreProtect had no 26.2 build, and NBTAPI's main listing was stale. So my rule now: before I update Paper or any plugin, I check that each plugin actually lists my exact MC version on its own page, and I take a manual world backup before bumping anything. Everything here is pinned on purpose.

That's the whole thing. Four friends, a cheap box, and a world that stays up on its own.
