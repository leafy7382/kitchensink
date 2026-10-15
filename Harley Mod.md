Harley ECU reverse engineering and protocols
Open source Harley ECU projects
Unicorn Security Dynojet reverse engineering
Unicorn Security ECU protocol details
Fuelpak FP4 bluetooth and flash details
Harley ECM types by model year
2021+ Harley ECU tuning security
Euro 5 Harley ECU tuning anti-tampering

Yes, **it is technically possible** to create an app to tune Harley-Davidson motorbike computers—products like the **Vance & Hines Fuelpak (FP3 / FP4)** and **Dynojet Power Vision (PV4)** prove the business model. 

However, building a flash-tuning app from scratch is a **major reverse-engineering and engineering undertaking**. It is helpful to understand the feasibility, the exact technical hurdles, how existing solutions work, and the legal implications.

---

### 1. Harley-Davidson ECM Generations & Hardware

Harley-Davidson has used three distinct electronic architectures over the fuel-injected era. Your app and hardware must target one or more of these:

| Generation | Era / Models | Protocol & Data Link | ECM Supplier & Architecture | Tuning Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Legacy Delphi** | 2001–2010 (Softail/Dyna), 2001–2013 (Touring/Sportster) | **SAE J1850 VPW** (10.4 kbps) via 4-pin Deutsch DT connector | Delphi (Motorola/Freescale MCUs) | **Medium** (Obsolete physical bus; requires specialized J1850 transceivers) |
| **HDLAN (CAN-bus)** | 2011/2012+ (Softail/Dyna), 2014–2020 (Touring Rushmore / M8) | **CAN Bus (500 kbps)** via 6-pin Deutsch / Molex connector | Delphi MT21M / MT05 (Infineon TriCore / PowerPC) | **High** (Standard CAN physical layer; proprietary seed-key security) |
| **Modern Euro 5 / 5+** | 2021–Present (M8 Softail/Touring, RevMax: Pan America, Sportster S) | **CAN / CAN-FD** via standardized **Red 6-Pin OBD-II** (ISO 19689) | Delphi / Continental / Bosch | **Very High** (Encrypted binaries, anti-tamper flags, secure bootloader) |

---

### 2. The Four Pillars Needed to Build a Tuning App

To create an app like the FP4 or PV4, you need to solve four distinct engineering layers:

#### A. Hardware: The Vehicle Communication Interface (VCI Dongle)
Your mobile app (iOS/Android) cannot plug into the bike directly; it needs a **Bluetooth Low Energy (BLE) or Wi-Fi dongle**:
* **For 2014–2020 (CAN) and 2021+ (Euro 5):** You can build an ESP32 or STM32-based hardware adapter paired with a CAN transceiver (e.g., SN65HVD230 or MCP2515). Standard off-the-shelf OBDLink MX+ dongles can handle basic diagnostic communication, but custom flashing typically requires high-throughput raw ISO-TP frames without ELM command-set latency.
* **For 2001–2013 (J1850 VPW):** Modern microcontrollers do not have native J1850 peripherals. You must incorporate a dedicated J1850 transceiver chip (e.g., NXP MC33390 or STN1110).

#### B. The Communication & Security Wall (UDS & Seed-to-Key)
Modern Harley ECMs follow **ISO 14229 (Unified Diagnostic Services - UDS)** over **ISO 15765-2 (ISO-TP)**. While reading live PIDs (RPM, speed, engine temp) is open via standard diagnostic services, **writing to flash memory is strictly locked**:
1. **Diagnostic Session Control (`0x10`):** App requests a Programming Session (`0x10 0x02`).
2. **Security Access (`0x27`):** The ECM provides a pseudo-random seed (`0x27 0x01`). The app must return the mathematically valid response key (`0x27 0x02`).
3. **The Seed-Key Algorithm:** This algorithm is secret and hardcoded inside the ECM firmware. To reverse-engineer it, researchers typically:
   * Dump the ECM ROM on a bench via BDM/JTAG pads.
   * Disassemble the binary in **Ghidra** or **IDA Pro**.
   * Locate the `0x27` service routine and extract the mathematical transformation (bitwise operations, S-boxes, or polynomial shifts) that turns the seed into the key.

#### C. The Calibration Layer: ROM Definition Files (XDF / A2L)
Even if you can read and write raw binary (`.bin`) from the ECM, a raw binary dump is just a blob of hexadecimal numbers. To make a tuning app, you must know the exact memory offsets for:
* **Volumetric Efficiency (VE) Tables:** Separate tables for the **Front Cylinder** and **Rear Cylinder** (Harley 45° V-Twins have radically different airflow characteristics between cylinders due to manifold design and heat).
* **Spark Advance / Ignition Timing:** Advance vs. MAP (Manifold Absolute Pressure) vs. RPM.
* **Target Air-Fuel Ratio (AFR) / Lambda Tables.**
* **Electronic Throttle Control (DBW):** Grip-to-throttle plate mapping.
* **Rev Limiters, Idle Speed vs. Engine Temp, and Injector Flow Constants.**

Commercial tuners spend years building or reverse-engineering **definition files** for every single Harley calibration strategy (there are hundreds across engine sizes: 88, 96, 103, 110, 107, 114, 117, 121, 131ci).

#### D. The Auto-Tune Feedback Algorithm
Tuning apps achieve their reputation on their "Auto-Tune" feature:
1. While riding, the dongle logs RPM, MAP/Throttle, and wideband/narrowband O2 sensor voltages or AFR.
2. The app compares the actual air-fuel ratio against the target table.
3. The app computes VE correction percentages:  
   $$\text{VE}_{\text{new}} = \text{VE}_{\text{old}} \times \left(\frac{\text{AFR}_{\text{actual}}}{\text{AFR}_{\text{target}}}\right)$$
4. The user clicks "Apply", and the app flashes the updated VE tables back into the ECM.

---

### 3. Key Risks & Challenges

1. **Bricking the ECM:** If Bluetooth disconnects, your app crashes, or the checksum calculation fails mid-flash, the ECM can enter an unbootable state. You must implement a rock-solid bootloader recovery mechanism (or require users to keep the bike on a battery tender during flashing).
2. **EPA & CARB Clean Air Act Enforcement:** 
   * Tuning apps that alter emissions controls, disable catalytic converter monitors, or richen AFR outside stoichiometric ranges are classified by the EPA as **illegal defeat devices**.
   * Harley-Davidson settled with the EPA for \$15 million over their Screamin' Eagle Pro Super Tuners, and aftermarket companies have faced heavy fines. 
   * If you release a commercial flash tuner, you will need either **CARB Executive Orders (EO)** for road legality or strictly market it for closed-course racing (which still carries legal scrutiny).
3. **2021+ Euro 5 Anti-Tampering:** Recent models run software that actively checks for calibration changes and logs "anti-tamper" flags, which can trigger check-engine lights, limp mode, and immediately void the factory warranty upon dealer scan.

---

### 4. Realistic Development Pathways (Feasibility Spectrum)

If you want to build a product in this space, here is how the difficulty tiers break down:

```
[Tier 1: Diagnostics & Gauges]  ──►  [Tier 2: Piggyback Controller]  ──►  [Tier 3: Direct ECM Flash Tuner]
  * Reads DTCs & Live PIDs              * Intercepts injector/O2 signals        * Full binary read/write
  * Low risk, fast MVP                  * No ECM security cracking needed       * Extreme reverse engineering
  * Standard OBD-II / UDS               * Custom hardware module required       * Seed-key & ROM definitions
```

#### Approach 1: Harley Diagnostic & Live Telemetry App (Lowest Effort, Quick Win)
* **What it does:** Reads live telemetry (head temp, gear, RPM, battery, fuel efficiency) and reads/clears Diagnostic Trouble Codes (DTCs).
* **Feasibility:** Very achievable in a few weeks using Flutter/React Native and an ELM327 / OBDLink BLE dongle using standard OBD PIDs (`Service 0x01`, `0x03`, `0x04`).

#### Approach 2: Piggyback Fuel Management App (Medium Effort, High Safety)
* **What it does:** Instead of flashing the stock Delphi ECM, you build an external hardware box (similar to a Dynojet Power Commander or Vance & Hines Fuelpak FP1) that plugs inline with the fuel injectors and O2 sensors.
* **Why it works:** You modify injection duration externally via your own BLE microcontroller. You never have to crack Harley's seed-key security or worry about bricking the factory ECM.

#### Approach 3: Full Custom ECM Flash Tuner (High Effort, Commercial Grade)
* **What it does:** Complete replacement for an FP4 or Dynojet PV4.
* **Feasibility:** Requires an experienced team with automotive reverse engineering (Ghidra, CAN bus sniffing), embedded C/C++ firmware development, and automotive powertrain calibration experience. Estimated 9–18 months of R&D on bench-top ECM test rigs before touching a real motorcycle.

---

### Suggested Next Steps if You Want to Explore Further

1. **Pick a Specific Target Model Year:** Start with **2014–2020 Touring / Softail (CAN-bus Delphi MT21M)**. They use standard 500 kbps CAN (much easier to develop hardware for than pre-2011 J1850), without the heavy anti-tampering encryption of the 2021+ Euro 5 models.
2. **Set up a Bench Rig:** Acquire a salvaged Harley Delphi ECM, wiring harness, 12V power supply, and a USB-CAN analyzer (such as a PEAK PCAN-USB or an ESP32 CAN board running SavvyCAN/Wireshark).
3. **Sniff Known Tuner Traffic:** Connect a commercial tuner (like a Power Vision or FP3) to the bench setup and log the CAN frames during an unlock and flash sequence to inspect the UDS service IDs and seed-key handshake.
