# biurko-ha — Home Assistant control for an SHD20 standing desk

Reverse-engineered the serial protocol between the control panel and the SHD20 controller box of an electric standing desk, then built a man-in-the-middle device on an XIAO ESP32-C6 that lets Home Assistant read the desk height and recall presets — while the original panel keeps working.

No soldering to the desk's own PCBs: the device sits inline on the RJ45 cable.

<!-- TODO: photo of the finished device under the desk + short video/GIF of a preset move -->

## Result

- Live desk height in Home Assistant
- **Closed-loop height control:** set any target height (70–115 cm) from a slider; the ESP32 drives the desk there and stops on its own
- Presets 1/2/3, manual up/down and an emergency stop exposed as HA entities
- Physical panel buttons still work and are visible in HA
- Verified end to end: preset move 70.0 → 105 cm, confirmed in logs
- Powered from the desk's own 5 V line; connected over Thread (Wi-Fi at the desk is −90 dBm)

## Hardware

| Part | Notes |
| --- | --- |
| Controller box | SHD20, 24 VDC 3.8 A, duty cycle 2 min ON / 18 min OFF |
| Control panel | PCB 006F-SA32_V7.0, display + buttons 1/2/3/M/up/down |
| MCU | Seeed XIAO ESP32-C6 |
| Level shifter | BSS138 bidirectional, 5 V ↔ 3.3 V, both data lines |
| Connectors | NEKU cat.5e keystone (panel side), cut T568B patch cord (box side) |
| Enclosure | Designed in Fusion 360, FDM printed, 77 × 77 × 31 mm |

## Step 1 — Mapping the connector

Measured each pad on the panel PCB, then traced it to the RJ45 pins.

| RJ45 pin | Signal | Idle voltage |
| --- | --- | --- |
| 1 | DIS-TX (box → panel) | 3.34 V |
| 2 | DIS-RX (panel → box) | 4.08 V |
| 5 | GND | — |
| 7 | SWD | — |
| 8 | DIS-5V | 4.89 V |

Box-side logic is 5 V, so the ESP32 needs a level shifter on both lines.

## Step 2 — Decoding the protocol

Captured UART traffic in both directions: **9600 baud, 8N1**.

**Box → panel** (display data):

```
5A  D1  D2  D3  01  CHK
CHK = (D1 + D2 + D3 + 0x01) & 0xFF
```

D1–D3 are 7-segment patterns for the three digits; bit 7 is the decimal point.

**Panel → box** (key state, sent continuously every ~20 ms):

```
A5  00  KEY  ~KEY  FF
```

| KEY | Action |
| --- | --- |
| 0x00 | none |
| 0x01 | M |
| 0x02 | preset 1 |
| 0x04 | preset 2 |
| 0x08 | preset 3 |
| 0x20 | up |
| 0x40 | down |

**Key finding:** the box stops transmitting if it does not receive a continuous stream from the panel. A passive sniffer is not enough — the device has to forward traffic both ways, in real time, and inject its own key codes into that stream.

## Step 3 — Man-in-the-middle device

```
 controller box  ⇄  [BSS138]  ⇄  XIAO ESP32-C6  ⇄  [BSS138]  ⇄  panel
                                       │
                                  Thread → Home Assistant
```

| XIAO pin | Direction |
| --- | --- |
| D0 / GPIO0 | ← box |
| D1 / GPIO1 | → panel |
| D2 / GPIO2 | ← panel |
| D3 / GPIO21 | → box |

The ESP32 relays every frame, decodes the 7-segment bytes into a height value, and replaces the panel's key byte with its own frames when Home Assistant issues a command.

Firmware: ESPHome (ESP-IDF), Thread via `openthread`. Configuration: [`esphome/biurko.yaml`](esphome/biurko.yaml). Secrets go in `esphome/secrets.yaml` (see the `.example` file).

## Step 4 — Height control loop

The controller only understands "up", "down" and presets, so arbitrary heights need a control loop on the ESP32 (every 50 ms):

- **Speed estimation** — derivative of the decoded height, exponentially smoothed. Readings above 100 cm jump in whole centimetres, so the raw derivative is clipped to ±4 cm/s (the desk physically moves at ~3 cm/s).
- **Predictive braking** — the desk coasts after the key is released, so the loop stops early by `tolerance + speed × lead`, with lead times tuned separately for up (0.445 s) and down (0.42 s).
- **Resolution-aware tolerance** — below 100 cm the display shows 0.1 cm, above it only whole centimetres; tolerance switches 0.25 → 0.6 cm and targets above 100 cm are rounded.
- **One-shot correction** — after the desk settles (1 s still), if the error is still above 0.6 cm it makes one more approach.
- **Safety stops** — abort after 40 s of travel or 3 s without movement.

Two details that took debugging to find:

- **Wake-up pulse** — the controller goes to sleep after a few seconds of inactivity and swallows the first keypress. Any command after >8 s of silence is preceded by a 40 ms pulse (too short to move the motor) and a short gap.
- **Watchdog by design** — the movement command is only ever injected 80 ms ahead and refreshed by the loop. If the loop stalls, injection expires and the desk stops by itself.

## Problems solved along the way

- **ESP32 would not boot from the desk's 5 V** — the low-voltage side of the shifter read only 2 V. Suspected the controller could not supply enough current; the real cause was a broken wire in the supply conductor. Replacing the cable fixed it.
- **Device dropping off Wi-Fi** — RSSI below −90 dBm at the desk. Migrated to Thread (the home network already has a Thread border router), which also removed the dependency on router placement.
- **Keystone would not latch in the printed wall** — 1.2 mm wall held but wobbled, 1.5 mm held rigidly. That pushed the keystone 7–8 mm proud of the face, so the final design uses a removable mounting wall in guide rails with a 7.5 mm recess.

## Repository

```
esphome/       ESPHome configuration (secrets excluded)
docs/          protocol captures, photos, wiring diagram   (coming)
enclosure/     STL files                                   (coming)
```

## Possible next steps

- Respect the controller's 2 min ON / 18 min OFF duty cycle in software
- Time limit on manual up/down switches

---

Author: Przemysław Pers · [LinkedIn](https://www.linkedin.com/in/przemysław-pers-b2324120b)
