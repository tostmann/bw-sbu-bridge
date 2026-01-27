# bw-sbu-bridge

**The Ultimate ESP32-C6 IoT Gateway & USB-C SBU Breakout**

![Status](https://img.shields.io/badge/Status-Hardware_Design-blue) ![License](https://img.shields.io/badge/License-MIT-green)

## 1. Overview

**bw-sbu-bridge** is a next-generation IoT Gateway architecture designed by **BUSWARE**. It bridges up to **4 external RF sticks** (CUL, Zigbee, Z-Wave, EnOcean) to Ethernet/WiFi/Matter, utilizing the **ESP32-C6** as the host processor.

Unlike traditional USB hosts, this project introduces the **"SBU-Native"** standard, leveraging the USB-C Sideband Use pins for raw UART communication. This eliminates USB protocol overhead on the ESP32 side while maintaining full USB compatibility for PC maintenance.

---

## 2. Key Features

* **Dual-Path Architecture:**
    * **Operation Path:** Raw UART via SBU pins -> SPI Bridge -> ESP32-C6 (Zero Latency).
    * **Maintenance Path:** Standard USB D+/D- -> USB Hub -> PC (Access to all Sticks AND the Host C6).
* **Triple-Radio Host:**
    * **Matter / Thread:** Native 802.15.4 support via ESP32-C6.
    * **Zigbee 3.0:** Can act as Coordinator or Router.
    * **WiFi 6 (802.11ax):** Low-power, high-efficiency wireless uplink.
* **Professional Connectivity:**
    * Dedicated **WIZnet W5500** Ethernet Controller.
    * **Power over Ethernet (PoE)** support (802.3af) for single-cable installation.
* **High-Performance I/O:** **WK2124** SPI-to-Quad-UART bridge for reliable serial streams.

---

## 3. System Architecture

The system is built around a split-bus topology with 5 USB downstream ports (4x Sticks + 1x Host MCU) and a dedicated wireless frontend:

```mermaid
graph TD
    subgraph External [External Connections]
        PC["Computer USB-C"]
        RJ45["LAN Switch / PoE"]
        HomeNet[("Matter Fabric / WiFi / Zigbee Mesh")]
    end

    subgraph PCB [bw-sbu-bridge PCB]
        Hub["USB Hub 5-Port"]
        MCU["ESP32-C6"]
        Eth["W5500 Ethernet"]
        UART_Bridge["WK2124 SPI-UART"]
        PoE_Power["PoE Power Extraction"]
        
        % Wireless Section
        Antenna(("2.4GHz Antenna"))
        
        subgraph Ports [Stick Ports]
            Slot1["Stick Slot 1"]
            Slot2["Stick Slot 2"]
            Slot3["Stick Slot 3"]
            Slot4["Stick Slot 4"]
        end
        
        % USB Maintenance Path
        PC == "USB 2.0 Uplink" ==> Hub
        Hub -. "D+/D- (Port 5)" .-> MCU
        Hub -. "D+/D- (Port 1)" .-> Slot1
        Hub -. "D+/D- (Port 2)" .-> Slot2
        Hub -. "D+/D- (Port 3)" .-> Slot3
        Hub -. "D+/D- (Port 4)" .-> Slot4

        % Ethernet & Power Path
        RJ45 == "Data & Power" ==> PoE_Power
        PoE_Power -- "5V DC" --> Hub
        PoE_Power -- "5V DC" --> MCU
        PoE_Power -- "Magnetics" --> Eth

        % Internal Logic Path (SPI)
        MCU == "SPI" ==> Eth
        MCU == "SPI" ==> UART_Bridge
        
        % Wireless Path
        MCU -. "802.15.4 / WiFi 6" .- Antenna
        Antenna -. "Matter / Zigbee / Thread" .-> HomeNet
        
        % SBU Operational Path
        UART_Bridge == "UART/SBU" ==> Slot1
        UART_Bridge == "UART/SBU" ==> Slot2
        UART_Bridge == "UART/SBU" ==> Slot3
        UART_Bridge == "UART/SBU" ==> Slot4
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
| **A4/B4** | **VBUS** | 5V Power | Bus Power |
| **GND** | **GND** | Ground | Common Ground |

> **Note:** The 1kΩ resistors on SBU lines are mandatory to protect the MCU if the stick is inserted into a non-compliant USB-C audio adapter.

---

## 5. Bill of Materials (BOM)

| Component | Description | Role |
| :--- | :--- | :--- |
| **ESP32-C6-WROOM-1** | RISC-V MCU | Application Host / Matter Coordinator |
| **WK2124** | SPI to 4x UART | High-speed Serial Bridge (256 Byte FIFO) |
| **W5500** | SPI Ethernet | Hardwired Network Interface |
| **FE1.1s / GL850G** | USB 2.0 Hub | 5-Port Configuration (or 4-Port + 1 Shared) |
| **Ag9905 / Discrete** | PoE Module | 802.3af to 5V DC Converter |

## 6. Software Support

* **ESP-IDF:** Native support for C6, W5500, and WK2124 drivers available.
* **Tasmota:** Potential for a custom build supporting the WK2124 driver for easy scripting.
* **ESPHome:** Custom component required for quad-UART bridging over SPI.

## 7. License

MIT License - Copyright (c) 2026 BUSWARE
