---
title: "A cheap, low-maintenance Minecraft server for four friends — and the things that tripped me up"
date: 2026-08-08
draft: false
tags: ["minecraft", "self-hosting", "papermc", "linux"]
---

I wanted a survival server four friends could jump on whenever — cheap to run, and low-maintenance enough that I'd basically forget it was there. Vanilla-ish, no giant modpack, just a persistent world that stays up on its own. This is the whole build start to finish. And because the stuff that actually cost me an evening was never the stuff I expected, I've left every gotcha in place, right where it bit me.

## The box

I'm on a Contabo Cloud VPS 10 — 8GB RAM, Ubuntu Server. For four players on a vanilla-ish world that's plenty. For a small server the RAM headroom matters more than core count, and this tier is cheap monthly, which was the entire point.

## PaperMC and the Java version that has to match

The server runs Minecraft 26.2 on **PaperMC** rather than the vanilla jar. Paper is a drop-in replacement that performs better and lets me run plugins later — no downside for a survival server.

First gotcha, and it's a startup-blocker: **the Java version has to match the Minecraft version.** MC 26.x needs Java 25 or newer. I used Amazon Corretto 25. If you just install whatever JDK your package manager hands you by default, you can easily land on something older and Paper simply won't boot. Sort the Java version out first.

The server lives at `/home/minecraft/mcserver` and runs as a dedicated `minecraft` user. Don't run a game server as root.

## Keeping it alive: systemd + screen

I run it as a systemd service, so it comes back after a reboot and auto-restarts if it crashes. Start/stop/restart is just:

```bash
sudo systemctl restart minecraft
```

The live console runs inside a `screen` session owned by the `minecraft` user. To attach and actually see the console:

```bash
su - minecraft -c "screen -r"
```

Detach with **Ctrl+A** then **D**.

Gotcha: **never hit Ctrl+C in that console** — it doesn't detach, it kills the server. And for anything a running server might overwrite when it shuts down (`server.properties` is the big one), stop the service first, make your edit, then start it again. Otherwise your change gets clobbered on the next restart.

## The `sudo -u minecraft` quirk that ate an hour

Here's one that surprised me: `sudo -u minecraft ...` doesn't work in my setup. So my workflow for putting any file where the server can read it is: download or edit it as root, then hand ownership over to the `minecraft` user:

```bash
chown minecraft: server-icon.png
```

Small thing, but I spent real time on "why can't the server see this file" before it clicked that it was an ownership problem, not a path problem.

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

BlueMap renders the world into a browsable 3D map you open in a browser. It runs its own little webserver on port 8100, which I opened in the firewall. You view it at `http://<server-ip>:8100`.

Worth knowing: mine is plain http and publicly reachable — anyone with the address can look at it. It's read-only, but it does reveal builds and base locations. For a trusted friend group that's a fine tradeoff; just go in eyes-open that the map is as public as its address is.

### Chunky — pre-generate the world

New chunks are the laggy part of Minecraft: the game generates terrain the first time someone walks into it, and that's where the stutter comes from. Chunky generates it ahead of time, while nobody's around. I pre-generated a 3000-block radius around spawn:

```
chunky radius 3000
```

```
chunky start
```

It's CPU-heavy while it runs, so do it with friends offline. Progress survives restarts — `chunky pause` and `chunky continue` to manage it, `chunky progress` for an ETA. Nice bonus: BlueMap picks up the newly generated chunks automatically, so pre-genning also fills in the map.

One thing that confused me early on: **server console commands take no leading slash.** The commands above are typed straight into the console as `chunky ...`. In-game as an op, the *same* commands take a leading `/`.

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

A nightly cron at 3am tars up the world folder. Nothing clever, but it means a bad day is a restore instead of a rebuild. Prism runs its own record purge at midnight, deliberately clear of the backup so they don't collide.

## The meta-lesson: pin your versions

If there's one thing I took away from this build, it's that Paper and its plugins move fast and don't always agree with each other across versions. Two of my three real headaches were pure compatibility problems — CoreProtect had no 26.2 build, and NBTAPI's main listing was stale. So my rule now: before I update Paper or any plugin, I check that each plugin actually lists my exact MC version on its own page, and I take a manual world backup before bumping anything. Everything here is pinned on purpose.

That's the whole thing. Four friends, a cheap box, and a world that stays up on its own.
