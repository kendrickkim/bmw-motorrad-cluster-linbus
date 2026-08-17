# BMW Motorrad Cluster — LIN Bus

[한국어](README-kr.md)

Capture LIN bus signals from a BMW Motorrad motorcycle, decode vehicle and control data, and forward them to an Android cluster app (including Wonder Wheel / left-handle controls as Bluetooth keyboard input).

This repository is the **signal-analysis and documentation hub**. Firmware and the Android app live in related repositories.

| Repository | Role |
|---|---|
| **This repo** (`bmw-motorrad-cluster-linbus`) | LIN frame analysis, connector docs, decoded fields |
| [bmw-motorrad-cluster-esp32](https://github.com/kendrickkim/bmw-motorrad-cluster-esp32) | ESP32 firmware (LIN sniff → BLE / BT HID) |
| [bmw-motorrad-cluster-application](https://github.com/kendrickkim/bmw-motorrad-cluster-application) | Android Compose cluster UI |

## Goals

1. Capture LIN traffic from the bike
2. Document and decode message IDs / payloads
3. Deliver decoded data to the Android cluster app
4. Map Wonder Wheel and left-handle controller events to keyboard-like input
5. Send that input over Bluetooth to the phone used as the cluster / head unit

## Demo

<p align="center">
  <a href="https://www.youtube.com/watch?v=BmVxNDDYLkI">
    <img src="https://img.youtube.com/vi/BmVxNDDYLkI/0.jpg" width="500" alt="Project demo video"/>
  </a>
</p>

[Watch on YouTube](https://www.youtube.com/watch?v=BmVxNDDYLkI)

## Architecture

```text
BMW Motorrad (LIN)
        │
        ▼
 LIN transceiver (e.g. TJA1021) + ESP32 / ESP32-S3
        │
        ▼
 Bluetooth (BLE / HID keyboard for Wonder Wheel & controls)
        │
        ▼
 Android phone (cluster app) + Android Auto receiver (e.g. Headunit Reloaded)
```

![System structure](images/structure.png)

## Requirements

### Motorcycle

- A compatible BMW Motorrad model. See the [Chigee BMW Quick Release compatibility list](https://support.chigee.com/portal/en/kb/articles/bmw-quick-release-module-compatible-with-my-bmw#Check_the_compatibility).
- **SA272** (ConnectedRide Cradle / related connectivity option) must be enabled. Models on the list without SA272 need a dealer activation (in Korea this has been reported as free at some dealers, e.g. Daegu Motorrad).
- Background on SA272 / ConnectedRide Cradle: [BMW Group Press](https://www.press.bmwgroup.com/global/article/detail/T0359072EN/bmw-motorrad-presents-the-connectedride-cradle?language=en).

Analysis notes in this repo are based on a **BMW S1000XR (2016)** unless stated otherwise. Contributions for other models are welcome.

### Hardware & software

| Item | Notes |
|---|---|
| Android phone | Build/install the [cluster application](https://github.com/kendrickkim/bmw-motorrad-cluster-application) |
| ESP32 board | Verified on ESP32 and ESP32-S3 (Arduino platform) |
| LIN transceiver | TJA1021-based modules recommended (e.g. AliExpress) |
| Android Auto receiver | This project used Headunit Reloaded |

### Skills (if you build the full stack)

- Android (Jetpack Compose)
- Hardware / digital signal analysis
- ESP32 (Arduino) and BLE / HID concepts

## Physical connection (S1000XR)

On the S1000XR, remove the left fairing to reach the navigation / cradle socket:

![S1000XR nav socket](images/s1000xr_nav_socket.png)

Connector pinout:

![KSLIN connector](images/kslin_connector.png)

| Pin | Function | Notes |
|---|---|---|
| 1 | GND | Ground |
| 2 | LIN | Bus signal |
| 3 | 12V | Present when the bike is on |

Matching sockets are commonly available from online marketplaces.

## Why decode LIN?

BMW Motorrad exposes vehicle data over CAN and LIN. LIN is slower and cheaper than CAN, so fewer signals are available, but capture is simpler.

Decoded data can drive a phone-based cluster (speed, RPM, gear, voltage, etc.). The **Wonder Wheel** and left-handle controls can also steer the phone — useful when the phone is used as a navigation / head-unit display.

![Left handle controller](images/left_handle_controller.png)

## Decoded signals (as of 2025-06-19)

| Available | Still needed / requested |
|---|---|
| Speed | Engine warning |
| RPM | Turn-signal activity |
| Wonder Wheel | Coolant temperature *(partially documented in analysis notes)* |
| Left handle controller | Odometer *(documented in analysis notes; verify on more bikes)* |
| Gear | Remaining fuel |
| Battery voltage | Lighting status |
| | Front/rear brake pressure |

Detailed byte/bit layouts live in [`lin_analysis/LIN_ANALYSIS.md`](lin_analysis/LIN_ANALYSIS.md) ([한국어](lin_analysis/LIN_ANALYSIS-kr.md)).

## Repository layout

```text
bmw-motorrad-cluster-linbus/
├── README.md                 # English (this file)
├── README-kr.md              # Korean
├── images/                   # Connector, architecture, UI photos
└── lin_analysis/
    ├── LIN_ANALYSIS.md       # Message ID decode tables (English)
    └── LIN_ANALYSIS-kr.md    # Korean
```

## Related work

- Cluster app: [bmw-motorrad-cluster-application](https://github.com/kendrickkim/bmw-motorrad-cluster-application)
- ESP32 firmware: [bmw-motorrad-cluster-esp32](https://github.com/kendrickkim/bmw-motorrad-cluster-esp32)

## Contributing

Riders and engineers are welcome to extend decode tables, verify other model years, and improve firmware / app integration. Open an issue or PR with capture logs, bike model/year, and decoded field notes.
