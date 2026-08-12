---
title: "Setting Up a WiFi Pineapple: A Recovery Story"
date: 2026-08-12
draft: false
tags: ["wifi-pineapple", "cybersecurity", "homelab", "wifi-security", "hak5", "wifi-pineapple-mark-7"]
---

I finally got around to setting up my WiFi Pineapple Mark VII — a piece of hardware I'd used before but hadn't touched in a while, which meant exactly one thing: there was no way I remembered the login password. What I expected to be a twenty-minute "plug it in and reset the password" job turned into most of an afternoon spent in bootloader recovery mode. Here's how it actually went, gotchas included.

## Getting connected

The standard setup is simple enough: screw in the antennas before powering it on (transmitting without them can fry the radios), then connect the Mark VII to my Ubuntu laptop over USB-C. It shows up as a USB Ethernet adapter, and the management UI lives at `http://172.16.42.1:1471`.

Except that's not what I got.

## Flash error

Instead of a login screen:

> A flash error was detected on your device. It appears that your device has been flashed improperly, and will not function correctly. Please restore your device using the recovery process... Remember you MUST flash the RECOVERY/STAGER FIRMWARE first before flashing the full firmware.

Not the start I was hoping for. Turns out this is a fairly known state for a previously-configured Mark VII to land in — not something I'd broken by doing anything wrong, just an inconsistent firmware partition.

## The recovery process

Getting into actual bootloader recovery mode is a specific dance:

1. Unplug the Pineapple completely.
2. Hold the reset button, then plug the power back in while still holding it. The LED flashes red — let go after about the third flash.
3. The LED should settle to solid red, which means you're in the bootloader.
4. Set a static IP on the USB interface (mine kept dropping DHCP in this mode, so I did it manually via the CLI):

```
sudo ip link set <interface> down
sudo ip addr add 172.16.42.42/24 dev <interface>
sudo ip link set <interface> up
```

5. Browse to `http://172.16.42.1` — critically, **no port this time**. That's different from the normal `:1471` UI, and it trips people up because it looks like the same address.
6. Upload the recovery/stager `.bin` file (downloaded from Hak5's portal, checksum verified first) and let it flash. Don't touch the power.

## Attempt one: stuck and reverted

First time through, the upload succeeded, the page said "update in progress," and I waited. And waited. Ten minutes in, the LED had briefly gone blue and then settled back to solid red — which sounds like progress, but a fresh incognito tab back at `:1471` showed the exact same flash error. The recovery hadn't actually stuck.

## Attempt two: more colors, same result

Second attempt, I saw a much wilder LED sequence — purple, green, blue, blinking blue — before it landed back on solid red again. More activity, same outcome.

At this point I tried reaching out to Hak5 support directly and didn't have any luck getting a response.

## The actual fix: don't trust "latest"

What got me unstuck was a Hak5 forum thread I found with the identical symptom — recovery flash appears to succeed, then the flash error comes right back. Their fix: after flashing the recovery/stager file, don't let the setup wizard auto-download the latest firmware. Manually side-load an older, known-stable firmware build instead.

I tried it — same recovery process, but this time selecting an older firmware file rather than trusting the auto-update — and watched the LED run through every color again. This time it landed on blue and *stayed* there instead of reverting to red. That was the tell that something was different. Sure enough, a fresh tab got me an actual login screen instead of the flash error.

If your own Mark VII is stuck in this same loop, it's worth searching the forums for your specific symptom before assuming it's a hardware fault. Mine wasn't.

## Almost there: the ICS detour

Logged in, and the setup wizard asked how I wanted the Pineapple to reach the internet: WiFi, through the connected computer (ICS), or a separate USB Ethernet adapter. I tried ICS first, since it doesn't tie up one of the Pineapple's own radios. That meant going into Ubuntu's Network settings and setting the USB interface to "Shared to other computers." Clicking "connect through the computer" in the wizard even redirected me to a Hak5 docs page for a totally different connection method, which didn't clarify anything.

It didn't work, and in trying to fix it I made things worse — I also flipped a second, unrelated connection profile ("Wired connection 2") to Shared, which broke my ability to reach the Pineapple at all. Lesson learned: GNOME's simplified Shared-mode toggle defaults to a different subnet (10.42.0.x) than the Pineapple expects (172.16.42.x), and duplicate connection profiles for the same USB link are an easy trap.

The fix was to just undo it: set both connections back to Automatic (DHCP), unplug and replug the Pineapple to force a clean interface, and confirm I could reach `172.16.42.1:1471` again. Once that baseline was back, I skipped ICS entirely and picked plain WiFi client mode instead — one screen, SSID and password, done.

Small twist: normally, connecting to WiFi at this stage triggers an automatic firmware download. Mine didn't download anything, which makes sense in hindsight — the "older file" I'd manually flashed during recovery was already the full firmware, not the stripped-down stager. There was nothing left to download. The WiFi step was just normal internet connectivity setup on an already-complete install.

## Finally, the real deal

Full sidebar — Recon, PineAP, Modules, Settings — the actual interface. First thing I did was change the password. Second time's the charm, hopefully.

## The actual project: recon on my own network

With the Pineapple working, the point was to run it against my own home network — a low-stakes first real use before doing anything more involved.

I opened Recon, ran a scan, and found my own SSID in the results along with the clients connected to it. I cross-referenced those against my router's own client list to confirm which entries were actually mine before doing anything beyond passively looking.

Clicking into my own SSID brought up a slide-out panel: add to PineAP pool, add to filter, capture WPA handshakes, clone the AP. I went with **Capture WPA Handshakes** — the safe, targeted option. It just listens for the handshake between a device and my own AP rather than broadcasting anything.

## The false lead

My first guess for why nothing was showing up was my router's client isolation setting — I have it configured so devices on my network can't see each other. Turns out that's not the cause: isolation blocks device-to-device traffic *after* a device is already connected, but has no effect on the handshake between a device and the AP itself, which happens before isolation rules matter at all. Good to rule out, but not my answer.

## The real reason: fast roaming

I toggled WiFi off and on, and airplane mode, on multiple devices — probably fifteen times each — and never captured a single handshake. My actual suspect: my router almost certainly has fast reconnect / cached-key roaming enabled (802.11r or opportunistic key caching), which lets a device reassociate using a cached key instead of doing a fresh 4-way handshake. A quick WiFi toggle just isn't enough to force a genuine handshake when that's turned on.

I could confirm this by having a device fully "forget" the network, or by digging into the router's admin panel to temporarily disable fast roaming — but I decided not to weaken my own network's security just to force a capture I already understood the cause of. That felt like the wrong trade for a learning exercise.

## Where I landed

I didn't walk away with a captured handshake, but I did walk away with PineAP fully working, a real understanding of why the capture failed, and a home network that apparently resists a standard technique better than I expected going in. That's not really a failed session — it's a legitimate finding about my own setup.

If I pick this back up, the next things worth trying are PMKID capture (which talks directly to the AP instead of waiting on a client) or building a dedicated practice AP with a known password so I can run the full capture-to-crack pipeline without touching anything real. Neither of those has happened yet — just next on the list.
