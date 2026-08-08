---
title: "Messing with an ESP32-C3 Super Mini"
date: 2026-08-08
draft: false
tags: ["esp32", "arduino", "ubuntu", "electronics"]
---

A HackerBox dropped an ESP32-C3 Super Mini on my desk — a thumb-sized board with WiFi, Bluetooth, and a USB-C port, all for a couple of bucks. This is the whole path from sealed bag to "it's printing every WiFi network in my house," including the ten minutes Ubuntu 24.04 spent flat-out refusing to open the Arduino IDE. If you've got the same board and the same OS, you can follow along and skip the head-scratching.

The one rule I set for myself: prove each piece works before adding the next. Get the IDE running, then get the board blinking, then get the radio doing something. That way if anything breaks, you know exactly which thing broke.

## The board

The Super Mini is tiny — smaller than a stick of gum — and it's got a RISC-V core with WiFi and BLE built in. It uses **native USB**, which matters in a couple of small ways later, so file that away.

## Getting the Arduino IDE to launch on Ubuntu 24.04

I grabbed the Linux AppImage from arduino.cc, made it executable, and tried to run it:

```bash
cd ~/Downloads
chmod +x arduino-ide_2.3.10_Linux_64bit.AppImage
./arduino-ide_2.3.10_Linux_64bit.AppImage
```

And it immediately face-planted:

```
[FATAL:setuid_sandbox_host.cc(158)] The SUID sandbox helper binary was found,
but is not configured correctly. Rather than run without sandboxing I'm aborting
now. You need to make sure that /tmp/.mount_.../chrome-sandbox is owned by root
and has mode 4755.
Trace/breakpoint trap (core dumped)
```

Here's the gotcha, and it's got nothing to do with the download being bad. The Arduino IDE 2.x is an Electron app (Chromium under the hood), and **Ubuntu 24.04 tightened the kernel setting that Electron's sandbox relies on.** So it aborts rather than run unsandboxed. This bites basically everyone on 24.04.

The fix that got me moving: just tell it to skip the sandbox.

```bash
./arduino-ide_2.3.10_Linux_64bit.AppImage --no-sandbox
```

That launched it straight away. The "sandbox" here is browser-renderer isolation meant for loading untrusted web pages — not a big deal for a local dev tool you're running yourself. You do have to pass the flag every time you launch; I just retype it, but you could alias it if that gets old.

## The serial-port permission thing

On Ubuntu your user usually can't touch the serial port by default, which shows up later as "the board won't upload" even when everything else is fine. Get ahead of it by adding yourself to the `dialout` group:

```bash
sudo usermod -a -G dialout $USER
```

The catch: **this doesn't take effect until you log out and back in** (or reboot). Group membership is set at login, so your current terminal won't know about it. Do it now so you're not debugging a phantom permissions error later.

## Board support and settings

Inside the IDE, I added ESP32 support. **File → Preferences**, and in "Additional boards manager URLs" paste:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Then **Tools → Board → Boards Manager**, search **esp32**, install the Espressif package. It's a chunky download, give it a few minutes.

Then three settings that matter for this board specifically:

- **Board:** "ESP32C3 Dev Module"
- **USB CDC On Boot:** **Enabled** — without this, the Serial Monitor stays blank, because the C3 talks over native USB rather than a separate serial chip.

The Port setting comes next, once we confirm the board's actually there.

## Is the board even seen?

Before uploading anything, I checked that the OS could see the board:

```bash
ls /dev/ttyACM* /dev/ttyUSB*
```

```
ls: cannot access '/dev/ttyUSB*': No such file or directory
/dev/ttyACM0
```

That output looks half-broken but it's exactly right. Because the C3 uses native USB, it shows up as **`/dev/ttyACM0`**, *not* `/dev/ttyUSB0`. The "no such file" for `ttyUSB` is normal — that path is for boards with a separate USB-to-serial chip, which this one doesn't have. So: `ttyACM0` present, `ttyUSB` absent = everything's fine.

Back in the IDE I set **Tools → Port → /dev/ttyACM0**.

One more note if your board *doesn't* show up at all: try a different USB-C cable first. A shocking number of cables are charge-only with no data lines, and that's the number-one reason a board looks dead.

## Blink, to prove the whole chain works

The classic first test. But there's a Super Mini quirk worth knowing: the onboard LED is on **GPIO8**, and it's **active-low** — `LOW` turns it *on*, `HIGH` turns it *off*. If you write the "obvious" blink and it seems inverted, that's why.

```cpp
// ESP32-C3 Super Mini — blink + serial heartbeat
// Onboard LED is on GPIO8 and is ACTIVE-LOW (LOW = on).

#define LED_PIN 8

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  digitalWrite(LED_PIN, LOW);    // LED on
  Serial.println("blink: ON");
  delay(300);
  digitalWrite(LED_PIN, HIGH);   // LED off
  Serial.println("blink: OFF");
  delay(300);
}
```

Hit upload. For me it flashed automatically — no button-holding needed. (If yours hangs at "Connecting...", the trick is to hold the **BOOT** button, tap **RESET**, release BOOT, and upload again to force download mode. I didn't have to, but some of these boards do.)

The LED blinked. That's the entire toolchain proven — IDE, board package, port, permissions, upload — all in one tiny working sketch.

## The first actually-useful thing: a WiFi scanner

Blinking is nice, but the reason I wanted this board is the radio. So the next test was to make it *see* — scan for WiFi networks and print them to serial. This needs zero extra hardware; the Serial Monitor is your whole display.

I wanted it to look like a tool, not a debug dump, so it sorts strongest-signal-first and draws little ASCII signal bars. It rescans on a loop, which turns out to be a decent dead-zone finder — walk the board around and watch the numbers move.

```cpp
/* WiFi SCANNER — ESP32-C3 Super Mini (serial output, no extra hardware)
 * Scans 2.4GHz on a loop, prints a table sorted by signal strength.
 * Open the Serial Monitor at 115200 baud.
 */

#include <WiFi.h>

#define RESCAN_MS 8000     // pause between scans

static const char *securityLabel(wifi_auth_mode_t enc) {
  switch (enc) {
    case WIFI_AUTH_OPEN:            return "OPEN";
    case WIFI_AUTH_WEP:             return "WEP";
    case WIFI_AUTH_WPA_PSK:         return "WPA";
    case WIFI_AUTH_WPA2_PSK:        return "WPA2";
    case WIFI_AUTH_WPA_WPA2_PSK:    return "WPA/2";
    case WIFI_AUTH_WPA2_ENTERPRISE: return "WPA2-ENT";
    case WIFI_AUTH_WPA3_PSK:        return "WPA3";
    case WIFI_AUTH_WPA2_WPA3_PSK:   return "WPA2/3";
    default:                        return "?";
  }
}

// Build a 5-segment signal bar like "[####.]" from an RSSI value
static void signalBar(int rssi, char *out) {
  int bars;
  if      (rssi >= -55) bars = 5;
  else if (rssi >= -65) bars = 4;
  else if (rssi >= -72) bars = 3;
  else if (rssi >= -80) bars = 2;
  else                  bars = 1;
  out[0] = '[';
  for (int i = 0; i < 5; i++) out[1 + i] = (i < bars) ? '#' : '.';
  out[6] = ']';
  out[7] = '\0';
}

void setup() {
  Serial.begin(115200);
  delay(300);
  WiFi.mode(WIFI_STA);     // station mode, not connecting to anything
  WiFi.disconnect();
  delay(100);
  Serial.println();
  Serial.println("=== ESP32-C3 WiFi Scanner ===");
}

uint32_t scanCount = 0;

void loop() {
  scanCount++;
  Serial.printf("\n--- scan #%lu ---\n", (unsigned long)scanCount);

  int n = WiFi.scanNetworks(false, true);   // blocking; show hidden too

  if (n <= 0) {
    Serial.println("No networks found.");
    WiFi.scanDelete();
    delay(RESCAN_MS);
    return;
  }

  // Sort indices by RSSI, strongest first
  int idx[64];
  if (n > 64) n = 64;
  for (int i = 0; i < n; i++) idx[i] = i;
  for (int i = 1; i < n; i++) {
    int key = idx[i];
    int j = i - 1;
    while (j >= 0 && WiFi.RSSI(idx[j]) < WiFi.RSSI(key)) {
      idx[j + 1] = idx[j];
      j--;
    }
    idx[j + 1] = key;
  }

  Serial.println(" #  SIGNAL   RSSI  CH  SECURITY  BSSID              NETWORK");
  Serial.println("--------------------------------------------------------------------");

  int openCount = 0;
  for (int r = 0; r < n; r++) {
    int i = idx[r];
    int rssi = WiFi.RSSI(i);
    char bar[8];
    signalBar(rssi, bar);

    wifi_auth_mode_t enc = WiFi.encryptionType(i);
    if (enc == WIFI_AUTH_OPEN) openCount++;

    String ssid = WiFi.SSID(i);
    if (ssid.length() == 0) ssid = "<hidden>";

    Serial.printf("%2d  %s  %4d  %2d  %-8s  %-17s  %s\n",
                  r + 1, bar, rssi, WiFi.channel(i),
                  securityLabel(enc), WiFi.BSSIDstr(i).c_str(),
                  ssid.c_str());
  }

  Serial.println("--------------------------------------------------------------------");
  Serial.printf("%d networks", n);
  if (openCount > 0) Serial.printf("  (%d OPEN — no password)", openCount);
  Serial.println();

  WiFi.scanDelete();
  delay(RESCAN_MS);
}
```

Upload it the same way, open the Serial Monitor at **115200**, and you get something like:

```
 #  SIGNAL   RSSI  CH  SECURITY  BSSID              NETWORK
--------------------------------------------------------------------
 1  [#####]   -41   6  WPA2      a4:2b:8c:11:22:33  HomeWiFi
 2  [####.]   -63  11  WPA2/3    de:ad:be:ef:00:11  Neighbor_5G
 3  [##...]   -78   1  OPEN      12:34:56:78:9a:bc  xfinitywifi
--------------------------------------------------------------------
3 networks  (1 OPEN — no password)
```

Two honest caveats. The C3 is **2.4GHz only** — it has no 5GHz radio — so if a network shows up on your phone but not here, it's probably a 5GHz-only one. And the scan is *blocking*: the board freezes for the couple of seconds it takes to scan. Totally fine for a standalone tool like this, but something to keep in mind if you ever want the board doing other things at the same time.

## Where I'm stopping (for now)

That's the point I called it. In one sitting the board went from unknown-object-in-a-bag to a working toolchain and a little WiFi tool that actually reads the air. The two scary unknowns — *will it flash* and *will the radio work* — are both answered, which is honestly the risky part of any new-board project.

## The actual goal: a thing I can walk around with

The serial scanner is fun, but tethered to a laptop it's not really a *tool* — it's a demo. What I actually want to build is a little self-contained handheld: the ESP32-C3 driving a small color screen, with a rotary knob to twist between views. Twist to flip from a **radar sweep** (devices plotted as blips, signal strength as distance from the center) to a **channel-congestion graph** to a **live probe-request feed**, press the knob to freeze a view. Battery on board, no laptop, no wires — something I can pocket and carry around the house or on a walk and just *see* the RF around me.

None of that is built yet. This section is the shopping list and the plan, not a build log — but writing it down is half of talking myself into it. Here's what I'll need beyond the board itself:

**The essentials — makes it a device at all**
- **ST7789 IPS display, 240x240, SPI** (1.3" or 1.54"). The square resolution fits a radar circle perfectly. *Buy the version with the CS pin broken out* — the cheaper "no-CS" boards tie that line low internally and become a headache the moment you add anything else to the SPI bus.
- **EC11 rotary encoder with a push button** — twist to change view, press to select or freeze. Only needs three GPIOs and feels great. (A couple of plain tactile buttons would also work if I want to keep it dead simple.)

**Portability — so it's not tethered**
- **A single LiPo cell**, roughly 500–1000mAh, with a JST connector.
- **A combined LiPo charge + 5V boost module.** This is the clean path: it recharges the cell over USB *and* gives a stable 5V to feed the board. The gotcha I want to remember: **don't feed a raw LiPo straight into the 3.3V pin** — a full cell sits at 4.2V and will over-volt that rail. The boost module sidesteps the whole problem. (A bare TP4056 only *charges* — it won't give you a stable 5V — so it needs a boost stage alongside it, or just get the combined board.)

**Assembly**
- **A half-size breadboard and dupont jumper wires** to get it all working *before* committing anything to solder. Non-negotiable for a first build.
- **Perfboard, header pins, and solder** for the permanent version once it works on the breadboard.

**Optional / down the line**
- **An enclosure** — a 3D-printed case or a small project box. Honestly the bare exposed-PCB-and-wires look is very on-theme, so this is genuinely optional.
- **A second ESP32-C3**, if I ever want two of these talking directly to each other over ESP-NOW — no router, no internet. Not needed for the scanner, just a tempting rabbit hole.

That's a satisfyingly cheap pile — the screen and the encoder are a few dollars each. When I actually wire it up, that'll be its own post, gotchas and all.

If you're starting from the same HackerBox and the same Ubuntu, the takeaway is: the `--no-sandbox` flag and the `dialout` group are the two things that'll trip you up before you even get to write code. Clear those two and the rest is smooth.
