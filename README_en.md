# Yutaki M-series H-Link Bus Monitoring with ESP8266 and Home Assistant

**Author: Jussi**  
**GitHub user: [jkirjo](https://github.com/jkirjo)**  
**Heat pump model: Hitachi Yutaki M RHUE5AVN**

This project documents how to monitor data from the H-Link bus of Hitachi Yutaki M-series heat pumps using an ESP8266 microcontroller and integrate it with Home Assistant. The approach has been field-tested and provides reliable access to internal system data such as temperatures, pressures, and operational status — without any official protocol documentation.

## 🧠 Motivation and Goal

The H-Link protocol used by Hitachi is undocumented, and no official API is available. This project aims to provide a practical and tested method to:

- Passively listen to the bus without interfering with system operation
- Capture and validate 9600N1 UART-formatted packets
- Decode critical byte values from the stream
- Present these readings as readable sensors in Home Assistant

⚠️ This project offers **monitoring only** — no control or modification of system behavior.

## 📡 Bus Characteristics and Tap Point

The H-Link bus operates at **9600 baud, 8N1 serial format**, modulated over a 9.6 kHz square wave. Clean UART-compatible signal is available after the system controller’s RX interface, specifically from one leg of the **information LED (post-filter)** — this tap point delivers clean, demodulated data.

An **ESP8266** was used in this setup, configured as a read-only listener. The system must include a **system controller card**, otherwise H-Link activity is not present on the bus.

## 🧰 Wiring

- ESP8266 RX → tapped LED leg (clean UART signal)
- TX not connected (left floating)
- GND shared with the system controller

## 📦 Packet Format

Two types of packets circulate, but only the **61-byte packet beginning with 3F:01:09** is required.

| Byte | Meaning                    | Format           |
|------|-----------------------------|------------------|
| 22   | Heating setpoint            | °C               |
| 34   | System state / error        | Hex → Text       |
| 38   | Circuit water in            | °C               |
| 39   | Circuit water out           | °C               |
| 40   | Outdoor temperature         | °C               |
| 49   | Compressor frequency        | Hz               |
| 52   | Suction pressure            | Bar (value /10)  |
| 53   | Discharge pressure          | Bar (value /5)   |
| 55   | Discharge gas temperature   | °C               |
| 56   | Suction gas temperature     | °C               |
| 57+58| EEV (expansion valve) steps | 16-bit value     |
| 59   | Liquid line temperature     | °C               |

## 🔧 ESPHome YAML Configuration

🛠️ The complete YAML configuration is provided in [`esp8266_yutaki_uart.yaml`](esp8266_yutaki_uart.yaml), defining:

- UART config for 9600N1
- Template sensors for temperatures, pressures, state
- Text decoding for compressor/system state (e.g. `0x80 = Running`, `0xA2 = Defrost`)
- Simple packet validation and sensor throttling

## ⚠️ Limitations

- Works only with **M-series** units (e.g. RHUE5AVN)
- **No control functions** — read-only monitoring
- Requires a **system controller card** to activate H-Link traffic. Could work wihtout also. If motherboard lowest yellow led blinking slowly, data is delivered to bus. That led is place to connect esp rx.
- All byte offsets and values have been reverse-engineered from a live system (empirical, not official)

---

> This is a working, field-tested H-Link readout method shared to benefit others who want more insight into their Yutaki system.  
> Built and shared by **Jussi** — feedback, questions and improvements welcome!
