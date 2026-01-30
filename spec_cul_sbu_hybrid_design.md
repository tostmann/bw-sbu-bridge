# Hardware Design Specification: Hybrid CUL-SBU Stick (ATmega32u4)

## 1. Abstract
This design enables "Dual-Path" functionality on ATmega32u4-based RF sticks (CUL). It ensures 100% backward compatibility with legacy firmware (culfw) while supporting the new "SBU-Native" UART standard for next-gen gateways.

## 2. Core Architecture

### The Problem
* **Legacy FW** expects CC1101 GDO signals on **PD2/PD3** (Hardware INT2/INT3).
* **SBU-Native FW** requires **PD2/PD3** for Hardware UART (Serial1).

### The Solution: "Switched & Mirrored" Topology
We use a **Dual SPDT Analog Switch** (Multiplexer) to toggle PD2/PD3, while simultaneously mirroring the radio signals to backup pins (PB4/PB5) for the new firmware.

---

## 3. Circuit Logic & Schematic

### A. Multiplexer Configuration
* **Component:** TS5A23157 (or equivalent Dual SPDT)
* **Control Signal:** **PD4** (or any free GPIO)
* **State Requirement:** Must have a **10k Pull-Down Resistor** to GND (Default = LOW = Legacy).

| Switch Pin | Connected To | Function |
| :--- | :--- | :--- |
| **IN (Select)** | **ATmega PD4** | **LOW** = Legacy (Radio), **HIGH** = SBU (UART) |
| **COM1** | **ATmega PD3** | Shared Pin (TX / INT3) |
| **NC1** (Default) | **CC1101 GDO0** | Radio Interrupt (Legacy Path) |
| **NO1** (Active) | **USB-C A8** | SBU1 UART TX (via 1k Resistor) |
| **COM2** | **ATmega PD2** | Shared Pin (RX / INT2) |
| **NC2** (Default) | **CC1101 GDO2** | Radio Interrupt (Legacy Path) |
| **NO2** (Active) | **USB-C B8** | SBU2 UART RX (via 1k Resistor) |

### B. The "Permanent" Radio Link
To allow the SBU-Native firmware to receive radio packets while PD2/PD3 are busy with UART, the GDO lines must be physically forked (Y-connection).

* **CC1101 GDO0** --> also connected to **ATmega PB4** (PCINT4)
* **CC1101 GDO2** --> also connected to **ATmega PB5** (PCINT5)

---

## 4. Connector Pinout (USB-C Male)

| USB Pin | Signal | Connection Requirement |
| :--- | :--- | :--- |
| **A6, B6** | **D+** | Direct to ATmega D+ |
| **A7, B7** | **D-** | Direct to ATmega D- |
| **A8** | **SBU1** | To Mux **NO1** via **1kΩ Series Resistor** |
| **B8** | **SBU2** | To Mux **NO2** via **1kΩ Series Resistor** |
| **CC1** | **CC** | **5.1kΩ Pull-Down** to GND |
| **CC2** | **CC** | **5.1kΩ Pull-Down** to GND |
| **A4, B4** | **VBUS** | 5V Input (Fuse recommended) |

---

## 5. Firmware Operations

### Mode 1: Legacy (Standard culfw)
* **GPIO State:** PD4 is Input/Low (held by Pull-Down).
* **Mux State:** NC connected to COM.
* **Behavior:** ATmega sees Radio on PD2/PD3. USB-CDC used for communication.

### Mode 2: SBU-Native (NextGen FW)
* **GPIO State:** Firmware sets PD4 to **HIGH**.
* **Mux State:** NO connected to COM.
* **UART:**  active on PD2/PD3 (communicating via SBU pins).
* **Radio:** Firmware ignores INT2/INT3. Instead, it enables **PCINT on PB4/PB5** to read GDO signals.
