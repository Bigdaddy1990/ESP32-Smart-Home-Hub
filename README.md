# ESPHome Smart Home Hub — ESP32

> 🇬🇧 [English](#english) · 🇩🇪 [Deutsch](#deutsch)

---

<a name="english"></a>
# 🇬🇧 English

## Overview

ESPHome configuration for an ESP32 that acts as a central smart home hub for a two-unit building (ground floor / top floor). It reads one SML smart meter via optical IR head, derives energy direction for a second bidirectional meter via pulse counting, aggregates two balcony solar systems, and acts as a Bluetooth proxy for Home Assistant.

### The core challenge

The top floor meter (**DG**) is a **bidirectional meter** — its pulse LED fires for both grid consumption *and* feed-in. The SML pin unlock code was applied for but never delivered. This configuration solves the direction detection problem entirely in software using a permanently running base load (router) as an anchor.

---

## Hardware

| Component | Details |
|---|---|
| Microcontroller | ESP32 (generic `esp32dev` board) |
| Framework | ESP-IDF |
| EG meter (ground floor) | SML smart meter via IR optical head → `GPIO16` (RX) |
| DG meter (top floor) | Bidirectional meter, pulse output → `GPIO18` (INPUT_PULLUP) |
| Pulse rate | **10,000 imp/kWh** |
| Solar upper | Balcony power station, up to 800 W (via Home Assistant entity) |
| Solar lower | Balcony power station, up to 800 W (via Home Assistant entity) |
| Bluetooth | Passive BLE proxy for Home Assistant |
| Network | Static IP `192.168.2.40` |

---

## Wiring

Both optical interfaces use the same basic circuit: an NPN phototransistor (extracted from a Flying Fish IR module) and a single pull-up resistor. The phototransistor acts as a light-controlled switch:

- **LED off / dark** → transistor blocks → GPIO reads **HIGH** (idle)
- **LED on / pulse** → transistor conducts → GPIO reads **LOW** (event)

This matches the ESPHome `INPUT_PULLUP` logic for the pulse meter and produces correct UART logic levels (mark = HIGH = idle, space = LOW = data) for the SML reader — no signal inversion needed.

---

### Circuit 1 — DG Pulse Meter (GPIO18)

Reads the **visible pulse LED** of the bidirectional top floor meter.
Pull-up: **10 kΩ** (timing uncritical for individual pulses).

```
         3.3V
          │
         [ ]  10 kΩ  (pull-up)
          │
          ├──────────────────────────── GPIO18  (INPUT_PULLUP)
          │
          │   ┌──────────────────────┐
          └───┤ C  Phototransistor   │
              │    (NPN, Flying Fish)│◄──  faces meter pulse LED
              ├ E                    │
              └──────────────────────┘
          │
         GND

Signal behaviour:
  LED off  → transistor off  →  GPIO18 HIGH  (idle)
  LED pulse → transistor on  →  GPIO18 LOW   (counted as pulse event)
```

**Practical notes:**
- The phototransistor was desoldered from the Flying Fish module and installed in a custom 3D-printed housing that positions it precisely against the meter's pulse LED and shields all ambient light mechanically — no heat-shrink tubing over the sensor is needed; heat-shrink is only used on the connecting wires for strain relief and insulation
- The internal ESP32 pull-up (~45 kΩ) is present in addition to the external 10 kΩ; both work in parallel and together create a sufficiently fast rising edge
- Polarity: collector (C) toward 3.3V / pull-up, emitter (E) toward GND — verify with a multimeter in diode mode if unsure which leg is which

---

### Circuit 2 — EG SML Optical Head (GPIO16 / UART RX)

Reads the **infrared SML data LED** of the ground floor smart meter (9600 baud, 8N1).
Pull-up: **4.7 kΩ** — lower value for cleaner signal edges at UART baud rates.

```
         3.3V
          │
         [ ]  4.7 kΩ  (pull-up — lower value for UART signal integrity)
          │
          ├──────────────────────────── GPIO16  (UART RX)
          │
          │   ┌──────────────────────┐
          └───┤ C  Phototransistor   │
              │    (NPN, Flying Fish)│◄──  faces meter SML IR LED
              ├ E                    │
              └──────────────────────┘
          │
         GND

Signal behaviour (IEC 62056-21 optical UART, non-inverted):
  IR LED off  → transistor off  →  GPIO16 HIGH  =  UART mark  (idle / logic 1)
  IR LED on   → transistor on   →  GPIO16 LOW   =  UART space (logic 0)
```

**Practical notes:**
- The phototransistor was desoldered from the Flying Fish module and housed in a custom 3D-printed enclosure that fits flush against the meter's optical port and blocks all ambient light — this eliminates stray light as a source of error entirely; heat-shrink tubing is only applied to the connecting wires
- The SML IR LED is invisible to the naked eye; use a smartphone camera to verify the LED is active (most cameras can see near-IR)
- Keep the wire from the phototransistor to the ESP32 as short as practical; longer runs increase capacitance and slow the rising edge — if signal quality is poor, reduce the pull-up to 2.2 kΩ
- Do not connect the IR emitter LED from the Flying Fish module — only the phototransistor (receiver side) is used here

---

### Pin Summary

```
ESP32 GPIO16  ←  Phototransistor collector  (EG SML, 4.7 kΩ pull-up to 3.3V)
ESP32 GPIO18  ←  Phototransistor collector  (DG pulse, 10 kΩ pull-up to 3.3V)
ESP32 3.3V    →  Pull-up resistors (both circuits)
ESP32 GND     ←  Phototransistor emitters   (both circuits, shared ground)
```

---

## How it works

### EG (Ground Floor) — SML

The ground floor meter exposes a full SML data stream. The IR optical head reads it directly via UART (9600 baud, 8N1). The following OBIS codes are read:

| OBIS | Meaning | Unit |
|---|---|---|
| `1-0:1.8.0` | Total grid consumption | kWh |
| `1-0:2.8.0` | Total grid feed-in | kWh |
| `1-0:16.7.0` | Instantaneous net grid power | W (signed) |
| `1-0:96.50.1` | Meter type | Text |
| `1-0:96.1.0` | Device ID | Hex |

OBIS `1-0:16.7.0` already delivers the **signed net value** (positive = consumption, negative = feed-in). Solar production is included in this measurement. No further solar subtraction is performed.

---

### DG (Top Floor) — Pulse Meter + Direction Heuristic

The top floor meter has no accessible SML interface (PIN code pending). Its pulse LED is connected to GPIO18. Because it is a **bidirectional meter**, the LED pulses for both consumption and feed-in — the raw pulse count always reflects `|net power|` without sign.

#### Direction Detection

Direction is determined using the router's **guaranteed minimum base load** as an anchor:

```
pulse  = |V - S|          (what the meter emits, always ≥ 0)
est_v  = S - pulse        (estimated gross consumption under feed-in assumption)

Feed-in  if  est_v ≥ (router_min + hysteresis)  →  net = -pulse
Grid draw if  est_v <  (router_min - hysteresis)  →  net = +pulse
Dead zone if  |est_v - router_min| < hysteresis   →  hold last value
```

**Default parameters (adjust in `globals`):**

| Parameter | Default | Meaning |
|---|---|---|
| `dg_router_min_w` | `10.0 W` | Minimum router idle power — conservative lower bound |
| `dg_hysterese` | `5.0 W` | Hysteresis band around the threshold |
| Entry threshold | `15 W` | `router_min + hysteresis` |
| Exit threshold | `5 W` | `router_min - hysteresis` |
| Dead zone | `5–15 W` | No direction change, last value held |

#### Filter Pipeline (order matters)

```yaml
filters:
  - multiply: 6          # imp/min → W  (10,000 imp/kWh)
  - throttle_average: 5s # time-based average; may emit NaN if no sample
  - filter_out: nan      # MUST be after throttle_average — that's where NaN originates
  - heartbeat: 5s        # forces update even without new pulse data
```

#### Additional Safeguards in the Lambda

| Guard | Threshold | Purpose |
|---|---|---|
| Rogue pulse filter | `< 3 W` | Rejects EMC spikes; below physically possible base load |
| Plausibility check | `> max_load × 1.1` | Logged via `ESP_LOGW`, last valid value held |
| Spike filter | `Δ > 1000 W / cycle` | Rate-of-change limiter, logged and discarded |
| Dead zone | `5–15 W est_v` | No direction flip, last value held |
| Solar clamp | `0 – rated power` | Prevents negative night-time offsets from HA distorting heuristic |

---

## Sensors exposed to Home Assistant

### DG (Top Floor)

| Sensor | Unit | Description |
|---|---|---|
| `Netzlast DG` | W | Signed net grid power (+ = draw, − = feed-in) |
| `DG Verbrauch` | W | Estimated gross consumption (solar + net) |
| `DG Bezug Leistung` | W | Grid draw only |
| `DG Einspeisung Leistung` | W | Grid feed-in only |
| `Solar-Eigenverbrauch DG` | % | Self-consumption ratio of upper solar |
| `Dachgeschoss Bezug (Tag)` | kWh | Daily grid draw |
| `Dachgeschoss Einspeisung (Tag)` | kWh | Daily grid feed-in |
| `DG Zähler-Richtung` | Text | Current direction + est_v, including dead zone indication |

### EG (Ground Floor)

| Sensor | Unit | Description |
|---|---|---|
| `Netzlast EG` | W | Signed net grid power from SML |
| `Erdgeschoss Einspeisung Leistung` | W | Grid feed-in only |
| `Solar-Eigenverbrauch EG` | % | Self-consumption ratio of lower solar |
| `Erdgeschoss Gesamtbezug` | kWh | Cumulative grid consumption |
| `Erdgeschoss Gesamteinspeisung` | kWh | Cumulative grid feed-in |
| `Erdgeschoss Momentanleistung` | W | Instantaneous net power |

### System

| Sensor | Description |
|---|---|
| `ESP32 Uptime` | Uptime in seconds |
| `ESP32 IP Adresse` | Current IP address |
| `ESP32 SSID` | Connected network |
| `ESP32 MAC Adresse` | MAC address |
| `Bluetooth Status` | BLE proxy status |
| `Erdgeschoss Meter Type` | SML meter type string |
| `Erdgeschoss Device ID` | SML device ID (hex) |

---

## Configuration & Customisation

All installation-specific values are centralised in the `globals` section at the top of the file. No changes are needed anywhere else.

| Variable | Default | Adjust if… |
|---|---|---|
| `dg_router_min_w` | `10.0` | Your router's idle draw is significantly different |
| `dg_hysterese` | `5.0` | Direction is flipping unstably |
| `dg_max_delta` | `1000.0` | Your DG max load exceeds 800 W solar + headroom |
| `dg_max_leistung` | `3680.0` | Your circuit breaker rating is different (16 A × 230 V) |

Also adjust in the sensor section:

```yaml
# Solar rated power — clamp at HA sensor input
max_value: 800.0   # solar_oben and solar_unten separately
```

---

## Secrets

Create a `secrets.yaml` next to this file:

```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
api_encryption_key: "base64-key-here"
ota_password: "your-ota-password"
```

---

## Diagnostics

Press the **SML Diagnose** button in Home Assistant (or via the web interface at `192.168.2.40`) to log a full snapshot:

```
EG: 234.5 W | Bezug: 1.2345 kWh | Einspeisung: 0.0023 kWh
DG Puls (roh): 120.0 W | Netzlast: -80.0 W | Richtung: Einspeisung
DG Verbrauch: 220.0 W | Bezug (Tag): 0.8721 kWh | Einspeisung (Tag): 0.1234 kWh
Solar Oben: 300.0 W | Eigenverbrauch DG: 73.3 %
Grenzwerte: Router-Min=10W | Hysterese=5W | Max-Delta=1000W | Max-Last=3680W
```

The **DG Zähler-Richtung** text sensor in Home Assistant shows the live direction decision, including when the system is in the dead zone:

```
Einspeisung (120W → Netz, est_V=180W)
Bezug (45W ← Netz, est_V=-15W)
Grauzone (est_V=12W in [5–15W])
```

---

## Home Assistant Energy Dashboard

Navigate to **Settings → Dashboards → Energy** (or the Energy Dashboard → Configure) and assign the sensors as follows.

---

### Grid — Electricity consumption & return

Add **two separate grid connections** — one for EG, one for DG. This ensures the Energy Dashboard shows both meters separately and does not hide DG consumption as "untracked energy".

#### EG (Ground Floor) — SML meter

| Dashboard field | Sensor |
|---|---|
| Electricity consumed | `Erdgeschoss Gesamtbezug` |
| Electricity returned | `Erdgeschoss Gesamteinspeisung` |

Both sensors are read directly from the SML meter and accumulate indefinitely. No helpers required.

#### DG (Top Floor) — Pulse meter with integration

| Dashboard field | Sensor |
|---|---|
| Electricity consumed | `Dachgeschoss Gesamtbezug` |
| Electricity returned | `Dachgeschoss Gesamteinspeisung` |

These sensors use ESPHome's `integration` platform — they integrate the directional power sensors (W) over time (h) to produce permanently accumulating kWh values. They survive reboots via flash (`restore: true`, written every minute via `flash_write_interval`). No Riemann Sum helpers in HA are needed.

> **First start / flash erase:** After a full flash erase the integration counters reset to 0. If you need continuity, note the last HA value before flashing and adjust the Energy Dashboard offset manually afterwards.

---

### Live view — Instantaneous power flow

For the Energy Dashboard live view and the energy flow card to display real-time power arrows correctly, HA needs **non-negative directional power sensors** (signed values are not interpreted as bidirectional flow):

| Flow direction | Sensor | Notes |
|---|---|---|
| EG grid draw | `Erdgeschoss Bezug Leistung` | Positive portion of SML net value |
| EG grid feed-in | `Erdgeschoss Einspeisung Leistung` | Negative portion, sign-flipped |
| DG grid draw | `DG Bezug Leistung` | Positive portion of heuristic net value |
| DG grid feed-in | `DG Einspeisung Leistung` | Negative portion, sign-flipped |

In the Energy Dashboard configuration, assign each directional sensor pair to its respective grid connection under **"Current power usage"**.

---

### Solar production

The ESPHome configuration provides only instantaneous power (W) for the solar systems. Use the **energy (kWh) sensors from your inverter integrations directly**:

| Dashboard field | Sensor | Notes |
|---|---|---|
| Solar production | kWh entity from `solar_oben` inverter | Must be `total_increasing`, kWh |
| Solar production | kWh entity from `solar_unten` inverter | Must be `total_increasing`, kWh |

If the inverter integration only provides a power sensor (W), create a **Riemann Sum Integral** helper in HA (**Settings → Devices & Services → Helpers → Create helper → Riemann sum integral**), select the power entity as input and set the time unit to hours.

---

### Areas (floors)

ESPHome assigns a single area per device — since this ESP32 serves both floors, areas must be set in HA:

1. **Settings → Areas & Zones** — create `Erdgeschoss` and `Dachgeschoss` if not present
2. **Settings → Devices & Services** → find `ESP32 Smart Home Hub` → assign entities to their respective areas via the entity detail page or the area configuration

---

### Summary

```
Energy Dashboard
│
├── Grid connection: EG
│   ├── Consumed  →  Erdgeschoss Gesamtbezug       (SML, direct, no reset)
│   ├── Returned  →  Erdgeschoss Gesamteinspeisung  (SML, direct, no reset)
│   └── Live      →  Erdgeschoss Bezug/Einspeisung Leistung
│
├── Grid connection: DG
│   ├── Consumed  →  Dachgeschoss Gesamtbezug       (integration, flash-persistent)
│   ├── Returned  →  Dachgeschoss Gesamteinspeisung  (integration, flash-persistent)
│   └── Live      →  DG Bezug/Einspeisung Leistung
│
└── Solar panels
    ├── solar_oben   →  kWh entity from inverter (or Riemann Sum helper)
    └── solar_unten  →  kWh entity from inverter (or Riemann Sum helper)
```

---

## Known Limitations

- **Direction accuracy depends on the router's base load being stable.** If the router is switched off or load drops below `dg_router_min_w`, feed-in may be misclassified as consumption during the dead zone window. The hysteresis and dead zone handling minimise — but cannot fully eliminate — this edge case.
- **SML PIN code:** Once the unlock code is received from the grid operator, the entire pulse-counting and direction heuristic for DG can be replaced by a second SML sensor on a second UART — this would make the measurement fully authoritative.
- **Daily energy counters reset on reboot and at midnight.** `Dachgeschoss Bezug (Tag)` and `Dachgeschoss Einspeisung (Tag)` use ESPHome's `total_daily_energy` which is not flash-persistent — a reboot mid-day loses the accumulated value for that day. `flash_write_interval` does not help here; it only covers `restore_value` globals. For reliable long-term tracking use the Riemann Sum Integral helpers in Home Assistant (see Energy Dashboard section above).

---

## License

MIT

---
---

<a name="deutsch"></a>
# 🇩🇪 Deutsch

## Übersicht

ESPHome-Konfiguration für einen ESP32, der als zentraler Smart-Home-Hub in einem Zweifamilienhaus (Erdgeschoss / Dachgeschoss) eingesetzt wird. Er liest einen SML-Zähler per optischem IR-Kopf aus, leitet die Energierichtung eines zweiten Zweirichtungszählers per Impulsauswertung ab, aggregiert zwei Balkonkraftwerke und fungiert als Bluetooth-Proxy für Home Assistant.

### Die zentrale Herausforderung

Der Dachgeschoss-Zähler (**DG**) ist ein **Zweirichtungszähler** — seine Impuls-LED leuchtet sowohl bei Netzbezug *als auch* bei Einspeisung. Der SML-PIN-Freischaltcode wurde beantragt, aber nie geliefert. Diese Konfiguration löst das Richtungserkennungsproblem vollständig in Software, indem eine dauerhaft laufende Grundlast (Router) als Anker genutzt wird.

---

## Hardware

| Komponente | Details |
|---|---|
| Mikrocontroller | ESP32 (generisches `esp32dev` Board) |
| Framework | ESP-IDF |
| EG-Zähler (Erdgeschoss) | SML-Zähler per optischem IR-Kopf → `GPIO16` (RX) |
| DG-Zähler (Dachgeschoss) | Zweirichtungszähler, Impulsausgang → `GPIO18` (INPUT_PULLUP) |
| Impulswertigkeit | **10.000 imp/kWh** |
| Solar Oben | Balkonkraftwerk, bis 800 W (über Home-Assistant-Entity) |
| Solar Unten | Balkonkraftwerk, bis 800 W (über Home-Assistant-Entity) |
| Bluetooth | Passiver BLE-Proxy für Home Assistant |
| Netzwerk | Statische IP `192.168.2.40` |

---

## Verdrahtung

Beide optischen Schnittstellen verwenden dieselbe Grundschaltung: ein NPN-Phototransistor (aus einem Flying-Fish-IR-Modul) und ein einzelner Pull-up-Widerstand. Der Phototransistor arbeitet als lichtgesteuerter Schalter:

- **LED aus / dunkel** → Transistor sperrt → GPIO liest **HIGH** (Ruhezustand)
- **LED an / Puls** → Transistor leitet → GPIO liest **LOW** (Ereignis)

Das passt exakt zur `INPUT_PULLUP`-Logik des Pulse-Meters und erzeugt korrekte UART-Logikpegel (Mark = HIGH = Idle, Space = LOW = Daten) für den SML-Empfänger — keine Signalinvertierung notwendig.

---

### Schaltung 1 — DG Impulszähler (GPIO18)

Liest die **sichtbare Impuls-LED** des Zweirichtungszählers im Dachgeschoss.
Pull-up: **10 kΩ** (Timing unkritisch für einzelne Impulse).

```
         3,3V
          │
         [ ]  10 kΩ  (Pull-up)
          │
          ├──────────────────────────── GPIO18  (INPUT_PULLUP)
          │
          │   ┌──────────────────────┐
          └───┤ C  Phototransistor   │
              │    (NPN, Flying Fish)│◄──  zeigt auf Impuls-LED des Zählers
              ├ E                    │
              └──────────────────────┘
          │
         GND

Signalverhalten:
  LED aus    → Transistor sperrt  →  GPIO18 HIGH  (Ruhezustand)
  LED pulst  → Transistor leitet  →  GPIO18 LOW   (wird als Impuls gezählt)
```

**Praktische Hinweise:**
- Der Phototransistor wurde aus dem Flying-Fish-Modul ausgelötet und in ein selbstgedrucktes 3D-Gehäuse eingebaut, das ihn präzise vor der Impuls-LED des Zählers positioniert und Streulicht mechanisch vollständig abschirmt — Schrumpfschlauch über dem Sensor ist nicht nötig; er wird nur auf den Zuleitungen zur Zugentlastung und Isolierung verwendet
- Der interne ESP32-Pull-up (~45 kΩ) liegt parallel zum externen 10 kΩ und sorgt gemeinsam für ausreichend steile Flanken
- Polarität: Kollektor (C) zur 3,3V / Pull-up-Seite, Emitter (E) zu GND — mit Multimeter im Diodenmessmodus prüfen wenn die Beinchen nicht eindeutig beschriftet sind

---

### Schaltung 2 — EG SML-Optikkopf (GPIO16 / UART RX)

Liest den **infraroten SML-Datenstrom** des Erdgeschoss-Zählers (9600 Baud, 8N1).
Pull-up: **4,7 kΩ** — niedrigerer Wert für sauberere Signalflanken bei UART-Baudraten.

```
         3,3V
          │
         [ ]  4,7 kΩ  (Pull-up — niedrigerer Wert für UART-Signalqualität)
          │
          ├──────────────────────────── GPIO16  (UART RX)
          │
          │   ┌──────────────────────┐
          └───┤ C  Phototransistor   │
              │    (NPN, Flying Fish)│◄──  zeigt auf SML-IR-LED des Zählers
              ├ E                    │
              └──────────────────────┘
          │
         GND

Signalverhalten (optisches UART nach IEC 62056-21, nicht invertiert):
  IR-LED aus  → Transistor sperrt  →  GPIO16 HIGH  =  UART Mark  (Idle / Logik 1)
  IR-LED an   → Transistor leitet  →  GPIO16 LOW   =  UART Space (Logik 0)
```

**Praktische Hinweise:**
- Der Phototransistor wurde aus dem Flying-Fish-Modul ausgelötet und in ein selbstgedrucktes 3D-Gehäuse eingebaut, das bündig an die optische Schnittstelle des Zählers passt und Streulicht als Fehlerquelle vollständig eliminiert — Schrumpfschlauch wird nur auf den Zuleitungen verwendet
- Die SML-IR-LED ist für das bloße Auge unsichtbar; mit der Smartphone-Kamera prüfen ob die LED aktiv ist (die meisten Kameras sehen Nah-IR)
- Zuleitung zum ESP32 so kurz wie möglich halten; längere Leitungen erhöhen die Kapazität und verlangsamen die Anstiegsflanke — bei Signalproblemen Pull-up auf 2,2 kΩ reduzieren
- Die IR-Sender-LED des Flying-Fish-Moduls **nicht** anschließen — nur der Phototransistor (Empfängerseite) wird verwendet

---

### Pin-Übersicht

```
ESP32 GPIO16  ←  Phototransistor Kollektor  (EG SML, 4,7 kΩ Pull-up auf 3,3V)
ESP32 GPIO18  ←  Phototransistor Kollektor  (DG Impuls, 10 kΩ Pull-up auf 3,3V)
ESP32 3,3V    →  Pull-up-Widerstände (beide Schaltungen)
ESP32 GND     ←  Phototransistor Emitter    (beide Schaltungen, gemeinsame Masse)
```

---

## Funktionsweise

### EG (Erdgeschoss) — SML

Der Erdgeschoss-Zähler stellt einen vollständigen SML-Datenstrom bereit. Der IR-Kopf liest ihn direkt per UART (9600 Baud, 8N1) aus. Folgende OBIS-Codes werden ausgelesen:

| OBIS | Bedeutung | Einheit |
|---|---|---|
| `1-0:1.8.0` | Gesamtbezug | kWh |
| `1-0:2.8.0` | Gesamteinspeisung | kWh |
| `1-0:16.7.0` | Momentane Netzleistung (Netto) | W (vorzeichenbehaftet) |
| `1-0:96.50.1` | Zählertyp | Text |
| `1-0:96.1.0` | Geräte-ID | Hex |

OBIS `1-0:16.7.0` liefert bereits den **vorzeichenbehafteten Nettowert** (positiv = Bezug, negativ = Einspeisung). Die Solarproduktion ist im Messwert bereits enthalten. Eine zusätzliche Solar-Subtraktion erfolgt nicht.

---

### DG (Dachgeschoss) — Pulse Meter + Richtungsheuristik

Der Dachgeschoss-Zähler hat keine zugängliche SML-Schnittstelle (PIN-Code ausstehend). Seine Impuls-LED ist an GPIO18 angeschlossen. Da es ein **Zweirichtungszähler** ist, pulst die LED sowohl bei Bezug als auch bei Einspeisung — der Rohwert spiegelt immer `|Nettoleistung|` ohne Vorzeichen wider.

#### Richtungserkennung

Die Richtung wird über die **garantierte Mindestgrundlast** des Routers als Anker ermittelt:

```
pulse  = |V - S|         (was der Zähler emittiert, immer ≥ 0)
est_v  = S - pulse       (geschätzter Bruttoverbrauch unter Einspeisungsannahme)

Einspeisung wenn  est_v ≥ (router_min + hysterese)  →  netzlast = -pulse
Bezug       wenn  est_v <  (router_min - hysterese)  →  netzlast = +pulse
Grauzone    wenn  |est_v - router_min| < hysterese   →  letzten Wert halten
```

**Standardparameter (in `globals` anpassen):**

| Parameter | Standard | Bedeutung |
|---|---|---|
| `dg_router_min_w` | `10,0 W` | Minimale Router-Idle-Leistung — konservativer Mindestwert |
| `dg_hysterese` | `5,0 W` | Hystereseband um die Schwelle |
| Eintritt-Schwelle | `15 W` | `router_min + hysterese` |
| Austritts-Schwelle | `5 W` | `router_min - hysterese` |
| Grauzone | `5–15 W` | Kein Richtungswechsel, letzter Wert wird gehalten |

#### Filter-Pipeline (Reihenfolge entscheidend)

```yaml
filters:
  - multiply: 6          # imp/min → W  (10.000 imp/kWh)
  - throttle_average: 5s # zeitbasierter Mittelwert; kann NaN liefern wenn kein Sample
  - filter_out: nan      # MUSS nach throttle_average stehen — dort entsteht NaN
  - heartbeat: 5s        # erzwingt Update auch ohne neue Impulsdaten
```

#### Weitere Absicherungen im Lambda

| Absicherung | Schwelle | Zweck |
|---|---|---|
| Fehlpuls-Guard | `< 3 W` | Verwirft EMV-Spikes; unter physikalisch möglicher Grundlast |
| Plausibilitätsprüfung | `> max_last × 1,1` | Geloggt per `ESP_LOGW`, letzter gültiger Wert wird gehalten |
| Spike-Filter | `Δ > 1000 W / Zyklus` | Änderungsratenbegrenzer, geloggt und verworfen |
| Grauzone | `5–15 W est_v` | Kein Richtungsflip, letzter Wert wird gehalten |
| Solar-Clamp | `0 – Nennleistung` | Verhindert dass negative Nacht-Offsets aus HA die Heuristik stören |

---

## Sensoren in Home Assistant

### DG (Dachgeschoss)

| Sensor | Einheit | Beschreibung |
|---|---|---|
| `Netzlast DG` | W | Vorzeichenbehaftete Netzleistung (+ = Bezug, − = Einspeisung) |
| `DG Verbrauch` | W | Geschätzter Bruttoverbrauch (Solar + Netzlast) |
| `DG Bezug Leistung` | W | Nur Netzbezug |
| `DG Einspeisung Leistung` | W | Nur Netzeinspeisung |
| `Solar-Eigenverbrauch DG` | % | Eigenverbrauchsquote Solar Oben |
| `Dachgeschoss Bezug (Tag)` | kWh | Täglicher Netzbezug |
| `Dachgeschoss Einspeisung (Tag)` | kWh | Tägliche Netzeinspeisung |
| `DG Zähler-Richtung` | Text | Aktuelle Richtung + est_v, inkl. Grauzonenindikation |

### EG (Erdgeschoss)

| Sensor | Einheit | Beschreibung |
|---|---|---|
| `Netzlast EG` | W | Vorzeichenbehaftete Netzleistung aus SML |
| `Erdgeschoss Einspeisung Leistung` | W | Nur Netzeinspeisung |
| `Solar-Eigenverbrauch EG` | % | Eigenverbrauchsquote Solar Unten |
| `Erdgeschoss Gesamtbezug` | kWh | Kumulativer Netzbezug |
| `Erdgeschoss Gesamteinspeisung` | kWh | Kumulative Netzeinspeisung |
| `Erdgeschoss Momentanleistung` | W | Momentane Nettoleistung |

### System

| Sensor | Beschreibung |
|---|---|
| `ESP32 Uptime` | Laufzeit in Sekunden |
| `ESP32 IP Adresse` | Aktuelle IP-Adresse |
| `ESP32 SSID` | Verbundenes Netzwerk |
| `ESP32 MAC Adresse` | MAC-Adresse |
| `Bluetooth Status` | BLE-Proxy-Status |
| `Erdgeschoss Meter Type` | SML-Zählertyp-String |
| `Erdgeschoss Device ID` | SML-Geräte-ID (Hex) |

---

## Konfiguration & Anpassung

Alle anlagenspezifischen Werte sind im `globals`-Abschnitt am Anfang der Datei zentralisiert. Änderungen an anderen Stellen sind nicht notwendig.

| Variable | Standard | Anpassen wenn… |
|---|---|---|
| `dg_router_min_w` | `10,0` | Router-Idle-Leistung deutlich abweicht |
| `dg_hysterese` | `5,0` | Richtungserkennung instabil wechselt |
| `dg_max_delta` | `1000,0` | DG-Maximallast > 800 W Solar + Reserve |
| `dg_max_leistung` | `3680,0` | Sicherungsautomat DG hat anderen Wert (16 A × 230 V) |

Zusätzlich im Sensor-Abschnitt anpassen:

```yaml
# Nennleistung Wechselrichter — Clamp am HA-Sensor-Eingang
max_value: 800.0   # solar_oben und solar_unten separat
```

---

## Secrets

Eine `secrets.yaml` neben dieser Datei anlegen:

```yaml
wifi_ssid: "DeinNetzwerk"
wifi_password: "DeinPasswort"
api_encryption_key: "base64-schluessel"
ota_password: "ota-passwort"
```

---

## Diagnose

Den Button **SML Diagnose** in Home Assistant drücken (oder über die Weboberfläche unter `192.168.2.40`), um einen vollständigen Snapshot zu loggen:

```
EG: 234.5 W | Bezug: 1.2345 kWh | Einspeisung: 0.0023 kWh
DG Puls (roh): 120.0 W | Netzlast: -80.0 W | Richtung: Einspeisung
DG Verbrauch: 220.0 W | Bezug (Tag): 0.8721 kWh | Einspeisung (Tag): 0.1234 kWh
Solar Oben: 300.0 W | Eigenverbrauch DG: 73.3 %
Grenzwerte: Router-Min=10W | Hysterese=5W | Max-Delta=1000W | Max-Last=3680W
```

Der Text-Sensor **DG Zähler-Richtung** in Home Assistant zeigt die live Richtungsentscheidung — einschließlich Grauzonenindikation:

```
Einspeisung (120W → Netz, est_V=180W)
Bezug (45W ← Netz, est_V=-15W)
Grauzone (est_V=12W in [5–15W])
```

---

## Home-Assistant-Energie-Dashboard

Unter **Einstellungen → Dashboards → Energie** (oder Energie-Dashboard → Konfigurieren) die Sensoren wie folgt zuweisen.

---

### Stromnetz — Bezug und Einspeisung

**Zwei separate Netzanschlüsse** anlegen — einen für EG, einen für DG. Damit stellt das Energie-Dashboard beide Zähler getrennt dar und DG-Verbrauch erscheint nicht mehr als „nicht erfasster Verbrauch".

#### EG (Erdgeschoss) — SML-Zähler

| Dashboard-Feld | Sensor |
|---|---|
| Bezug | `Erdgeschoss Gesamtbezug` |
| Einspeisung | `Erdgeschoss Gesamteinspeisung` |

Beide Sensoren werden direkt vom SML-Zähler gelesen und akkumulieren dauerhaft. Keine Helfer erforderlich.

#### DG (Dachgeschoss) — Impulszähler mit Integration

| Dashboard-Feld | Sensor |
|---|---|
| Bezug | `Dachgeschoss Gesamtbezug` |
| Einspeisung | `Dachgeschoss Gesamteinspeisung` |

Diese Sensoren nutzen ESPHomes `integration`-Plattform — sie integrieren die gerichteten Leistungssensoren (W) über die Zeit (h) zu dauerhaft akkumulierenden kWh-Werten. Sie überleben Neustarts via Flash (`restore: true`, Schreibintervall 1 min über `flash_write_interval`). Riemann-Summen-Helfer in HA sind nicht mehr nötig.

> **Erster Start / Flash-Löschung:** Nach vollständigem Flash-Erase starten die Integrations-Zähler bei 0. Falls Kontinuität benötigt wird, den letzten HA-Wert vor dem Flashen notieren und den Energie-Dashboard-Offset danach manuell anpassen.

---

### Live-Ansicht — Momentanleistung und Energiefluss

Damit die Live-Ansicht im Energie-Dashboard und die Energiefluss-Karte die Leistungspfeile korrekt darstellen, benötigt HA **nicht-negative gerichtete Leistungssensoren** (vorzeichenbehaftete Werte werden nicht als bidirektionaler Fluss interpretiert):

| Flussrichtung | Sensor | Hinweis |
|---|---|---|
| EG Netzbezug | `Erdgeschoss Bezug Leistung` | Positiver Anteil des SML-Nettowerts |
| EG Einspeisung | `Erdgeschoss Einspeisung Leistung` | Negativer Anteil, Vorzeichen umgekehrt |
| DG Netzbezug | `DG Bezug Leistung` | Positiver Anteil des Heuristik-Nettowerts |
| DG Einspeisung | `DG Einspeisung Leistung` | Negativer Anteil, Vorzeichen umgekehrt |

Im Energie-Dashboard-Konfigurationsmenü das jeweilige Leistungssensor-Paar dem entsprechenden Netzanschluss unter **„Aktuelle Leistungsaufnahme"** zuweisen.

---

### Solarproduktion

Die ESPHome-Konfiguration liefert nur Momentanleistung (W) für die Solaranlagen. Die **Energie-Entitäten (kWh) direkt aus den Wechselrichter-Integrationen** verwenden:

| Dashboard-Feld | Sensor | Hinweis |
|---|---|---|
| Solarproduktion | kWh-Entität des `solar_oben`-Wechselrichters | Muss `total_increasing`, kWh sein |
| Solarproduktion | kWh-Entität des `solar_unten`-Wechselrichters | Muss `total_increasing`, kWh sein |

Falls die Wechselrichter-Integration nur einen Leistungssensor (W) bereitstellt, einen **Riemann-Summen-Integral**-Helfer in HA anlegen (**Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Riemann-Summen-Integral**), die Leistungs-Entität als Eingang wählen und die Zeiteinheit auf Stunden setzen.

---

### Areas (Stockwerke)

ESPHome weist einem Gerät eine einzige Area zu — da dieser ESP32 beide Stockwerke bedient, müssen Areas in HA gesetzt werden:

1. **Einstellungen → Areas & Zonen** — `Erdgeschoss` und `Dachgeschoss` anlegen falls noch nicht vorhanden
2. **Einstellungen → Geräte & Dienste** → `ESP32 Smart Home Hub` suchen → Entitäten über die Entitäts-Detailseite oder die Area-Konfiguration den jeweiligen Areas zuweisen

---

### Übersicht

```
Energie-Dashboard
│
├── Netzanschluss: EG
│   ├── Bezug        →  Erdgeschoss Gesamtbezug       (SML, direkt, kein Reset)
│   ├── Einspeisung  →  Erdgeschoss Gesamteinspeisung  (SML, direkt, kein Reset)
│   └── Live         →  Erdgeschoss Bezug/Einspeisung Leistung
│
├── Netzanschluss: DG
│   ├── Bezug        →  Dachgeschoss Gesamtbezug       (Integration, Flash-persistent)
│   ├── Einspeisung  →  Dachgeschoss Gesamteinspeisung  (Integration, Flash-persistent)
│   └── Live         →  DG Bezug/Einspeisung Leistung
│
└── Solaranlagen
    ├── solar_oben   →  kWh-Entität des Wechselrichters (oder Riemann-Summen-Helfer)
    └── solar_unten  →  kWh-Entität des Wechselrichters (oder Riemann-Summen-Helfer)
```

---

## Bekannte Einschränkungen

- **Richtungsgenauigkeit hängt von der Stabilität der Router-Grundlast ab.** Wenn der Router ausgeschaltet ist oder die Last unter `dg_router_min_w` fällt, kann Einspeisung in der Grauzone fälschlich als Bezug klassifiziert werden. Hysterese und Grauzonenbehandlung minimieren — können aber nicht vollständig eliminieren — diesen Randfall.
- **SML-PIN-Code:** Sobald der Freischaltcode vom Netzbetreiber vorliegt, kann die gesamte Impulsauswertung und Richtungsheuristik für das DG durch einen zweiten SML-Sensor auf einem zweiten UART ersetzt werden — was die Messung vollständig autoritär macht.
- **Tagesenergiezähler resetten bei Neustart und um Mitternacht.** `Dachgeschoss Bezug (Tag)` und `Dachgeschoss Einspeisung (Tag)` nutzen ESPHomes `total_daily_energy`, das nicht flash-persistent ist — ein Neustart verliert den bisher aufgelaufenen Tageswert. `flash_write_interval` hilft hier nicht; es sichert nur `restore_value`-Globals. Für zuverlässige Langzeitauswertung die Riemann-Summen-Integral-Helfer in Home Assistant verwenden (siehe Abschnitt Energie-Dashboard oben).

---

## Lizenz

MIT
