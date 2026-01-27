# Letter of Intent: Next Generation Busware Architecture (USB-C SBU & Gateway)

**Date:** January 27, 2026  
**To:** Engineering & Product Management, BUSWARE  
**Subject:** Proposal for the "SBU-Native" Stick Standard and the "Ultimate Busware Gateway" Architecture

---

## 1. Executive Summary

This document outlines the strategic technical roadmap for the next generation of busware IoT devices. It proposes two fundamental shifts in hardware design:
1.  **The "SBU-Native" Standard:** A unified USB-C pinout leveraging Sideband Use (SBU) pins for raw UART communication, enabling low-latency integration without USB protocol overhead.
2.  **The "Ultimate Gateway":** A highly integrated host controller based on the ESP32-C6, featuring a dual-path architecture (SPI-UART bridging + USB Hub) to support legacy PC operation and modern embedded bridging simultaneously.

---

## 2. The New Stick Standard: "USB-C SBU-Native"

### 2.1 Concept
Current USB-to-Serial bridging relies heavily on USB CDC (Communication Device Class), which introduces significant software overhead and latency on embedded hosts. The new design utilizes the **USB-C Sideband Use (SBU)** pins to transmit raw UART (TTL) signals.

### 2.2 Pinout Specification
Future busware sticks (CUL, Zigbee, Z-Wave) shall implement the following USB-C mapping:

| Pin | Name | Usage | Electrical Spec |
| :--- | :--- | :--- | :--- |
| **A6/B6** | **D+** | USB 2.0 Data | Standard USB High-Speed (Legacy PC Support) |
| **A7/B7** | **D-** | USB 2.0 Data | Standard USB High-Speed (Legacy PC Support) |
| **A8** | **SBU1** | **UART TX** | Stick transmits, Host receives. **Must have 1kΩ Series Resistor.** |
| **B8** | **SBU2** | **UART RX** | Stick receives, Host transmits. **Must have 1kΩ Series Resistor.** |
| **A4/B4** | **VBUS** | Power | 5V DC |
| **A1/B1** | **GND** | Ground | Common Ground |

### 2.3 Safety & Compatibility
* **PC Compatibility:** On standard USB-A to USB-C cables, SBU pins are floating. The stick behaves as a standard USB CDC device.
* **Protection:** The 1kΩ series resistors on SBU lines protect the MCU against accidental connection to non-compliant ports (e.g., active audio adapters).

---

## 3. The Host Architecture: "Ultimate Busware Gateway"

To maximize the potential of the SBU-Native sticks, we propose a dedicated 4-Port Gateway.

### 3.1 Core Components
* **Main Processor:** **Espressif ESP32-C6**
    * *Role:* Application Host, Zigbee/Matter/Thread Coordinator (Native).
    * *OS:* ESP-IDF (FreeRTOS) / Tasmota.
* **Network Interface:** **WIZnet W5500**
    * *Connection:* SPI.
    * *Role:* Reliable, hardwired 10/100 Ethernet connectivity.
* **UART Bridge (Internal):** **WK2124** (SPI-to-Quad-UART)
    * *Connection:* SPI (Shared bus with W5500).
    * *Role:* Provides 4x high-speed hardware UARTs connected to the SBU pins of the 4 slots.
* **USB Hub (External):** **Terminus FE1.1s** (or GL850G)
    * *Role:* Provides 4x USB Downstream ports connected to the D+/D- pins of the 4 slots.

### 3.2 The "Dual-Path" Topology
This architecture allows two simultaneous, non-blocking access paths to the attached sticks:

#### Path A: The Embedded Operation Path (via SBU)
> **Flow:** Stick MCU (UART) -> SBU Pins -> WK2124 -> SPI -> ESP32-C6

* **Usage:** Normal Gateway operation (Home Automation).
* **Advantage:** Zero USB overhead on the ESP32. Extremely low latency. Large FIFOs (256 Byte) on the WK2124 prevent packet loss during network congestion.

#### Path B: The Maintenance Path (via USB Hub)
> **Flow:** Stick MCU (USB) -> D+/D- Pins -> FE1.1s Hub -> USB-C Uplink (PC/Laptop)

* **Usage:** Firmware updates (DFU), advanced debugging, legacy PC connectivity.
* **Advantage:** The Gateway acts as a transparent USB Hub. A user can plug the Gateway into a PC and flash all 4 sticks individually without the ESP32 needing to pass-through data.

---

## 4. Technical Feasibility & BOM Estimate

The proposed design is cost-effective and relies on industry-standard components available in high volume.

| Component | Function | Estimated Cost (High Vol.) |
| :--- | :--- | :--- |
| **ESP32-C6-WROOM-1** | Host MCU + Matter Radio | ~.00 |
| **WIZnet W5500** | Ethernet Controller | ~.00 |
| **WK2124** | SPI-to-4xUART Bridge | ~.50 |
| **FE1.1s** | 4-Port USB 2.0 Hub | ~-bash.40 |
| **Misc** | PCB, LDOs, Connectors | ~.00 |

**Power Requirement:** Due to the potential load of 4 transmitting sticks + Ethernet, an external 5V/3A DC barrel jack or dedicated USB-C Power Input is mandatory.

---

## 5. Conclusion

By adopting the **USB-C SBU Standard**, busware secures a competitive edge in latency and integration ease. The **Ultimate Gateway** leverages this standard to offer a device that is simultaneously a cutting-edge Matter/Thread Border Router (via ESP32-C6) and a robust bridge for legacy 433/868MHz protocols, with a fail-safe maintenance mode via standard USB.

**Recommendation:** Proceed to PCB prototyping phase for the "SBU-Native" Stick breakout and the Gateway mainboard.
