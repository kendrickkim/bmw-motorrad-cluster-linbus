# BMW Motorrad LIN 신호 분석

[English](LIN_ANALYSIS.md)

> **기준 차종:** BMW S1000XR (2016)  
> 다른 모델·연식 캡처 및 분석 기여를 환영합니다.

구현(파싱·UI)은 아래 저장소를 참고하세요.

- [클러스터 애플리케이션](https://github.com/kendrickkim/bmw-motorrad-cluster-application)
- [ESP32 펌웨어](https://github.com/kendrickkim/bmw-motorrad-cluster-esp32)

---

## 목차

1. [프레임 표기 규칙](#프레임-표기-규칙)
2. [Message ID 요약](#message-id-요약)
3. [ID `0x14` — 좌측 핸들 / Wonder Wheel](#id-0x14--좌측-핸들--wonder-wheel)
4. [ID `0x20` — 이그니션 / 후륜 RPM·속도](#id-0x20--이그니션--후륜-rpm속도)
5. [ID `0x25` — 적산거리](#id-0x25--적산거리)
6. [ID `0x2B` / `0x2E`](#id-0x2b--0x2e)
7. [ID `0x61` — 냉각수 온도](#id-0x61--냉각수-온도)
8. [ID `0xE2` — RPM / 기어 / 전압](#id-0xe2--rpm--기어--전압)

---

## 프레임 표기 규칙

바이크 LIN 프레임은 **Message ID 1바이트 + 데이터 7바이트**로 다룹니다.

### 바이트 인덱스

| Index | 이름 | 설명 |
|:-----:|:----:|------|
| 0 | Message ID | 프레임 식별자 |
| 1 | Data-0 | 페이로드 |
| 2 | Data-1 | 페이로드 |
| 3 | Data-2 | 페이로드 |
| 4 | Data-3 | 페이로드 |
| 5 | Data-4 | 페이로드 |
| 6 | Data-5 | 페이로드 |
| 7 | Data-6 | 페이로드 |

일부 필드는 1바이트를 **니블(4bit)** 단위로 나눕니다.

### 비트 순서

비트 **7 = MSB**, 비트 **0 = LSB**.

예: 값 `0xA1`

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 1 | 0 | 1 | 0 | 0 | 0 | 0 | 1 |

- 비트 7–4 → 상위 니블 (MSB nibble) = `0xA`
- 비트 3–0 → 하위 니블 (LSB nibble) = `0x1`

---

## Message ID 요약

| ID | 상태 | 내용 |
|:--:|:----:|------|
| `0x14` | 분석됨 | 좌측 핸들 버튼, Wonder Wheel In/Out·휠 값 |
| `0x20` | 분석됨 | 이그니션, 후륜 RPM → 차속 |
| `0x25` | 분석됨 | 적산거리 (km) |
| `0x2B` | 미분석 | — |
| `0x2E` | 미분석 | — |
| `0x61` | 분석됨 | 냉각수 온도 |
| `0xE2` | 분석됨 | 엔진 RPM, 기어, 배터리 전압 |

---

## ID `0x14` — 좌측 핸들 / Wonder Wheel

| Data | 상태 | 설명 |
|:----:|:----:|------|
| Data-0 | 미분석 | — |
| Data-1 | 분석됨 | Left Handle Controller 1 (버튼 비트맵) |
| Data-2 | 분석됨 | Left Handle Controller 2 (버튼 비트맵) |
| Data-3 | 분석됨 | Wonder Wheel In / Out |
| Data-4 | 미분석 | — |
| Data-5 | 분석됨 | Wonder Wheel 회전 값 |
| Data-6 | 미분석 | — |

### Data-1 — Left Handle Controller 1

차종: S1000XR 등

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Horn | 방향지시등 취소 | 방향지시등 오른쪽 | 방향지시등 왼쪽 | 크루즈 | 경보 | ABS | ESA |

### Data-2 — Left Handle Controller 2

차종: S1000XR 등

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| ? | ? | Trip | Info | 크루즈 Set | 크루즈 Reset | 하이빔 | ? |

### Data-3 — Wonder Wheel In / Out

차종: S1000XR 등

| 값 | 의미 |
|:--:|------|
| `0xFD` | Out |
| `0xFE` | In |

### Data-5 — Wonder Wheel 회전 값

차종: S1000XR 등

- 범위: `0x00` – `0xFF`
- 휠 회전에 따라 값이 변합니다.

---

## ID `0x20` — 이그니션 / 후륜 RPM·속도

| Data | 상태 | 설명 |
|:----:|:----:|------|
| Data-0 | 미분석 | — |
| Data-1 | 분석됨 | 이그니션 (`0x7F` = ON) |
| Data-2 | 분석됨 | 후륜 RPM 하위 8비트 |
| Data-3 | 분석됨 | 후륜 RPM 상위 4비트 (니블) |
| Data-4…6 | 미분석 | — |

### Data-1 — 이그니션

| 값 | 의미 |
|:--:|------|
| `0x7F` | Ignition ON |

### Data-2 / Data-3 — 후륜 RPM · 차속

```text
rear_wheel_rpm = ((Data-3 & 0xF0) << 8) + (Data-2 & 0xFF)
speed_kmh      ≈ rear_wheel_rpm × 0.121
```

**타이어 가정 (후륜 190/55ZR17)**

| 항목 | 값 |
|------|-----|
| 유효 지름 | 642 mm |
| 1회전 이동 | 2017 mm |

```text
speed_kmh = (rear_wheel_rpm × 60 × 2017) / 1000 / 1000
```

> 상위 4비트가 Data-3의 어느 니블인지(`0xF0` vs `0x0F`)는 캡처로 재검증하는 것을 권장합니다. 펌웨어 구현과 식이 다를 수 있습니다.

---

## ID `0x25` — 적산거리

```text
odometer_km = (Data-2 × 65536) + (Data-1 × 256) + Data-0
```

즉 `(Data-2 << 16) | (Data-1 << 8) | Data-0` 과 동일합니다.

> 바이트 인덱스는 Message ID를 제외한 **Data-n** 기준입니다. 버퍼 배열로 보면 `buf[1]…buf[3]`에 해당합니다.

---

## ID `0x2B` / `0x2E`

| ID | 상태 |
|:--:|:----:|
| `0x2B` | 미분석 |
| `0x2E` | 미분석 |

---

## ID `0x61` — 냉각수 온도

```text
coolant_celsius = Data-2 − 40
```

---

## ID `0xE2` — RPM / 기어 / 전압

| Data | 상태 | 설명 |
|:----:|:----:|------|
| Data-0 | 분석됨 | RPM/5 하위 8비트 |
| Data-1 | 분석됨 | 상위 니블 = 기어, 하위 니블 = RPM/5 상위 4비트 |
| Data-2…4 | 미분석 | — |
| Data-5 | 분석됨 | 배터리 전압 |
| Data-6 | 미분석 | — |

### Data-0 / Data-1 — 엔진 RPM

```text
rpm = (((Data-1 & 0xF0) << 8) + (Data-0 & 0xFF)) × 5
```

> Data-1의 **상위 니블은 기어**와 겹칩니다. 실제 펌웨어는 하위 니블(`Data-1 & 0x0F`)을 RPM 상위로 쓰는 경우가 있으므로, 배포 코드와 반드시 대조하세요.

### Data-1 — 기어 (상위 니블)

| 값 | 의미 |
|:--:|------|
| `0x10` | 1단 |
| `0x20` | N |
| `0x40` | 2단 |
| `0x70` | 3단 |
| `0x80` | 4단 |
| `0xB0` | 5단 |
| `0xD0` | 6단 |
| `0xF0` | 단수 변경 중 (사이) |

### Data-5 — 배터리 전압

```text
battery_v = (Data-5 & 0xFF) × 0.1
```

---

## 기여 안내

- 캡처 로그, 차종·연식, Message ID, Data 인덱스, 비트/마스크, 해석을 함께 적어 주세요.
- 미분석 필드(`?`)는 추측으로 채우지 말고, 확인된 내용만 추가해 주세요.
