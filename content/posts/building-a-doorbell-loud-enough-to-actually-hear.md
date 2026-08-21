---
title: "Building a Doorbell Loud Enough for Me to Actually Hear"
date: 2026-08-21
draft: false
tags: ["esp32", "diy", "homelab"]
---

I don't hear people knock. Never have, not reliably anyway. Meanwhile I've got a spare ESP32 lying around and an old computer wired into a very loud PA system that mostly just sits there unused. Putting those two things together to make a doorbell I can actually hear felt obvious once I thought about it, so that's what I built.

Here's how it went, snags included.

## The plan

The chain is simple: a button gets pressed, an ESP32 notices, it fires an HTTP request over WiFi to an old ThinkPad T410 sitting inside, and that computer plays a chime through the PA. No hub, no cloud service, just a button and some code.

## Turns out I didn't actually have a button

I was sure I had a spare momentary switch somewhere. I did not. Rather than spend money on one — this was end-of-month broke — I went digging through old computer parts instead. The front panel of an old PC case has exactly what I needed: the power/reset button is just a tiny momentary switch on a 2-pin header, already built for this exact job.

## Wiring it to the ESP32-C3

I'm using an ESP32-C3 Super Mini for the button board, programmed from an Ubuntu laptop through the Arduino IDE. A couple of things about this specific board worth knowing before you wire anything:

- It doesn't need an external resistor for the button — the internal pull-up handles it, so the switch just goes from a GPIO pin straight to ground.
- Not every GPIO is safe to use. GPIO 2, 8, and 9 are strapping pins that affect how the chip boots, so I kept the button off those and used GPIO4 instead.

Setting up Arduino IDE on Ubuntu for this board took a few extra steps beyond just installing it:

```bash
sudo snap install arduino
```

Then in Preferences, add the ESP32 board package URL:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Install the esp32 package from Boards Manager, then under Tools select **ESP32C3 Dev Module** as the board, and turn on **USB CDC On Boot** — the Super Mini talks over its native USB port rather than a separate USB-serial chip, and this setting is what makes Serial Monitor output actually show up.

## The upload that wouldn't upload

First real snag. I plugged the board in, hit Upload, and got this:

```
esptool v5.3.1
Serial port /dev/ttyS4:
Connecting......................................
A fatal error occurred: Failed to connect to ESP32-C3: No serial data received.
```

The giveaway was `/dev/ttyS4` — that's not the ESP32. It's a placeholder legacy serial port that basically every Linux machine has, connected or not. The real board almost always shows up as something like `/dev/ttyACM0`. Easiest way to find it:

```bash
ls /dev/tty*         # note what's there
# unplug the board, plug it back in
ls /dev/tty*         # the new entry is your board
```

Selecting the right port in Tools > Port is the fix most people need for this exact error. For me it turned out to be something else entirely — the USB-C cable I was using was charge-only, no data lines. Swapped it for a cable I knew synced a phone before, and the board connected on the first try.

If you hit this error: check the port first, but don't rule out the cable if the correct port still won't respond.

## Powering it

I went back and forth on how to power the ESP32. My first plan was a 3000mAh battery mounted outside with the button, but I live somewhere hot, and cooking a li-ion cell in an outdoor enclosure all summer isn't a great idea — heat is genuinely bad for those. So I flipped the design: the ESP32 stays inside, plugged into a wall charger, and just two wires run out to the switch. The switch doesn't care about voltage or current, so running it a distance on scrap wire is a non-issue.

I'd also wondered whether an older USB port on a laptop would supply enough current — the WiFi radio pulls sharp brief spikes that can brown out a marginal supply — but I sidestepped the question entirely and just used a wall charger. One less thing to worry about.

## The listener, on the T410

The other half runs on an old ThinkPad T410 running Linux Mint, wired into the PA. It's a small Flask app that listens for the ESP32's request and plays an mp3 through `mpg123`:

```python
@app.route('/doorbell', methods=['POST'])
def doorbell():
    data = request.get_json(silent=True) or {}
    if data.get('token') != SECRET_TOKEN:
        return jsonify({"error": "unauthorized"}), 403

    subprocess.Popen(['mpg123', '-q', SOUND_FILE])
    return jsonify({"status": "playing"}), 200
```

I set that up as an actual systemd service, so it starts on its own and doesn't depend on anyone being logged into the desktop:

```ini
[Unit]
Description=PA Doorbell HTTP listener
After=network-online.target sound.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/USERNAME/doorbell/listener.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

And since a suspended laptop drops off the network and can't hear the ESP32 at all, I disabled sleep on it entirely rather than mess with Wake-on-LAN:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

The T410's only job now is to sit there, stay awake, and listen.

## It works

Bench-tested it — button, ESP32, WiFi, the whole chain — and it went through clean. Button press, PA chime, no fuss. My solder held too, which after years of not doing much soldering felt like its own small win.

What's not done yet: the outdoor wire run and the actual weatherproof enclosure for the switch. Right now it's all sitting on my desk working perfectly and doing nothing for anyone at the front door. That's next.
