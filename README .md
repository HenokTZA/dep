# Modbus RTU‑Wi‑Fi Bridge with Two ESP32 Boards

A lightweight, open‑source solution for replacing a wired RS‑485 link with a wireless TCP hop. Two ESP32 boards act as a bidirectional Modbus RTU‑TCP bridge, letting any Modbus master talk to any Modbus slave over Wi‑Fi without touching higher‑level protocols.

---

## Table of Contents

1. [Project Goals](#project-goals)
2. [System Overview](#system-overview)
3. [Bill of Materials](#bill-of-materials)
4. [Network & Wiring](#network--wiring)
5. [Firmware Layout](#firmware-layout)
6. [Build & Flash Guide](#build--flash-guide)
7. [Quick‑Start Simulation](#quick‑start-simulation)
8. [Configuration](#configuration)
9. [Troubleshooting](#troubleshooting)
10. [Roadmap](#roadmap)
11. [License & Credits](#license--credits)

---

## Project Goals

- **Cable replacement**: Replace a point‑to‑point RS‑485 line between an inverter (master) and a smart meter (slave).
- **Zero protocol changes**: Modbus frames stay untouched. Timing and exception codes are preserved.
- **Low cost**: Two ESP32 DevKit boards plus MAX485 units.
- **Ready for field units**: Only SSID, password, and static IPs need editing.

## System Overview

```
┌────────┐   RS‑485    ┌────────┐   Wi‑Fi TCP   ┌────────┐   RS‑485    ┌────────┐
│MASTER  │◄──────────►│ESP‑A   │◄─────────────►│ESP‑B   │◄──────────►│SLAVE   │
│Inverter│   (9600)    │RTU→TCP │ 192.168.10.46 │TCP→RTU │   (9600)    │Meter   │
└────────┘             └────────┘               └────────┘             └────────┘
```

- **ESP‑A** sits near the inverter. It listens as Modbus RTU *slave* and forwards raw frames to ESP‑B using Modbus TCP *client* mode.
- **ESP‑B** creates a hidden Wi‑Fi AP and runs a Modbus TCP *server*. It re‑injects incoming frames on its UART as Modbus RTU *master* and ships the reply back.
- Roundtrip adds less than 2 ms latency at 9600 bps.

## Bill of Materials

| Qty | Item                                          | Notes               |
| --- | --------------------------------------------- | ------------------- |
| 2   | ESP32 DevKit‑V1 (or similar)                  | 3.3 V logic         |
| 2   | MAX485 (or MAX3485) RS‑485 transceiver module | DE/RE tied together |
| 2   | 4‑pin screw terminal + 120 Ω terminator       | For A/B pair        |
| 2   | 5 V USB wall adapter                          | ≥500 mA             |
| –   | Dupont wires                                  | Female‑female       |

## Network & Wiring

### Wi‑Fi Settings

- **SSID**: `ModbusLink`
- **Password**: `Bridge123`
- **Channel**: 6 (change if noisy)
- **IP plan**:
  - ESP‑B AP: 192.168.10.1/24
  - ESP‑A STA: 192.168.10.46/24 (static)

### Pinout (both boards)

| ESP32 Pin | RS‑485 | Purpose                |
| --------- | ------ | ---------------------- |
| 17        | RO     | UART RX                |
| 18        | DI     | UART TX                |
| 5         | DE+RE  | Direction ctl          |
| GND       | GND    | Common                 |
| 5 V       | VCC    | Power module if needed |

**Tip**: Tie DE and RE together; the Modbus library toggles the pair.

## Firmware Layout

```
firmware/
  espA_rtutcp_bridge.ino   # Inverter side
  espB_tcprtu_bridge.ino   # Smart‑meter side
```

Both sketches are adapted from the official [modbus‑esp8266](https://github.com/emelianov/modbus-esp8266) examples and stay under the upstream BSD license.

Key differences:

- Fixed Wi‑Fi credentials and static IPs
- Raw callbacks (`onRaw`) used so no frame parsing overhead
- Simple mapping table lets ESP‑A direct multiple slave IDs to different IPs if the project grows

## Build & Flash Guide

1. **Arduino IDE** 2.x or **PlatformIO**.
2. Install **ESP32 Core** (latest stable) and `` by Alexander Emelianov.
3. Select **Board**: *ESP32 Dev Module* (or matching one).
4. Edit the constants at the top if your pins, SSID, or baud differ.
5. Connect USB and flash each sketch:
   - `espA_rtutcp_bridge.ino` → ESP‑A
   - `espB_tcprtu_bridge.ino` → ESP‑B
6. Open serial monitor at 115200 baud to watch logs.

## Quick‑Start Simulation

1. **PC‑1** (Inverter side)
   - USB‑RS‑485 adapter on COM‑X.
   - [Modbus Poll](https://www.modbustools.com/modbus_poll.html) → Mode: *Master*, 9600 8N1.
   - Station ID = `1` to match mapping.
2. **PC‑2** (Meter side)
   - USB‑RS‑485 on COM‑Y.
   - Run the included `python_slave.py` (any pymodbus 3.x) as ID 1.
3. Requests from PC‑1 should read/write holding registers exposed by the Python slave with end‑to‑end latency well below one second.

## Configuration

| Constant                      | File  | Description                     |
| ----------------------------- | ----- | ------------------------------- |
| `ssid`, `password`            | both  | Wi‑Fi credentials               |
| `ip`, `gw`, `nm`              | ESP‑A | Static station address          |
| `PIN_DE_RE`                   | both  | Direction control pin           |
| `mapping[]`                   | ESP‑A | Slave‑to‑IP routing table       |
| UART `Serial2`/`Serial1` pins | both  | Swap if you use alternate GPIOs |

## Troubleshooting

- `` on ESP‑B → Increase timeout of the master or raise ESP‑B baud rate.
- **No reply**: Confirm that DE/RE toggles; LED on MAX485 should switch TX/RX.
- **Wi‑Fi drops**: Try static channel or move AP away from congested 2.4 GHz traffic.
- **Frame errors**: Check that grounds of RS‑485 lines are shared and termination resistors are present.

## Roadmap

- Web UI for live stats.
- Over‑the‑air firmware update.
- TLS tunnel for wider area links.

## License & Credits

Code inherits the **BSD‑New** license from the [modbus‑esp8266](https://github.com/emelianov/modbus-esp8266) project. See `LICENSE.txt` for details.

Inspired by Alexander Emelianov’s reference bridges. Hardware photos and extended docs will be added as field units arrive.

