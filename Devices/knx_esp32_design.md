# Hardware-Design Spezifikation: KNX Bus-Powered ESP32-C6

## 1. Management Summary
Dieses Design ermöglicht den **batterielosen Betrieb** eines ESP32-C6 direkt am KNX-Bus. Durch die Nutzung der hohen Busspannung zur Energiespeicherung (High-Voltage-Buffering) werden die hohen Sendestromspitzen des WLAN-Moduls abgefangen, ohne den Bus-Fan-In (max. Stromaufnahme) zu verletzen.

## 2. Power-Anforderungen (ESP32-C6)
Der ESP32-C6 ist effizient, stellt aber gepulste Anforderungen an die Versorgung, die weit über dem liegen, was der KNX-Bus (max. 1.2W) kontinuierlich liefern kann.

| Betriebsmodus | Durchschnittsstrom | Dauer | Spitzenstrom (Peak) | Bemerkung |
| :--- | :--- | :--- | :--- | :--- |
| **Deep Sleep** | 7 µA | Unbegrenzt | - | Reiner Standby |
| **Active RX** | 85 mA | Kontinuierlich | 90 mA | Listen Mode / Thread |
| **Active TX** | 280 - 350 mA | < 5 ms | **400 mA** | WiFi Beacon / Daten (Worst Case) |

* **Das Problem:** Der NCN5130 kann bei 3.3V maximal ca. **270 - 300 mA** kontinuierlich liefern (bei 40mA Fan-In).
* **Die Lücke:** Während eines TX-Bursts fehlen ca. **100 - 130 mA**, die nicht aus dem Bus kommen können.

## 3. Die Lösung: High-Voltage Tank
Anstatt riesige Kondensatoren auf der 3.3V-Seite zu verbauen, speichern wir die Energie auf der **28V-Seite (VFILT)**.
Die Energie in einem Kondensator wächst quadratisch zur Spannung ($E = \frac{1}{2} C U^2$).

### Rechnerischer Nachweis
Wir nehmen den **Worst-Case** an: Einen WiFi-TX-Burst von **400 mA** für **5 ms**.

1.  **Leistungsbedarf Last (ESP):**
    $$P_{Load} = 3.3V \times 0.400A = 1.32 W$$
2.  **Verfügbare Bus-Leistung (Supply):**
    $$P_{Bus} = 29V \times 0.040A \times 0.85 (\eta_{DCDC}) \approx 0.98 W$$
3.  **Leistungsdefizit:**
    $$\Delta P = 1.32 W - 0.98 W = 0.34 W$$
4.  **Energiebedarf aus Puffer (für 5ms):**
    $$E_{Burst} = 0.34 W \times 0.005 s = 1.7 mJ (Milli-Joule)$$

### Puffer-Kapazität (Der Tank)
Wir verwenden **2x 270µF Polymer** parallel = **540 µF** an ca. **28 V**.

* **Gespeicherte Energie im Tank:**
    $$E_{Tank} = 0.5 \times 0.000540 F \times (28V)^2 \approx 211 mJ$$

### Ergebnis
Der Burst entnimmt **1.7 mJ** aus einem Tank, der **211 mJ** enthält.
Der Spannungsabfall am VFILT-Pin beträgt dabei:
$$\Delta U \approx \frac{E_{Burst}}{C \times U} = \frac{0.0017}{0.000540 \times 28} \approx 0.11 V$$

**Fazit:** Die Spannung am Tank sinkt während eines massiven WiFi-Bursts nur von **28.0 V auf ca. 27.9 V**. Der DC/DC-Wandler regelt dies völlig unbeeindruckt aus. Das System verfügt über eine **Sicherheitsreserve von Faktor >100**.

## 4. Kern-Komponenten (BOM)
* **Transceiver:** ON Semi `NCN5130` (DC1 konfiguriert auf 3.6V)
* **MCU:** Espressif `ESP32-C6`
* **Tank (VFILT):** 2x KEMET `A768MS277M1VLAE022` (270µF, 35V, Polymer, AEC-Q200)
* **USB-LDO:** Diodes Inc. `AP2112K-ADJ` (oder `TLV757P`) eingestellt auf **3.6 V**.
* **Oring-Dioden:** 2x Schottky `SS14` (oder `BAT60`)

## 5. Schaltplan-Architektur

~~mermaid
graph TD
    KNX[KNX Bus 29V] ==>|I_lim 40mA| NCN[NCN5130 Transceiver]
    NCN -- "VFILT 28V (High Energy)" --> TANK["Tank: 2x 270µF Polymer"]
    TANK -. "Pufferung (Reserve 211 mJ)" .-> NCN

    NCN -- "DC1 Out 3.6V" --> D1[Schottky D1]
    USB[USB-C VBUS 5V] --> LDO["LDO 3.6V"]
    LDO --> D2[Schottky D2]

    D1 ==>|"3.3V (via 3.6V DC1)"| MIX((V_ESP))
    D2 ==>|"3.3V (via 3.6V LDO)"| MIX

    MIX ==> ESP[ESP32-C6 VDD]
    MIX -- Decoupling --> C_CER[Ceramic 22µF]

    NCN -- RESETB --> EN[ESP32 EN]
    
    NOTE[Symmetrie: Beide Quellen liefern stabil 3.3V]
    style NOTE fill:#fff5ad,stroke:#d4b400,color:black
~~

## 6. Implementation Details

### A. Spannungseinstellung (Symmetrisch 3.6 V)
Um den Spannungsabfall der Schottky-Dioden (~0.3 V) auszugleichen, werden beide Quellen angehoben:
1.  **KNX (DC1):** Feedback-Widerstände am NCN5130 so dimensionieren, dass $V_{out} = 3.6 V$.
2.  **USB (LDO):** Spannungsteiler am Adjustable LDO auf 3.6 V einstellen.

* **Ergebnis:** Der ESP32 erhält stabil **3.3 V**, weit entfernt vom Brownout-Level (2.8 V).

### B. VFILT Layout Guidelines
Die Verbindung zwischen NCN5130 `Pin VFILT` und dem positiven Pol der Polymer-Elkos muss **niederinduktiv** sein.
* Leiterbahnbreite > 0.6 mm.
* Direkte Anbindung an die GND-Plane.
* Platzierung < 10 mm vom IC entfernt.

### C. Priorisierung
* **Szenario 1 (Nur KNX):** Versorgung rein über DC1 + Tank.
* **Szenario 2 (USB gesteckt):** Da USB eine "harte" Spannungsquelle ist und der NCN strombegrenzt ist, übernimmt der LDO automatisch die Hauptlast. Das schont den Bus während Entwicklungsarbeiten.
