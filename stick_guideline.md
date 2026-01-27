# Busware "SBU-Native" Stick Design Guidelines

**Target Platform:** ESP32-C6 (RISC-V)  
**Version:** 1.0 (Draft)

![Status](https://img.shields.io/badge/Compatibility-ESP32--C6-green) ![Standard](https://img.shields.io/badge/Standard-USB--C_SBU-blue)

## 1. Introduction

This document defines the hardware design standard for future Busware RF sticks (Zigbee, CUL, Z-Wave). The goal is to create a **"Dual-Path"** device:

1.  **Legacy PC Mode:** Connects as a standard USB CDC Device via D+/D-.
2.  **Gateway Mode:** Connects as a raw UART Device via SBU pins (for the **bw-sbu-bridge**).

---

## 2. USB-C Connector Pinout

The stick **MUST** use a USB-C plug (Male). The wiring is non-standard but safe for legacy hosts.

| Pin | Name | Direction | ESP32-C6 Pin | Function | Protection |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **A6, B6** | **D+** | I/O | GPIO 13 | USB Serial/JTAG (PC Link) | Direct |
| **A7, B7** | **D-** | I/O | GPIO 12 | USB Serial/JTAG (PC Link) | Direct |
| **A8** | **SBU1** | **Output** | **U0TXD** (GPIO16) | **UART TX** (Gateway Link) | **1k Ohm Series Resistor** |
| **B8** | **SBU2** | **Input** | **U0RXD** (GPIO17) | **UART RX** (Gateway Link) | **1k Ohm Series Resistor** |
| **A4, B4** | **VBUS** | Power | VBUS | 5V Input | Fuse / Protection |
| **GND** | **GND** | Power | GND | Ground | - |
| **CC1, CC2** | **CC** | Input | - | 5.1k Ohm Pull-Down | **Required** for USB-C Power |

> **WARNING:** The 1k Ohm resistors on SBU1/SBU2 are mandatory. They prevent damage if the stick is inserted into a host utilizing Audio Adapter Accessory Mode.

---

## 3. ESP32-C6 Wiring & Configuration

### 3.1 UART Assignment
The ESP32-C6 has multiple UARTs. To support the "Dual-Path" architecture efficiently:

* **USB Controller (Internal):** Handled by hardware (GPIO 12/13). Maps to `/dev/ttyACM0` on Linux PCs. Used for **Flashing & Debugging**.
* **UART0 (Hardware Serial):** Maps to SBU1/SBU2. Used for **High-Speed Payload Data** to the Gateway.
* **UART1:** Connects to the internal RF Transceiver (e.g., CC1101) or other peripherals.

### 3.2 Bootloader & Strapping Pins
To ensure the stick boots correctly and can be flashed via USB:

* **GPIO 9 (BOOT):** Must be High during boot. The internal USB Serial/JTAG controller automatically handles the reset sequence (DTR/RTS) when connected to a PC. 
    * *Design Note:* You do **not** need a physical "Boot" button if the USB lines (D+/D-) are connected.
* **GPIO 8:** High during boot (Internal Pull-up is usually sufficient, check datasheet).
* **Enable (Reset):** Connect `EN` to a simple RC delay circuit (10k Ohm / 1uF) and the internal USB controller reset logic.

---

## 4. Power Supply Design

RF Sticks can have high peak currents (TX bursts).

* **LDO Regulator:** Use a Low-Dropout Regulator (LDO) capable of at least **600mA** (e.g., AP2112K-3.3 or TLV75733).
* **Capacitors:** Place a 10uF (min) ceramic capacitor close to the LDO input and a 22uF (min) close to the ESP32-C6 power pins to buffer Zigbee TX spikes.

---

## 5. Wiring Diagram (Schematic View)

```mermaid
graph LR
    subgraph USB_C_Plug [USB-C Connector]
        VBUS((VBUS))
        GND((GND))
        DP[D+ A6/B6]
        DM[D- A7/B7]
        SBU1[SBU1 A8]
        SBU2[SBU2 B8]
        CC[CC1/CC2]
    end

    subgraph ESP32_C6 [ESP32-C6-WROOM-1]
        GPIO13[GPIO 13 / USB D+]
        GPIO12[GPIO 12 / USB D-]
        GPIO16[GPIO 16 / U0TXD]
        GPIO17[GPIO 17 / U0RXD]
        PWR[3V3 / GND]
    end

    subgraph Protection
        R_TX[1k Resistor]
        R_RX[1k Resistor]
        R_CC[5.1k Pull-Down]
        LDO[3.3V LDO Regulator]
    end

    % Power Flow
    VBUS --> LDO
    GND --> LDO
    LDO --> PWR
    CC -- "GND" --> R_CC

    % USB Data Flow (PC)
    DP === GPIO13
    DM === GPIO12

    % SBU Data Flow (Gateway)
    GPIO16 -- "TX Signal" --> R_TX
    R_TX --> SBU1
    
    SBU2 --> R_RX
    R_RX -- "RX Signal" --> GPIO17
```

---

## 6. PCB Layout Recommendations

1.  **Antenna Placement:** Place the PCB Antenna or SMA connector at the opposite end of the USB connector. Keep the "Keep-Out Area" free of copper on all layers.
2.  **Differential Pairs:** Route D+ (GPIO13) and D- (GPIO12) as a differential pair with 90 Ohm impedance.
3.  **SBU Routing:** Route SBU1/SBU2 as standard digital traces. No specific impedance matching required, but keep them short.
4.  **Thermal:** The ESP32-C6 generates heat. Use the bottom layer as a solid Ground plane for heat dissipation.

## 7. Firmware Requirements

To support Auto-Detection:

* **Boot Check:** The firmware should check if D+/D- are enumerated (USB VBUS present is not enough, as Gateway provides power too).
* **Stream Mirroring:** Ideally, the firmware should mirror the data stream:
    * `Radio Packet` -> `UART0` (SBU) **AND** `USB-CDC` (if connected).
    * This allows simultaneous logging via USB while the Gateway operates via SBU.
