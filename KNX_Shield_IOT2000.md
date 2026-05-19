# KNX Shield on the Siemens SIMATIC IOT2000 Platform

## 1. Platform Overview

The SIMATIC IOT2040 is an industrial IoT gateway based on the **Intel Quark X1020** processor running **Yocto Linux**. It exposes an Arduino Uno R3-compatible shield interface internally. Because it runs Linux — not a bare-metal AVR — shield interaction goes through Intel's **MRAA library** from userspace, not through Arduino core libraries.

This has direct consequences for KNX shield selection: any shield that requires AVR-specific timing, hardware interrupts at microsecond resolution, or Timer2 will not function. Only shields that offload all KNX timing to an on-board transceiver and communicate with the host via UART are compatible.

---

## 2. Shield Selection

### 2.1 Transceiver Variants

The ON Semiconductor **NCN5100ASGEVB** evaluation board (EVBUM2715/D) is available in three transceiver variants:

| Transceiver | OSI Layer | Host handles timing? | Compatible with IOT2040? |
|---|---|---|---|
| NCN5110 | Physical only | Yes — all timing | ❌ No |
| NCN5121 | Physical + MAC | No — transceiver handles it | ✅ UART version |
| NCN5130 | Physical + MAC | No — transceiver handles it | ✅ UART version |

**The NCN5110 is incompatible.** It implements only the physical layer; the host microcontroller must generate the 35 µs active pulse and handle collision avoidance. Linux scheduler jitter makes this impossible without a real-time kernel.

**The NCN5121 and NCN5130 UART versions are compatible.** Both implement the MAC layer internally, so the host only exchanges complete KNX frames over UART.

### 2.2 NCN5121 vs NCN5130

| Parameter | NCN5121 | NCN5130 |
|---|---|---|
| Max bus current | 24 mA | 40 mA |
| Fan-in modes | 10 mA / 20 mA fixed | 10 / 20 mA fixed + 5–40 mA adjustable via R3 |
| TP-UART compatible | Yes | Yes |
| UART interface | Yes | Yes |
| SPI interface | Yes (avoid on IOT2040) | Yes (avoid on IOT2040) |

For a single KNX device with standard current draw, the **NCN5121 UART** is sufficient. Choose the **NCN5130 UART** if higher or adjustable bus current is required.

> **Note:** The SPI versions of both transceivers are not recommended on the IOT2040. SPI via the MRAA library on the Intel Quark is unreliable and not plug-and-play.

---

## 3. IOT2040 Jumper Configuration

Two motherboard jumpers must be set before installing the shield. Both are accessible after opening the left and right covers.

### 3.1 X18 — Shield IO Voltage

Located on the motherboard component side.

| Setting | Voltage | Use |
|---|---|---|
| Pins 1–2 | 5 V | For 5 V shields |
| Pins 2–3 | 3.3 V | **Required for NCN5121/5130** |

Set **X18 to pins 2–3 (3.3 V)**. The NCN5100 shield operates entirely at 3.3 V. Applying 5 V risks damaging the transceiver.

### 3.2 X82 — VIN Connection

| Setting | Effect |
|---|---|
| Pins 1–2 | IOT2040 supply (9–36 V DC) connected to Arduino VIN pin |
| Pins 2–3 | VIN disconnected from shield |

Set **X82 to pins 2–3**. The NCN5100 shield powers itself from the KNX bus via its onboard DC-DC converters. The IOT2040 uses its own 24 V DC industrial supply. Do not connect both supplies to VIN simultaneously.

### 3.3 Shield Supply Jumpers (on the NCN5100 shield itself)

The shield has two additional jumpers, J10 and J11, that control how it powers an attached Arduino board. Since the IOT2040 powers itself independently, both must be left **open (removed)**:

| Jumper | Function | Setting |
|---|---|---|
| J10 | Routes 9 V DC-DC2 output to Arduino VIN | **Open** |
| J11 | Routes 3.3 V DC-DC1 output to Arduino 3V3 | **Open** |

---

## 4. Physical Installation

> ⚠️ Disconnect the IOT2040 from power before installing any shield.

1. Open the right cover, then open the left cover.
2. Align the shield contact pins with the motherboard terminal strips (X10, X11, X12, X13, X15).
3. Press down firmly and evenly — do not rock the shield.
4. Verify no component on the shield touches any component on the motherboard.
5. Optionally secure the shield using the four PCB boreholes with **plastic** standoffs only — never metallic or conductive fasteners.

> **Approval note:** Installing an Arduino shield voids the IOT2040's factory certifications (CE, UL, UKCA). The customer assumes responsibility for any re-approval required.

---

## 5. KNX Bus Wiring

Connect the KNX twisted-pair bus to the shield's screw terminal (J5, Wago 243-211 compatible):

| Terminal | Signal |
|---|---|
| KNX+ | KNX bus positive |
| KNX− | KNX bus negative |

Polarity is not critical for the KNX bus itself, but follow your installation convention. Use the correct KNX power supply — a standard 24 V DC lab supply cannot drive the bus without an additional choke.

---

## 6. UART Communication Path

The shield UART connects to the IOT2040's shield interface as follows:

| Shield Pin | Arduino Header | IOT2040 Connector | Linux Device |
|---|---|---|---|
| SDI/RXD (shield RX) | D0 / RX | X11 pin 1 | `/dev/ttyS0` |
| SDO/TXD (shield TX) | D1 / TX | X11 pin 2 | `/dev/ttyS0` |

The IOT2040 exposes this as `/dev/ttyS0`. On the shield, the communication mode is selected by resistor population:

| Mode | Resistors populated |
|---|---|
| UART | R16, R17, R32 |
| SPI | R9, R11, R12, R13, R15, R25 |

Ensure only the **UART resistors** are populated.

### Baud Rate (Jumpers J1/J2 on shield)

| J2 | J1 | Parity | Baud rate |
|---|---|---|---|
| 0 | 0 | Even | 19 200 bps |
| 0 | 1 | Even | 38 400 bps |
| 1 | 0 | None | 19 200 bps |
| 1 | 1 | None | 38 400 bps |

The Tapko KAIstack demo software uses **19 200 bps with even parity** (J1=0, J2=0). Configure the IOT2040 serial port to match.

---

## 7. Software Access on the IOT2040

Access the KNX transceiver from the IOT2040 via `/dev/ttyS0` using any of the following approaches:

### 7.1 MRAA UART (C/C++ or Python)

```python
import mraa
uart = mraa.Uart(0)          # /dev/ttyS0
uart.setBaudRate(19200)
uart.setMode(8, mraa.UART_PARITY_EVEN, 1)
uart.setFlowcontrol(False, False)
```

### 7.2 Node-RED

Use the standard **Serial In / Serial Out** nodes pointing to `/dev/ttyS0` at 19 200 bps, even parity. Install `node-red-contrib-knx` for higher-level KNX group address handling.

### 7.3 knxd (Linux KNX daemon)

The NCN5121/5130 UART version is **TP-UART compatible**. Install `knxd` on the Yocto image and point it at `/dev/ttyS0`:

```bash
knxd --layer2=tpuart:/dev/ttyS0 --listen-tcp=6720
```

Node-RED can then connect to `knxd` over TCP, abstracting all frame-level handling.

---

## 8. Compatibility Summary

| Requirement | Status |
|---|---|
| Shield physically fits IOT2040 Arduino header | ✅ Arduino Uno R3 form factor |
| 3.3 V logic compatible | ✅ NCN5121/5130 operate at 3.3 V |
| No AVR-specific timing required | ✅ MAC layer in transceiver |
| UART accessible from Linux via MRAA | ✅ `/dev/ttyS0` |
| TP-UART compatible (knxd) | ✅ |
| Industrial temperature rating | ⚠️ Verify shield rating vs IOT2040 enclosure (+30 °C above ambient) |
| CE approval maintained | ❌ Voided by shield installation |

---

## 9. References

- Siemens SIMATIC IOT2020/IOT2040 Operating Instructions, 07/2022, A5E37656492-AC
- ON Semiconductor EVBUM2715/D — NCN5100 Arduino Shield Evaluation Board User's Manual, Rev. 2, October 2020
- knxd project: [https://github.com/knxd/knxd](https://github.com/knxd/knxd)
- KNX Association: [https://www.knx.org](https://www.knx.org)
