# Busware "SBU-Native" Combo Stick Specification

**Version:** 1.0 (Final Design)
**Architecture:** Dual-Path Gateway Stick
**Modules:** Espressif ESP32-C6-MINI-1 + Silicon Labs MGM240PA32VNN3

![Status](https://img.shields.io/badge/Status-Production_Ready-green)

---

## 1. Abstract

This design defines the ultimate Zigbee/Matter/Thread stick. It combines a powerful **ESP32-C6** Host with a High-Power **MGM240P** Radio (+20 dBm).

**Key Features:**
* **Dual-Path:** Works as standard USB Device (Legacy) AND raw UART Device (SBU-Native).
* **Self-Healing:** ESP32 can reset and re-flash the Radio via internal SWD.
* **Industrial Grade:** Full EMI filtering and ESD protection.
* **High Power RF:** External Antenna via SMA/U.FL for maximum range.

---

## 2. Bill of Materials (Critical Components)

| Component | Part Number | Function | Critical Spec |
| :--- | :--- | :--- | :--- |
| **Host MCU** | **ESP32-C6-MINI-1** | Controller & USB Bridge | RISC-V, USB-Serial-JTAG |
| **Radio Module** | **MGM240PA32VNN3** | Zigbee/Thread Radio | +20dBm, **RF Pin (No PCB Ant)** |
| **LDO Regulator**| **TI TLV75733** | Power Supply | 3.3V, **1A Output**, Low Noise |
| **VBUS Ferrite** | **Würth 742792040** | Power Filter | **Z>600R, I>1A, R_DC<0.2R** |
| **Data CMC** | **Murata DLW21SN900HQ2**| EMI Filter (D+/D-) | 90 Ohm @ 100MHz |
| **ESD Protection**| **ST USBLC6-2** | Input Protection | Ultra-low C (<1pF) |

---

## 3. Pin Mapping

### 3.1 External Interface (USB-C Plug)

| Pin | Name | Direction | Connected To | Note |
| :--- | :--- | :--- | :--- | :--- |
| **A6/B6** | **D+** | I/O | **ESP32 GPIO 13** | Via CMC & ESD Protection |
| **A7/B7** | **D-** | I/O | **ESP32 GPIO 12** | Via CMC & ESD Protection |
| **A8** | **SBU1** | Output | **ESP32 GPIO 16** | **1kΩ Series Resistor** |
| **B8** | **SBU2** | Input | **ESP32 GPIO 17** | **1kΩ Series Resistor** |
| **CC1/2**| **CC** | Input | GND | **5.1kΩ Pull-Down** |

### 3.2 Internal Interconnect (ESP32 -> MGM240P)

| Signal | ESP32 Pin | MGM240P Pin | Pad # | Function |
| :--- | :--- | :--- | :--- | :--- |
| **UART_TX** | **GPIO 4** | **PA06** (RX) | 13 | Payload Data (C6 -> MGM) |
| **UART_RX** | **GPIO 5** | **PA05** (TX) | 12 | Payload Data (C6 <- MGM) |
| **SWCLK** | **GPIO 0** | **PA01** | 8 | **Factory Flashing / Rescue** |
| **SWDIO** | **GPIO 2** | **PA02** | 9 | **Factory Flashing / Rescue** |
| **RESET** | **GPIO 7** | **RESETn** | 31 | Hard Reset (Active Low) |

---

## 4. RF & Layout Guidelines (MGM240P VNN3)

Since the **VNN3** variant has no internal antenna, the RF routing is critical.

1.  **RF Path:** Route from **Pad 33 (RFOUT)** to SMA/U.FL connector.
2.  **Impedance:** Trace must be calculated for **50 Ohm** (Microstrip).
3.  **Grounding:** Pads **32 and 34** (flanking RFOUT) must connect immediately to the reference Ground plane.
4.  **Power:** Place **10µF** capacitor close to Pad 15 (VDD).

---

## 5. Wiring Diagram

```mermaid
graph LR
    subgraph USB_C ["USB-C Connector"]
        VBUS((VBUS))
        DP["D+"]
        DM["D-"]
        SBU1["SBU1/TX"]
        SBU2["SBU2/RX"]
    end

    subgraph EMI_Filter ["EMI / Protection"]
        L_PWR["Ferrite 1A"]
        CMC["Common Mode Choke"]
        ESD["TVS Array"]
        LDO["LDO 3.3V 1A"]
    end

    subgraph Host ["ESP32-C6-MINI-1"]
        USB_PHY["Internal USB"]
        UART0["UART0 SBU"]
        UART1["UART1 Radio"]
        SWD_M["BitBang SWD"]
    end

    subgraph Radio ["MGM240P +20dBm"]
        UART_S["UART Slave"]
        SWD_S["SWD Port"]
        RF["RFOUT Pad 33"]
    end

    %% Power Path
    VBUS --> L_PWR --> LDO
    LDO --> Host
    LDO --> Radio

    %% Legacy USB Path
    DP & DM === ESD === CMC === USB_PHY

    %% SBU Native Path
    SBU1 -- "1k" --> UART0
    SBU2 -- "1k" --> UART0

    %% Internal Data Link
    UART1 <==> UART_S

    %% Rescue / Flash Link
    SWD_M -.-> SWD_S
    Host -- "RST" --> Radio

    %% RF Output
    RF -.-> SMA((SMA Ant))

    style Radio fill:#ff9,stroke:#333,stroke-width:2px
    style L_PWR fill:#f96,stroke:#333
