---
title: "Moving My SSH Port and Standing Up a Honeypot — What Broke Along the Way"
date: 2026-08-13
draft: false
tags: ["homelab", "cybersecurity", "ssh", "honeypot", "cowrie"]
---

A bot connected to my server, tried `root` / `password`, got let in on the first try, and started typing into a shell that wasn't real. That was, genuinely, the best possible outcome — the whole point of what I'd spent the evening building was to let that happen.

I run a small Contabo VPS that mostly just hosts a Minecraft server for some friends. When Minecraft's stopped, though, it's still a real box sitting on a real public IP, and I've wanted a low-stakes place to practice cybersecurity work that's separate from my day job, purely for fun. So I decided to turn the idle time into something more interesting: stand up an SSH honeypot and watch what actually hits it. Here's how that went, including the parts that didn't work the first time.

## Taking Minecraft down first

This box only really has one job most of the time, so before touching anything else I stopped the Minecraft service and disabled it so it wouldn't come back on its own:

```
systemctl stop minecraft
systemctl disable minecraft
```

Nothing clever here, just making sure I wasn't going to step on it mid-experiment.

## Why bother moving SSH at all

The plan was Cowrie, an SSH/telnet honeypot that presents a fake shell to anyone who logs in and quietly records everything they do. But Cowrie is only interesting if it's actually catching traffic, and the traffic lives on port 22 — that's the port every mass-scanning bot on the internet already checks by default. Running a honeypot on some obscure port would mostly catch nothing. So step one wasn't installing Cowrie at all, it was freeing up port 22 by moving my real login somewhere else.

## Moving real SSH to a new port

I picked 31337 for the real login. Modern Ubuntu doesn't run sshd as a normal always-on service anymore — it uses `ssh.socket` for systemd socket activation, which changes how a port change actually gets applied. Worth checking before touching anything:

```
systemctl status ssh.socket
```

Then it was: open the new port in ufw, edit `sshd_config`, apply, verify.

**Gotcha #1:** the first time I opened `sshd_config`, I went looking for the port setting and found `#Port 22` — and assumed the `#` meant it was disabled. It's the opposite. OpenSSH ships that file almost entirely commented out just to document the defaults; nothing there is actually "off," it's just not overridden yet. The fix was adding real, uncommented lines rather than touching that block at all:

```
Port 22
Port 31337
```

(Both at once, on purpose — so the old port kept working while I confirmed the new one did too.)

Applying the change on socket-activated systems needs a specific sequence, not just a service restart:

```
systemctl daemon-reload && systemctl restart ssh.socket
```

**Gotcha #2, the sneaky one:** I ran that, and a fresh connection to port 31337 got refused. Everything *looked* right — `ufw status verbose` showed the port allowed, `grep -n "^Port" /etc/ssh/sshd_config` showed both lines saved correctly, `sshd -t` passed with no errors. The actual proof came from checking what was really bound:

```
ss -tlnp | grep ssh
```

Only port 22 showed up. The reload genuinely hadn't taken effect the first time. Running the exact same `daemon-reload && restart ssh.socket` a second time fixed it — no idea why the first pass didn't stick, but the lesson was: don't trust the config file, trust `ss -tlnp`.

Once a *brand-new* terminal session (not the one I was already logged into) could connect on 31337, I pulled the `Port 22` line back out of the config, reapplied, confirmed only 31337 was listening, and closed the old port in ufw:

```
ufw delete allow 22/tcp
```

Port 22 was now completely dark — nothing listening, nothing allowed through the firewall. Exactly the empty space Cowrie needed.

## Standing up Cowrie

Cowrie shouldn't run as root — if it's ever compromised in some novel way, I didn't want that process holding real privileges. But binding port 22 normally requires root. The fix is `authbind`, which lets one specific non-root user bind one specific privileged port and nothing else:

```
adduser --disabled-password cowrie
apt install -y git python3-venv python3-dev python3-minimal build-essential libssl-dev libffi-dev authbind

touch /etc/authbind/byport/22
chown cowrie:cowrie /etc/authbind/byport/22
chmod 770 /etc/authbind/byport/22
```

Then, as the `cowrie` user: clone the repo, set up a venv, install dependencies.

```
git clone https://github.com/cowrie/cowrie
cd cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

**Gotcha #3:** every install guide I'd read said to copy the default config next —

```
cp etc/cowrie.cfg.dist etc/cowrie.cfg
```

— which failed immediately: `No such file or directory`. Turned out the `etc/` folder was just empty. Not a bad clone (`git status` was clean, `git log` showed real history, plenty of disk space) — the project itself had moved on. This version of Cowrie ships as a real installable Python package (`pyproject.toml`, a `src/` layout) instead of a folder of loose scripts, and `pip install -r requirements.txt` only pulls in *dependencies*, not Cowrie itself. The actual fix:

```
pip install -e .
cowrie --help
cowrie init
```

That last command generated `etc/cowrie.cfg` and the log/state directories properly. From there I edited the config to listen on port 22 instead of the default 2222, and started it through authbind, since it's binding a privileged port:

```
authbind --deep cowrie start
cowrie status
```

Then a local test, over loopback, before exposing anything:

```
ssh -p 22 root@localhost
```

I typed a password, hit enter, and landed at `root@svr04:~#` — and had a genuine half-second "wait, did I just break into my own server?" moment. I hadn't. `svr04` is Cowrie's fake hostname; that prompt is the honeypot's simulated shell, not my real system. Which is exactly the point — if I couldn't immediately tell the difference, a scanning bot definitely wouldn't either.

**Gotcha #4, the dumbest one:** to actually let the outside world reach it, I needed `ufw allow 22/tcp` again — and ran it while still inside the `su - cowrie` session from setup:

```
ERROR: You need to be root to run this script
```

Right — `exit` back to the real root session first, then it went through fine.

## What showed up

I didn't have to wait long. Within a few hours, a real connection landed in the log from a real IP:

```
2026-08-13T06:53:41.870865Z [ssh,23bc9d33ae03,176.53.159.198] SSH client hassh fingerprint: 2a86d5946159539cb26c1657f9c447f2
2026-08-13T06:53:42.252694Z [ssh,23bc9d33ae03,176.53.159.198] login attempt [root/password] succeeded
2026-08-13T06:53:42.252935Z [HoneyPotSSHTransport,1,176.53.159.198] Initialized emulated server as architecture: linux-x64-lsb
2026-08-13T06:53:42.253550Z [HoneyPotSSHTransport,1,176.53.159.198] couldn't handle 50 b'...password...111111'
2026-08-13T06:53:42.253751Z [HoneyPotSSHTransport,1,176.53.159.198] couldn't handle 50 b'...password...12345678'
2026-08-13T06:53:42.380486Z [HoneyPotSSHTransport,1,176.53.159.198] avatar root logging out
2026-08-13T06:53:42.380816Z [HoneyPotSSHTransport,1,176.53.159.198] connection lost
2026-08-13T06:53:42.381122Z [ssh,23bc9d33ae03,176.53.159.198] Connection lost after 637 milliseconds
```

Translated: something fingerprinted my box, tried `root`/`password`, and Cowrie let it in — it's permissive by design, since the entire point is to see what happens *after* login. The odd part is those `couldn't handle` lines: raw auth packets containing more guessed passwords (`111111`, `12345678`, and a couple more), arriving *after* the login had already succeeded. That's not a bug on my end — it's a cheap automated client blasting an entire password list into the connection without waiting to see if it worked. The whole exchange lasted 637 milliseconds and never touched an actual command. Classic mass-scanner behavior: confirm the box is guessable, log it, move on.

## Second thoughts, and taking it back down

Watching that log land was genuinely satisfying — proof the whole chain worked, end to end. But sitting with it afterward, something about it didn't sit right: I'd deliberately made my box a more interesting, more logged, more "come back later" target for whoever's compiling lists of guessable servers. That's a different feeling than just quietly practicing in a lab, and I decided I didn't want that tradeoff running indefinitely.

So I took it back down. Closed the door first, then cleaned up behind it:

```
ufw delete allow 22/tcp
cowrie stop
rm /etc/authbind/byport/22
rm -rf /home/cowrie/cowrie
```

End state: real SSH stays on 31337 — that part's a genuine improvement regardless of the honeypot experiment — and port 22 is fully closed again, nothing listening, nothing logging.

## Worth it anyway

The honeypot itself had a short life, but I got real, reproducible practice out of it: the socket-activation gotcha on modern Ubuntu, Cowrie's newer packaging model, authbind as a cleaner alternative to iptables NAT redirects, and one real log entry proving the whole setup actually worked as designed. It also answered a question I hadn't fully thought through going in — how I'd feel about actively inviting that kind of attention onto my own box. Turns out: not as much as I expected. Good to know before trying this again with something more permanent.
