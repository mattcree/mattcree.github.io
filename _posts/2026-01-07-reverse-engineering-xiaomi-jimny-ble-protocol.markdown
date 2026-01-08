---
layout: post
title:  "Reverse Engineering the Xiaomi Jimny RC Car's Bluetooth Protocol"
date:   2026-01-07
categories: hardware
tags: bluetooth ble reverse-engineering rc-car xiaomi python apk decompilation iot hardware claude-code ai
---

I bought a Xiaomi Smart RC Car a few years ago - the 1:16 scale Suzuki Jimny (model XMYKC01CM). It's a neat little 4WD crawler that's controlled via Xiaomi's "Mi Smart RC Car" Android app. It sat on a shelf for quite a while, and when I finally got around to playing with it, I decided I wanted to build my own controller rather than use the official app.

Part of the motivation was that the official app (`com.rcfans`) isn't available on Google Play. To get it, you either have to trawl through shady APK mirror sites or install a Chinese app store. Neither option is particularly appealing. Reverse engineering the protocol seemed like the cleaner approach - and more fun.

What follows is my journey reverse engineering the Bluetooth Low Energy protocol, with some help from [Claude Code][claude-code] (Anthropic's CLI tool). Unfortunately, my particular unit appears to have withered on the vine during its time on the shelf - the BLE module was already dying when I started, and eventually let out its magic smoke during debugging. But not before I documented everything.

**A quick note on legality:** Reverse engineering for interoperability is well-protected in most jurisdictions:

- **UK:** The Copyright, Designs and Patents Act 1988 ([Section 50B][cdpa-50b]) explicitly permits decompilation for interoperability. Contractual terms that try to prohibit this are void.
- **EU:** [Directive 2009/24/EC][eu-directive] (the Software Directive) allows observation, study, and testing of software, plus decompilation for interoperability. These rights cannot be overridden by contract.
- **US:** DMCA has exemptions for reverse engineering for interoperability purposes.

Publishing protocol documentation is legal - APIs and protocols aren't copyrightable. This is standard practice in the maker community for IoT devices.

## The Hardware

| Spec | Value |
|------|-------|
| Model | XMYKC01CM |
| CMIIT ID | 2019DP6666 |
| Bluetooth | 4.2 (Low Energy) |
| Battery | 3.7V 500mAh Li-ion polymer |
| Runtime | ~30 minutes |
| Charge Time | ~90 minutes |
| Scale | 1:16 |
| Drive | 4WD |

The car has a single PCB that handles everything - motor control, BLE communications, and charging. The board is labelled "D-BLD" and contains the BLE module alongside the ESC circuitry. Charging is via a USB port hidden in the exhaust pipe - a nice touch.

The LED indicators are helpful for debugging:

| Pattern | Meaning |
|---------|---------|
| Blue blinking | Waiting for connection |
| Blue solid | Connected |
| White pulsing | Charging |
| Fast blue flash | Pairing mode |

## The Reverse Engineering Process

I decompiled the official Android app (`com.rcfans`, version 1.5.7) using [JADX][jadx]. The package name contains a typo - `com.andorid.blerc` instead of `com.android.blerc`. Classic.

The key files I dug through:

- `BleConstant.java` - UUID definitions
- `BaseMainActivity.java` - Main control logic (~1800 lines)
- `BarBaseActivity.java` - BLE service discovery
- `HexUtil.java` - Byte encoding utilities

I had Claude Code help me make sense of the decompiled Java. It was particularly useful for untangling the byte encoding scheme and tracing the connection sequence through multiple classes. Having an AI that can read code and answer questions about it while I'm poking around in a terminal is genuinely useful for this kind of exploratory work.

## The BLE Protocol

### Characteristic UUIDs

The car exposes a custom BLE service with four characteristics:

| UUID | Purpose | Properties |
|------|---------|------------|
| `4fbbffe3-c59c-478d-bb99-d6e06367e344` | Control (steering/throttle) | Write without response |
| `4fbbffe4-c59c-478d-bb99-d6e06367e344` | Configuration commands | Write |
| `4fbbffe5-c59c-478d-bb99-d6e06367e344` | Notifications | Notify |
| `4fbbffe6-c59c-478d-bb99-d6e06367e344` | Lights | Write |

The device advertises with a name like `JimnygreenSN` followed by a serial number.

One thing I found interesting: the official app doesn't filter by service UUID when scanning. It just shows all BLE devices and lets you pick. After connecting, it iterates through every service looking for the characteristic UUIDs. Not the most elegant approach, but it works.

### Control Encoding

This is the interesting bit. Steering and throttle are each 0-2000 values (1000 = centre/stop), packed into just 3 bytes.

**Steering:**
- 0-999: Left (0 = full left)
- 1000: Centre
- 1001-2000: Right (2000 = full right)

**Throttle:**
- 0-999: Reverse
- 1000: Stop
- 1001-2000: Forward

The encoding packs two 12-bit values into 3 bytes:

```
Byte 0: (steering >> 4) & 0xFF        // Upper 8 bits of steering
Byte 1: ((steering << 4) & 0xF0) |    // Lower 4 bits of steering
        ((throttle >> 8) & 0x0F)      // Upper 4 bits of throttle
Byte 2: throttle & 0xFF               // Lower 8 bits of throttle
```

Or visually:

```
Steering (12 bits):  SSSS SSSS SSSS
Throttle (12 bits):  TTTT TTTT TTTT

Byte 0: SSSS SSSS  (steering bits 11-4)
Byte 1: SSSS TTTT  (steering bits 3-0, throttle bits 11-8)
Byte 2: TTTT TTTT  (throttle bits 7-0)
```

It's a clever way to minimise packet size. Control packets should be sent at about 20Hz (every 50ms) using Write Without Response for minimal latency.

### Python Implementation

Here's the encoder in Python:

```python
def encode_control(steering: int, throttle: int) -> bytes:
    steering = max(0, min(2000, steering))
    throttle = max(0, min(2000, throttle))

    byte0 = (steering >> 4) & 0xFF
    byte1 = ((steering << 4) & 0xF0) | ((throttle >> 8) & 0x0F)
    byte2 = throttle & 0xFF

    return bytes([byte0, byte1, byte2])
```

### Lights

Lights are simple - just write a single byte to the lights characteristic:

```python
# Lights on
await client.write_gatt_char(UUID_CHAR4, bytes([0x01]))

# Lights off
await client.write_gatt_char(UUID_CHAR4, bytes([0x00]))
```

### Configuration Commands

These go to the config characteristic:

| Bytes | Purpose |
|-------|---------|
| `[0x20, 0x01]` | Initialisation (send after connecting) |
| `[0x10, 0x04]` | Request firmware version (part 1) |
| `[0x10, 0x05]` | Request firmware version (part 2) |
| `[0x21, 0x01, 0x01]` | Enable drag brake |
| `[0x21, 0x01, 0x00]` | Disable drag brake |

Responses come back via notifications on the notification characteristic.

## Connection Sequence

The full startup flow looks like this:

```python
import asyncio
from bleak import BleakClient, BleakScanner

UUID_CHAR1 = "4fbbffe4-c59c-478d-bb99-d6e06367e344"  # Config
UUID_CHAR2 = "4fbbffe5-c59c-478d-bb99-d6e06367e344"  # Notifications
UUID_CHAR3 = "4fbbffe3-c59c-478d-bb99-d6e06367e344"  # Control
UUID_CHAR4 = "4fbbffe6-c59c-478d-bb99-d6e06367e344"  # Lights
UUID_BATTERY = "00002a19-0000-1000-8000-00805f9b34fb"

async def connect_jimny():
    # Scan for devices
    devices = await BleakScanner.discover(timeout=10.0)

    # Find the Jimny
    jimny = next((d for d in devices if d.name and "Jimny" in d.name), None)
    if not jimny:
        raise Exception("Jimny not found")

    async with BleakClient(jimny.address) as client:
        # Enable notifications
        await client.start_notify(UUID_CHAR2, notification_handler)

        # Send init command
        await client.write_gatt_char(UUID_CHAR1, bytes([0x20, 0x01]))

        # Read battery
        battery = await client.read_gatt_char(UUID_BATTERY)
        print(f"Battery: {battery[0]}%")

        # Ready for control commands on UUID_CHAR3
```

The [Bleak][bleak] library makes this straightforward on macOS, Windows, and Linux.

## The Magic Smoke Incident

From the moment I pulled this thing off the shelf, the BLE module was behaving erratically:

1. It would only appear intermittently in BLE scans
2. Sometimes it wouldn't appear at all, even with the blue LED blinking
3. When it did appear, it couldn't maintain a connection

I was using Claude Code to help run BLE scans while I poked at the hardware. We'd run a 30-second scan, see the device appear once, then run another scan and it would be gone. Classic signs of a component that's already dead and just hasn't fully given up yet.

Suspecting cold solder joints from age, I opened it up to take a look. While removing the battery cable to get better access, the connector on the board snapped off. Great. I got out the soldering iron to reattach it, and while doing so a component next to the connector let out the magic blue smoke.

So that was the end of that.

I suspect several years of sitting on a shelf did it in. Lithium batteries don't love being left in a discharged state, and whatever degradation occurred may have weakened multiple components. Or it was just a dud from the factory that I never noticed because I never properly used it. Either way, it's now very thoroughly dead.

If your Jimny exhibits similar symptoms - appearing and disappearing from BLE scans, failing to connect even when the LED indicates it's ready - the BLE module might be dying. Hopefully yours won't expire quite as dramatically as mine did.

## What's Next?

With the protocol documented, there are some interesting possibilities:

- **ESP32 replacement controller** - Wire an ESP32 directly to the motor/servo connections, bypassing the dead BLE module entirely
- **Home Assistant integration** - Control it via home automation
- **Gamepad support** - Map a physical controller to the BLE protocol
- **Web Bluetooth controller** - Control it from a browser (Chrome supports the Web Bluetooth API)

I've got an untested Python controller and a Web Bluetooth controller in my [jimnyctrl repo][jimnyctrl] if anyone wants to pick this up with a functioning unit.

## In Closing

This was a fun little project, even if my hardware was dead before I could properly test it. The combination of APK decompilation and having Claude Code to help analyse the results worked well. Being able to ask "what does this byte encoding do?" and get an immediate explanation while I'm in the middle of debugging is genuinely useful.

The protocol is fully documented now - I just can't verify it works because my unit never did. If you've got a working Xiaomi Jimny and build something with this protocol, I'd love to hear about it at [mattcree@proton.me](mailto:mattcree@proton.me). Especially if you can confirm the protocol actually works as documented.

[claude-code]: https://github.com/anthropics/claude-code
[jadx]: https://github.com/skylot/jadx
[bleak]: https://github.com/hbldh/bleak
[jimnyctrl]: https://github.com/mattcree/jimnyctrl
[cdpa-50b]: https://www.legislation.gov.uk/ukpga/1988/48/section/50B
[eu-directive]: https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32009L0024
