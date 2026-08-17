# BMW Motorrad LIN Signal Analysis

[한국어](LIN_ANALYSIS-kr.md)

> **Reference bike:** BMW S1000XR (2016)  
> Captures and decode notes for other models / years are welcome.

For parsing and UI implementation, see:

- [Cluster application](https://github.com/kendrickkim/bmw-motorrad-cluster-application)
- [ESP32 firmware](https://github.com/kendrickkim/bmw-motorrad-cluster-esp32)

---

## Contents

1. [Frame notation](#frame-notation)
2. [Message ID overview](#message-id-overview)
3. [ID `0x14` — Left handle / Wonder Wheel](#id-0x14--left-handle--wonder-wheel)
4. [ID `0x20` — Ignition / rear-wheel RPM & speed](#id-0x20--ignition--rear-wheel-rpm--speed)
5. [ID `0x25` — Odometer](#id-0x25--odometer)
6. [ID `0x2B` / `0x2E`](#id-0x2b--0x2e)
7. [ID `0x61` — Coolant temperature](#id-0x61--coolant-temperature)
8. [ID `0xE2` — RPM / gear / voltage](#id-0xe2--rpm--gear--voltage)

---

## Frame notation

LIN frames are treated as **1-byte Message ID + 7 data bytes**.

### Byte index

| Index | Name | Description |
|:-----:|:----:|-------------|
| 0 | Message ID | Frame identifier |
| 1 | Data-0 | Payload |
| 2 | Data-1 | Payload |
| 3 | Data-2 | Payload |
| 4 | Data-3 | Payload |
| 5 | Data-4 | Payload |
| 6 | Data-5 | Payload |
| 7 | Data-6 | Payload |

Some fields split a byte into **nibbles (4 bits)**.

### Bit order

Bit **7 = MSB**, bit **0 = LSB**.

Example: value `0xA1`

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 1 | 0 | 1 | 0 | 0 | 0 | 0 | 1 |

- Bits 7–4 → high nibble (MSB nibble) = `0xA`
- Bits 3–0 → low nibble (LSB nibble) = `0x1`

---

## Message ID overview

| ID | Status | Contents |
|:--:|:------:|----------|
| `0x14` | Decoded | Left-handle buttons, Wonder Wheel In/Out & wheel value |
| `0x20` | Decoded | Ignition, rear-wheel RPM → road speed |
| `0x25` | Decoded | Odometer (km) |
| `0x2B` | Unknown | — |
| `0x2E` | Unknown | — |
| `0x61` | Decoded | Coolant temperature |
| `0xE2` | Decoded | Engine RPM, gear, battery voltage |

---

## ID `0x14` — Left handle / Wonder Wheel

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Unknown | — |
| Data-1 | Decoded | Left Handle Controller 1 (button bitmap) |
| Data-2 | Decoded | Left Handle Controller 2 (button bitmap) |
| Data-3 | Decoded | Wonder Wheel In / Out |
| Data-4 | Unknown | — |
| Data-5 | Decoded | Wonder Wheel rotation value |
| Data-6 | Unknown | — |

### Data-1 — Left Handle Controller 1

Bike: S1000XR and similar

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Horn | Turn cancel | Turn right | Turn left | Cruise | Hazard | ABS | ESA |

### Data-2 — Left Handle Controller 2

Bike: S1000XR and similar

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| ? | ? | Trip | Info | Cruise Set | Cruise Reset | High beam | ? |

### Data-3 — Wonder Wheel In / Out

Bike: S1000XR and similar

| Value | Meaning |
|:-----:|---------|
| `0xFD` | Out |
| `0xFE` | In |

### Data-5 — Wonder Wheel rotation value

Bike: S1000XR and similar

- Range: `0x00` – `0xFF`
- Value changes as the wheel is rotated.

---

## ID `0x20` — Ignition / rear-wheel RPM & speed

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Unknown | — |
| Data-1 | Decoded | Ignition (`0x7F` = ON) |
| Data-2 | Decoded | Rear-wheel RPM low 8 bits |
| Data-3 | Decoded | Rear-wheel RPM high 4 bits (nibble) |
| Data-4…6 | Unknown | — |

### Data-1 — Ignition

| Value | Meaning |
|:-----:|---------|
| `0x7F` | Ignition ON |

### Data-2 / Data-3 — Rear-wheel RPM & road speed

```text
rear_wheel_rpm = ((Data-3 & 0xF0) << 8) + (Data-2 & 0xFF)
speed_kmh      ≈ rear_wheel_rpm × 0.121
```

**Tire assumption (rear 190/55ZR17)**

| Item | Value |
|------|-------|
| Effective diameter | 642 mm |
| Distance per revolution | 2017 mm |

```text
speed_kmh = (rear_wheel_rpm × 60 × 2017) / 1000 / 1000
```

> Which nibble of Data-3 holds the high 4 bits (`0xF0` vs `0x0F`) should be re-verified from captures. Firmware may use a different mask.

---

## ID `0x25` — Odometer

```text
odometer_km = (Data-2 × 65536) + (Data-1 × 256) + Data-0
```

Equivalent to `(Data-2 << 16) | (Data-1 << 8) | Data-0`.

> Indexes are **Data-n** (excluding Message ID). In a raw buffer that is `buf[1]…buf[3]`.

---

## ID `0x2B` / `0x2E`

| ID | Status |
|:--:|:------:|
| `0x2B` | Unknown |
| `0x2E` | Unknown |

---

## ID `0x61` — Coolant temperature

```text
coolant_celsius = Data-2 − 40
```

---

## ID `0xE2` — RPM / gear / voltage

| Data | Status | Description |
|:----:|:------:|-------------|
| Data-0 | Decoded | Low 8 bits of RPM/5 |
| Data-1 | Decoded | High nibble = gear; low nibble = high 4 bits of RPM/5 |
| Data-2…4 | Unknown | — |
| Data-5 | Decoded | Battery voltage |
| Data-6 | Unknown | — |

### Data-0 / Data-1 — Engine RPM

```text
rpm = (((Data-1 & 0xF0) << 8) + (Data-0 & 0xFF)) × 5
```

> The **high nibble of Data-1 is also gear**. Shipping firmware may use the low nibble (`Data-1 & 0x0F`) for the RPM high bits — always cross-check against the deployed parser.

### Data-1 — Gear (high nibble)

| Value | Meaning |
|:-----:|---------|
| `0x10` | 1st |
| `0x20` | N |
| `0x40` | 2nd |
| `0x70` | 3rd |
| `0x80` | 4th |
| `0xB0` | 5th |
| `0xD0` | 6th |
| `0xF0` | Between gears |

### Data-5 — Battery voltage

```text
battery_v = (Data-5 & 0xFF) × 0.1
```

---

## Contributing

- Include capture logs, bike model/year, Message ID, Data index, bit/mask, and interpretation.
- Leave unknown fields as `?` — do not guess button semantics.
