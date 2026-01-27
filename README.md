# bw-sbu-bridge

**The Ultimate ESP32-C6 IoT Gateway & USB-C SBU Breakout**


## 1. Overview

**bw-sbu-bridge** is a next-generation IoT Gateway architecture designed by **BUSWARE**. It bridges up to **4 external RF sticks** (CUL, Zigbee, Z-Wave, EnOcean) to Ethernet/WiFi/Matter, utilizing the **ESP32-C6** as the host processor.

Unlike traditional USB hosts, this project introduces the **"SBU-Native"** standard, leveraging the USB-C Sideband Use pins for raw UART communication. This eliminates USB protocol overhead on the ESP32 side while maintaining full USB compatibility for PC maintenance.

---

## 2. Key Features

* **Dual-Path Architecture:**
    * **Operation Path:** Raw UART via SBU pins $ightarrow$ SPI Bridge $ightarrow$ ESP32-C6 (Zero Latency).
    * **Maintenance Path:** Standard USB D+/D- $ightarrow$ USB Hub $ightarrow$ PC (Pass-through for firmware updates).
* **Matter & Thread Ready:** Powered by **ESP32-C6** (RISC-V) with native 802.15.4 radio.
* **High-Performance I/O:** Dedicated **WIZnet W5500** Ethernet and **WK2124** SPI-to-Quad-UART bridge.
* **Universal Compatibility:** Supports legacy 868MHz CUL sticks alongside modern Zigbee/Matter dongles.

---

## 3. System Architecture

The system is built around a split-bus topology:

```mermaid
graph TD
    subgraph "Host PC / Debugger"
        PC[Computer USB-C]
    end

    subgraph "bw-sbu-bridge PCB"
        Hub[USB Hub FE1.1s]
        MCU[ESP32-C6]
        Eth[W5500 Ethernet]
        UART_Bridge[WK2124 SPI-UART]
        
        % Connections
        PC == USB 2.0 ==> Hub
        Hub -. D+/D- Pass-through .-> Slot1[Stick Slot 1]
        Hub -. D+/D- Pass-through .-> Slot2[Stick Slot 2]
        Hub -. D+/D- Pass-through .-> Slot3[Stick Slot 3]
        Hub -. D+/D- Pass-through .-> Slot4[Stick Slot 4]

        MCU == SPI ==> Eth
        MCU == SPI ==> UART_Bridge
        
        UART_Bridge == UART/SBU ==> Slot1
        UART_Bridge == UART/SBU ==> Slot2
        UART_Bridge == UART/SBU ==> Slot3
        UART_Bridge == UART/SBU ==> Slot4
    end
```

---

## 4. The "SBU-Native" Pinout Standard

This project defines a custom usage of the USB-C Sideband pins to enable dual-mode operation.

| USB-C Pin | Name | Function (Stick Side) | Notes |
| :--- | :--- | :--- | :--- |
| **A6 / B6** | **D+** | USB Data + | Connects to USB Hub (PC Link) |
| **A7 / B7** | **D-** | USB Data - | Connects to USB Hub (PC Link) |
| **A8** | **SBU1** | **UART TX** | 1kΩ Series Resistor required |
| **B8** | **SBU2** | **UART RX** | 1kΩ Series Resistor required |
| **A4/B4/A9/B9** | **VBUS** | 5V Power | Bus Power |
| **GND** | **GND** | Ground | Common Ground |

> **Note:** The 1kΩ resistors on SBU lines are mandatory to protect the MCU if the stick is inserted into a non-compliant USB-C audio adapter.

---

## 5. Bill of Materials (BOM)

| Component | Description | Role |
| :--- | :--- | :--- |
| **ESP32-C6-WROOM-1** | RISC-V MCU | Application Host / Matter Coordinator |
| **WK2124** | SPI to 4x UART | High-speed Serial Bridge (256 Byte FIFO) |
| **W5500** | SPI Ethernet | Hardwired Network Interface |
| **FE1.1s** | USB 2.0 Hub | 4-Port USB Pass-through Controller |
| **USB-C Receptacle** | 16-Pin / 24-Pin | High-quality connectors for sticks |

## 6. Software Support

* **ESP-IDF:** Native support for C6, W5500, and WK2124 drivers available.
* **Tasmota:** Potential for a custom build supporting the WK2124 driver for easy scripting.
* **ESPHome:** Custom component required for quad-UART bridging over SPI.

## 7. License

MIT License - Copyright (c) 2026 BUSWARE
