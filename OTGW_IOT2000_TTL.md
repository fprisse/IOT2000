# OpenTherm Gateway (OTGW) on the Siemens SIMATIC IOT2000 via Direct TTL UART

## 1. Overview

The **Schelte Bron OpenTherm Gateway (OTGW)** is a PIC-based hardware gateway that sits in-line between an OpenTherm thermostat and an OpenTherm boiler. All time-critical OpenTherm protocol work (Manchester encoding/decoding at 1 kbps, collision avoidance, message forwarding) is handled entirely by the **PIC16F1847 microcontroller** using its dedicated hardware peripherals — comparators, voltage reference, and Timer2. The IOT2040 has no timing obligations whatsoever; it only reads and writes plain ASCII text over a serial connection.

This document describes connecting the OTGW to the IOT2040 via a **direct TTL UART link**, bypassing the OTGW's onboard MAX232 RS232 level converter. This eliminates the need for the nodo-shop Serial board add-on.

```
Physical thermostat
      │  OpenTherm 2-wire
[OTGW slave interface]
      │
  [PIC16F1847]  ←── TTL UART 5 V ──→  [IOT2040 Arduino shield UART]
      │                                    X11 pins 1 & 2, /dev/ttyS0
[OTGW master interface]
      │  OpenTherm 2-wire
      Boiler
```

---

## 2. OTGW Serial Interface Architecture

The OTGW PCB contains the following serial-related components:

| Reference | Component | Function |
|---|---|---|
| IC1 | PIC16F1847 | Central controller, runs at 5 V |
| IC2 | MAX232 | Converts PIC TTL ↔ RS232 ±12 V |
| SV1 | 10-pin header (marked RS232) | Exposes MAX232 and TTL signals |

The MAX232 is an **optional** component. When it is not populated, the PIC's TTL-level TX and RX lines are still accessible at specific pins of the SV1/IC2 socket. These are 5 V TTL signals at 9600 baud, 8N1.

### TTL Signal Points on SV1 / IC2 Socket

Access these pins at the MAX232 socket (whether the chip is fitted or not — the socket pins are always accessible):

| MAX232 pin | Signal | Direction (relative to PIC) |
|---|---|---|
| Pin 11 | PIC RX (TTL input) | Input to OTGW |
| Pin 12 | PIC TX (TTL output) | Output from OTGW |
| Pin 15 | GND | Ground reference |
| Pin 16 | VCC (+5 V) | 5 V supply (do not power IOT2040 from this) |

> Connect pin 12 (PIC TX) to the IOT2040 RX line, and pin 11 (PIC RX) to the IOT2040 TX line. Always connect GND between the two boards.

---

## 3. IOT2040 Jumper Configuration

### 3.1 X18 — Shield IO Voltage (Critical)

This jumper sets the logic voltage on the Arduino shield interface pins, including the UART on D0/D1.

| Setting | Voltage |
|---|---|
| Pins 1–2 | **5 V** ← required for OTGW TTL connection |
| Pins 2–3 | 3.3 V (default) |

Set **X18 to pins 1–2 (5 V)**. This raises the shield interface to 5 V TTL, matching the OTGW PIC output voltage. Without this, the IOT2040 would output 3.3 V logic on its TX line, which the PIC may or may not reliably interpret, and the PIC's 5 V TX output could damage the IOT2040's 3.3 V input circuitry.

> ⚠️ Setting X18 to 5 V affects all shield interface pins. Do not simultaneously connect any 3.3 V-only component to the shield headers.

### 3.2 X82 — VIN Connection

| Setting | Effect |
|---|---|
| Pins 1–2 | IOT2040 supply (9–36 V DC) connected to Arduino VIN pin |
| Pins 2–3 | VIN disconnected from shield |

Set **X82 to pins 2–3**. The OTGW powers itself from its own AC transformer — the IOT2040 VIN supply must not be connected to it.

---

## 4. Obtaining 24 V from the IOT2040 (If Needed)

The OTGW is normally self-powered from its mains AC transformer (the primary power input). There is **no need** to supply the OTGW from the IOT2040.

However, if a 24 V supply is needed for another purpose (for example, powering additional field devices from the same cabinet), the IOT2040's supply input accepts 9–36 V DC. The IOT2040 itself does not generate or output 24 V externally.

If the IOT2040's VIN needs to be exposed to the shield:

| X82 Setting | Effect |
|---|---|
| Pins 1–2 | VIN rail (the IOT2040's DC supply, whatever voltage was connected to X80) appears on the Arduino VIN pin |

> This only routes the IOT2040's supply voltage to the Arduino VIN header pin — it does not regulate, limit, or convert it. If the IOT2040 is supplied at 24 V DC, the VIN pin will carry 24 V. Do not connect this to any component not rated for the supply voltage present on X80.

For the OTGW connection specifically: **leave X82 on pins 2–3. The OTGW does not need power from the IOT2040.**

---

## 5. Wiring

### 5.1 Connection Table

| IOT2040 | IOT2040 Reference | Wire to | OTGW Reference |
|---|---|---|---|
| D0 / RX | X11 pin 1 | ────── | Pin 12 (PIC TX) on IC2 socket |
| D1 / TX | X11 pin 2 | ────── | Pin 11 (PIC RX) on IC2 socket |
| GND | X10 pin 7 | ────── | Pin 15 (GND) on IC2 socket |

Do **not** connect the IOT2040's 5 V or 3.3 V pins to OTGW VCC (pin 16). The OTGW has its own power supply.

### 5.2 X11 Header Pinout (IOT2040 Arduino Shield Interface)

```
X11 (8-pin header, Arduino digital pins 0–7)

Pin 1  →  D0 / RX   ← connect to OTGW pin 12 (PIC TX)
Pin 2  →  D1 / TX   ← connect to OTGW pin 11 (PIC RX)
Pin 3  →  D2
Pin 4  →  D3 (PWM)
Pin 5  →  D4
Pin 6  →  D5 (PWM)
Pin 7  →  D6 (PWM)
Pin 8  →  D7
```

GND is available on X10 pin 7.

### 5.3 OTGW IC2 Socket Pin Numbering

The MAX232 is a 16-pin DIP. Pin 1 is marked with a notch or dot on the socket. Count from pin 1:

```
        ┌──────────────┐
   1 ── │  C1+         │ ── 16  VCC (+5 V)
   2 ── │  V+          │ ── 15  GND
   3 ── │  C1-         │ ── 14  DOUT1 (RS232 TX out)
   4 ── │  C2+         │ ── 13  RIN1  (RS232 RX in)
   5 ── │  C2-         │ ── 12  ROUT1 (TTL RX out → PIC TX) ← connect to IOT2040 D0/RX
   6 ── │  V-          │ ── 11  DIN1  (TTL TX in ← PIC RX)  ← connect to IOT2040 D1/TX
   7 ── │  DOUT2       │ ── 10  DIN2
   8 ── │  RIN2        │ ──  9  ROUT2
        └──────────────┘
```

> If IC2 (MAX232) is not populated, access pins 11 and 12 via the empty socket. If it is populated, the signals are still present at those pins — the MAX232 does not block TTL access.

---

## 6. Serial Parameters

| Parameter | Value |
|---|---|
| Baud rate | 9600 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |
| Linux device | `/dev/ttyS0` |

---

## 7. OTGW Message Protocol

The OTGW communicates in plain ASCII lines terminated with `\r\n`. No Modbus, no binary framing.

### 7.1 Received Messages (OTGW → IOT2040)

| Prefix | Meaning |
|---|---|
| `T` | Request received from thermostat (8 hex digits) |
| `B` | Response received from boiler (8 hex digits) |
| `R` | Request forwarded to boiler (gateway modified it) |
| `A` | Answer forwarded to thermostat (gateway modified it) |
| `E` | Error — parity or stop-bit error detected |

Example output:
```
T00000100
B40000100
T00010000
B40010064
```

### 7.2 Commands (IOT2040 → OTGW)

| Command | Example | Effect |
|---|---|---|
| `TT=xx.x` | `TT=20.5` | Override room temperature setpoint |
| `CS=xx.x` | `CS=65.0` | Override boiler water setpoint |
| `CH=0/1` | `CH=1` | Enable/disable central heating |
| `HW=0/1` | `HW=1` | Enable/disable domestic hot water |
| `OT=xx.x` | `OT=5.0` | Report outside temperature to thermostat |
| `SC=D/HH:MM` | `SC=3/07:30` | Set thermostat clock |
| `PS=0/1` | `PS=1` | Polling mode (suppress unsolicited output) |
| `PR=x` | `PR=A` | Request specific data from gateway |

---

## 8. Software Access on the IOT2040

### 8.1 Test from Command Line

```bash
# Open serial port at 9600 baud
stty -F /dev/ttyS0 9600 cs8 -cstopb -parenb raw

# Read incoming OTGW messages
cat /dev/ttyS0

# Send a setpoint override
echo "TT=20.5" > /dev/ttyS0
```

### 8.2 Node-RED

1. Install `node-red-contrib-opentherm-gateway` or use standard **Serial In/Out** nodes.
2. Configure Serial In node: `/dev/ttyS0`, 9600 baud, 8N1, split on `\n`.
3. Parse the prefix letter in a Function node to route `T`, `B`, `R`, `A`, `E` messages.
4. Use a Serial Out node to send commands.

### 8.3 Python (pyserial)

```python
import serial
import time

port = serial.Serial(
    port='/dev/ttyS0',
    baudrate=9600,
    bytesize=serial.EIGHTBITS,
    parity=serial.PARITY_NONE,
    stopbits=serial.STOPBITS_ONE,
    timeout=2
)

# Send a command
port.write(b'TT=20.5\r\n')

# Read messages
while True:
    line = port.readline().decode('ascii').strip()
    if line:
        prefix = line[0]          # T, B, R, A, or E
        data   = line[1:]         # 8 hex characters
        print(f"[{prefix}] {data}")
```

### 8.4 MRAA UART

```python
import mraa

uart = mraa.Uart(0)               # /dev/ttyS0
uart.setBaudRate(9600)
uart.setMode(8, mraa.UART_PARITY_NONE, 1)
uart.setFlowcontrol(False, False)
uart.write(b'TT=20.5\r\n')
```

---

## 9. Configuration Checklist

| Step | Action | ✓ |
|---|---|---|
| 1 | IOT2040 powered off | |
| 2 | X18 set to pins 1–2 (5 V) | |
| 3 | X82 set to pins 2–3 (VIN disconnected) | |
| 4 | IOT2040 D0/RX (X11 pin 1) wired to OTGW IC2 pin 12 | |
| 5 | IOT2040 D1/TX (X11 pin 2) wired to OTGW IC2 pin 11 | |
| 6 | GND (X10 pin 7) wired to OTGW IC2 pin 15 | |
| 7 | OTGW powered from its own AC transformer | |
| 8 | OTGW thermostat connector wired to physical thermostat | |
| 9 | OTGW boiler connector wired to boiler OpenTherm terminals | |
| 10 | IOT2040 powered on | |
| 11 | `cat /dev/ttyS0` shows T/B messages after thermostat cycles | |

---

## 10. References

- Schelte Bron, OpenTherm Gateway project: [https://otgw.tclcode.com](https://otgw.tclcode.com)
- OTGW schematic and part list: [https://otgw.tclcode.com/schematic.html](https://otgw.tclcode.com/schematic.html)
- OTGW firmware and serial commands: [https://otgw.tclcode.com/firmware.html](https://otgw.tclcode.com/firmware.html)
- Siemens SIMATIC IOT2020/IOT2040 Operating Instructions, 07/2022, A5E37656492-AC — section 7.5.1 (Motherboard, jumper table)
- nodo-shop OTGW kit: [https://www.nodo-shop.nl](https://www.nodo-shop.nl)
