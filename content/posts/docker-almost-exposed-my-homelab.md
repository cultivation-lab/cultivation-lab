---
title: "Docker Almost Exposed My Homelab, and ufw Couldn't Have Stopped It"
date: 2026-08-18
draft: false
tags: ["ctf", "ctfd", "docker", "homelab", "self-hosted", "cybersecurity"]
---

I was chasing a boring YAML error when I noticed something on my server that had no business being reachable from the internet. It had been sitting there, live, for a few minutes. My firewall didn't catch it, and it wouldn't have — because of how Docker handles ports.

This started as a simple weekend project: stand up [CTFd](https://ctfd.io), a self-hosted platform for running "capture the flag" security challenges, on the same VPS where I've been doing other lab work (and where a Minecraft server for my friends also lives). The plan was low-stakes — keep it loopback-only, reachable only through an SSH tunnel, same pattern I'd already used for a DVWA container earlier in this project. No public exposure, no firewall changes needed. Simple.

It mostly was simple. Except for the part where I almost put a service on the open internet without meaning to, and my firewall would have just... let it happen.

## Setting it up

CTFd ships with an official Docker Compose file, so getting it running is normally just a clone and a `docker compose up`:

```
git clone https://github.com/CTFd/CTFd.git /root/ctfd
```

I wanted it fully isolated from the other Docker stuff already running on the box (a separate `vulnlab` network has my DVWA container), so I gave it its own network first:

```
docker network create ctfd_net
```

and pointed the compose file at it by editing the `ports:` line for the `ctfd` service to bind to loopback only, and adding an external network declaration:

```yaml
ports:
  - "127.0.0.1:8000:8000"
```

```yaml
networks:
  default:
    external: true
    name: ctfd_net
```

Then, trying to bring it up:

```
docker compose up -d
```

```
yaml: construct errors: line 79: mapping key "networks" already defined at line 74
```

Fair enough — I'd added a `networks:` block without realizing the file already reserves that key for something. Nothing dramatic, just a YAML mistake. But chasing it down is what led to the actual story.

## What the error made me actually look at

To fix a duplicate key, you need to see the whole file, so I dumped it with line numbers:

```
cat -n /root/ctfd/docker-compose.yml
```

Turns out the stock compose file defines a root-level `networks:` block with two networks — `default` and an `internal`-only one that the database and cache containers use so they're never exposed at all. My edit had clobbered part of that instead of merging with it, which was the actual bug. Easy fix: put both back.

```yaml
networks:
  default:
    external: true
    name: ctfd_net
  internal:
    internal: true
```

But while I had the whole file in front of me, something else caught my eye. Buried in the middle, a service I hadn't even thought about:

```yaml
  nginx:
    image: nginx:stable
    restart: always
    ports:
      - 80:80
```

That's the reverse proxy CTFd ships with by default. And that port line has no host IP in front of it — which means it doesn't bind to loopback like I'd carefully set up for the `ctfd` service. It binds to *every* interface. `0.0.0.0:80`. The whole internet.

I don't even need nginx here — I'm reaching CTFd through an SSH tunnel straight to the container, not through a reverse proxy. It got included automatically because it's just part of the stock file, and I hadn't scrutinized every service in it before running `up`.

## The part that actually worried me

Here's the thing that made this more than a simple "oops, close the port" fix: ufw was running on this box the whole time, and it would not have saved me.

Docker doesn't play nicely with ufw. When you publish a container port, Docker writes its own rules directly into iptables to route that traffic — and those rules get inserted in a way that, in the default configuration, bypasses ufw's own filtering entirely. Ufw can show a totally locked-down ruleset, deny-by-default, everything looking safe, and a Docker-published port will still be reachable from the outside world regardless. This is a known, well-documented gotcha with running Docker and ufw together, not some exotic edge case — and I'd read about it before, in the abstract, without ever hitting it myself.

So the honest version of what happened: by the time I noticed the `ports: - 80:80` line, `docker compose up` had already run once. The nginx container had genuinely been alive, on the public IP, for a few minutes, and my firewall had no say in the matter. Nothing came of it that I know of — this isn't a "here's how I got hacked" post — but the exposure itself was real, not hypothetical, and closing it wasn't optional.

First move: stop bleeding immediately, before touching anything else.

```
docker compose stop nginx
```

```
✔ Container ctfd-nginx-1 Stopped
```

Then verify, with a real command instead of just trusting the compose output:

```
ss -tlnp | grep ':80 '
```

Empty output. Port 80 was actually closed, not just logically "stopped" in Docker's bookkeeping.

## The actual fix

Stopping the container is a band-aid — it comes right back on the next `docker compose up`. The real fix was deciding I didn't need nginx for this setup at all, and removing the service block from the compose file entirely:

```yaml
  nginx:
    image: nginx:stable
    restart: always
    volumes:
      - ./conf/nginx/http.conf:/etc/nginx/nginx.conf
    ports:
      - 80:80
    depends_on:
      - ctfd
```

deleted, then cleaned up the stopped container and brought the rest of the stack back up:

```
docker compose rm -f nginx
docker compose up -d
```

```
NAME           IMAGE           SERVICE   STATUS          PORTS
ctfd-cache-1   redis:4         cache     Up 24 seconds   6379/tcp
ctfd-ctfd-1    ctfd-ctfd       ctfd      Up 5 minutes    127.0.0.1:8000->8000/tcp
ctfd-db-1      mariadb:10.11   db        Up 24 seconds   3306/tcp
```

That's the state I actually wanted from the start: `ctfd` bound to loopback only, `db` and `cache` with no published ports at all, nothing sitting on the public interface. Reachable only by tunneling in:

```
ssh -p 31337 -L 8000:127.0.0.1:8000 root@your-server-ip
```

and visiting `http://127.0.0.1:8000` locally to run through CTFd's setup wizard.

## Seeding it with something to actually play

A CTF platform with nothing on it isn't much fun, so I pulled two challenges from [csivitu/ctf-challenges](https://github.com/csivitu/ctf-challenges), an open-source archive with real challenges and their own `challenge.yml` metadata files ready to import via CTFd's `ctfcli` tool — a Håstad's-broadcast-attack RSA puzzle called "Quick Math," and a password-protected-zip forensics one called "Panda."

Importing wasn't completely frictionless — the repo's `challenge.yml` files use an older CTFd scoring format (`type: dynamic` without a required `initial` field), so both needed a quick edit to `type: standard` before they'd install. Panda also had a files path written for a different folder layout than the one I was importing from. Small stuff, easily fixed once the error messages pointed at it, but worth mentioning since "just clone and import" wasn't quite true.

The one I'm most pleased with, though, I built myself. Back when I was working through DVWA's SQL injection module for an earlier post, I extracted five leaked password hashes and cracked four of them against a small wordlist — the fifth, belonging to a user named `1337`, didn't turn up in that list and I left it as an open thread. Finishing CTFd gave me a good reason to go back and actually close it out, this time with the real `rockyou.txt` wordlist instead of a token one:

```
python3 crack_md5_big.py
```

```
1337: CRACKED -> charley
```

`charley` — one of DVWA's own seeded default passwords, just not a common enough word to show up in a small list. I turned that into a proper challenge: here's a leaked hash, go crack it, the flag is `flag{charley}`.

All three challenges are live and confirmed working now — checked in the browser through the tunnel, visible on the board, submittable.

## Where it stands

CTFd is up, isolated on its own Docker network, loopback-only, reachable only through SSH, seeded with three real challenges. The box's Minecraft server never noticed any of this happening. And I've got a very concrete reminder, instead of just a read-somewhere warning, that "ufw is running" and "this port isn't actually exposed" are two different claims when Docker is involved — worth checking with `ss -tlnp` after every `docker compose up`, not just assuming the compose file's port mapping matches what ufw would allow if it were asked.

Next up on this box: probably CrowdSec, or finally finishing off the rest of DVWA's modules. Either way, I'll be checking `ss -tlnp` first.
