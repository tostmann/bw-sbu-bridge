# Power Architecture: Industrial 4-Port RF Hub
**Application:** High-Stability USB Hub for ESP32/EnOcean Sticks
**Target Host:** Raspberry Pi / Industrial PC
**Version:** v1.0

---

## 1. The Problem: Why Standard Hubs Fail

Operating four RF sticks (ESP32-C6 + EnOcean TCM515) simultaneously creates specific power demands that standard USB hubs or a direct Raspberry Pi connection cannot reliably meet:

1.  **Peak Current (Bursts):** Each stick can draw up to 300mA during transmission bursts. 4 sticks + hub electronics require a headroom of **> 2.5 Amps**.
2.  **The Raspberry Pi Limitation:** A Raspberry Pi 4 or 5 often limits the total current across all USB ports to **1.2A**. If all sticks transmit at once, the voltage sags, leading to brownouts and reboots.
3.  **Voltage Stability (RF Performance):** RF transceivers are sensitive to ripple and voltage drops. A long USB cable from a PC often delivers only 4.5V under load, significantly reducing RF range.
4.  **Lack of PD Sourcing:** A Raspberry Pi cannot output high voltages (9V/12V) to compensate for cable losses.

---

## 2. The Solution: "Split-Architecture" with High-Side Injection

We physically separate **Data** and **Power** into two distinct USB-C ports. Using a **Diode-ORing circuit**, we ensure an uninterruptible power supply from the strongest available source.

### 2.1 The Ports

* **Port A (DATA / Uplink):** USB-C. Connects only D+/D- and GND to the Host PC. The 5V (VBUS) line serves only as a fallback/emergency power source.
* **Port B (POWER / Inject):** USB-C. Dedicated to power only. Equipped with a **PD-Trigger (CH224K)** and a **Synchronous Buck Converter**.

### 2.2 Core Components

1.  **PD Decoy (CH224K):** Negotiates **9V** from the power supply (the "sweet spot" for compatibility with power banks and wall chargers).
2.  **Synchronous Buck Converter (e.g., MP2315):** Efficiently steps down the 9V to **5.3V**.
    * *Why 5.3V?* To compensate for protection diode voltage drops and to ensure this source is "stronger" than the PC (5.0V).
    * *Why Synchronous?* To allow **100% Duty Cycle Mode** in "Dumb 5V" scenarios (see below).
3.  **Diode-ORing (2x SS34):** Two high-current Schottky diodes merge the sources. The source with the higher voltage automatically takes the load.

---

## 3. Scenario Analysis (Behavioral Table)

How the hub reacts to different power sources connected by the user:

### Scenario A: Ideal Case (USB-PD Power Supply 9V/12V/15V)
*The user connects a modern USB-C charger or a PD power bank to Port B.*

* **Process:** CH224K negotiates 9V. Buck converter regulates to a stable **5.3V**.
* **Logic:** * Voltage PSU path (after diode): `5.3V - 0.3V = 5.0V`
    * Voltage PC path (after diode): `5.0V - 0.3V = 4.7V`
* **Result:** The diode towards the PC **reverse-biases (blocks)**. 100% of the current is drawn from the external PSU. The PC port is not loaded.
* **Status:** ✅ **Perfect.** Maximum power (3A+), cleanest voltage.

### Scenario B: Fallback (Dumb 5V Charger / Legacy Power Bank)
*The user connects an old 5V phone charger to Port B. PD negotiation fails.*

* **Process:** CH224K receives no PD response; input remains at 5.0V.
* **Buck Behavior:** Since $V_{in} (5V) < V_{target} (5.3V)$, the synchronous regulator enters **100% Duty Cycle mode**. It passes the 5V through with minimal loss (approx. 4.9V).
* **Logic:**
    * Voltage PSU path (after diode): `4.9V - 0.3V = 4.6V`
    * Voltage PC path (after diode): `5.0V - 0.3V = 4.7V`
* **Result:** **Load Sharing.** Since the voltages are nearly identical and diodes have "soft" curves, the sticks draw current from **both** sources simultaneously.
* **Status:** ⚠️ **Good.** The external charger significantly offloads the PC port. Operation is stable.

### Scenario C: Bus-Powered (No external Power Supply)
*The user forgets the power supply or wants a quick test.*

* **Process:** Port B is disconnected (0V).
* **Logic:** Only the PC provides 5V.
* **Result:** System voltage approx. **4.7V** (sufficient for the LDOs on the sticks). Current is limited to what the PC port provides (usually 500mA - 900mA).
* **Status:** ⚠️ **Limited.** Operation of 1-2 sticks is reliable. Using 4 sticks risks a PC port overcurrent shutdown.

---

## 4. Schematic Concept

```mermaid
graph LR
    subgraph INPUTS
        PC_USB[Host USB Port] -- "5.0V / 0.5A" --> D1
        WALL_USB[USB-C PD Wall Plug] -- "9V / 3.0A" --> CH224K
    end

    subgraph POWER_STAGE
        CH224K[PD Trigger Chip] -- "Request 9V" --> BUCK
        BUCK[Sync Buck Converter MP2315]
        
        Note1[Setting: V_out = 5.3V] --- BUCK
        
        WALL_USB -- "9V (or 5V fallback)" --> BUCK
        BUCK -- "5.3V (or 4.9V)" --> D2
    end

    subgraph ORING_PROTECTION
        D1[Diode SS34] 
        D2[Diode SS34]
        
        D1 --> SYS_RAIL((System 5V Rail))
        D2 --> SYS_RAIL
    end

    subgraph LOAD
        SYS_RAIL -- "Buffer Caps" --> CAPS[470uF Polymer]
        SYS_RAIL -- "Power Switch" --> HUB[USB Hub Chip]
        SYS_RAIL -- "Current Limit" --> STICK1[Stick 1]
        SYS_RAIL --> STICK2[Stick 2]
        SYS_RAIL --> STICK3[Stick 3]
        SYS_RAIL --> STICK4[Stick 4]
    end

    style Note1 fill:#ff9,stroke:#333,stroke-width:2px
