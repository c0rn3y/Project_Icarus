# Speed Build Program: Project Longshot + The Gauntlet

**Owner:** Kenneth \
**Status:** Design phase closed. Building toward Gauntlet V1 (see "Gauntlet V1: the minimum"); next step is component selection (contactor first). Powertrain specced. Test rig components sourced/ordered. Software stack decided. Physical enclosure designed. Open electrical decisions logged in A13. Airframe not yet started. \
**Document version:** 3.18 (version history at the end of the document)

---

This program has two halves that support each other:

- **The Gauntlet** is the ground system: a purpose-built, safety-engineered motor test rig that develops and validates every powertrain choice before it flies.
- **Project Longshot** is the aircraft: a fully 3D-printed high-speed FPV quad.

Everything below was reasoned from first principles, not copied from a reference build.

---
---

# PART A: THE GAUNTLET (the test rig / base station)

The Gauntlet is the ground system that develops Project Longshot. It is a permanent, flight-parity motor test bench where the motor, prop, **and battery configuration** are all swappable variables. Nothing earns a place on the aircraft until it has run the gauntlet: a punishing, fully-instrumented sequence of tests, measured and logged, under a controlled and safe test environment.

What it does, in five jobs:

- **Flight parity.** It drives motors through the same ESC (hardware and firmware) that flies, so bench data predicts **motor and ESC** behaviour (efficiency, current, heat, high-RPM behaviour) rather than approximating it. The Pico drives the ESC directly; flight-controller behaviour (Betaflight motor output shaping, blackbox) is deliberately not replicated on the bench.
- **Optimization.** Thrust, current, temperature, and efficiency (grams per watt) measured at real operating points, so choices are driven by data.
- **Comparison.** Head-to-head evaluation of different motors and battery topologies under identical, controlled conditions.
- **What the bench cannot do: rank speed props.** The rig measures **static** thrust with no airflow. A high-pitch speed prop (e.g. 7x15) is heavily stalled at zero airspeed, so it looks poor on the bench and excellent at 500+ km/h. Static g/W does not predict which prop is fastest. Props are chosen by pitch-speed calculation plus flight (blackbox) data; the bench characterises motors.
- **Prop proof and balance.** What the bench *does* do for props: every modified (cropped) prop is spun inside the containment **above its planned flight RPM** before it flies, and its vibration is checked in the load cell data (a direct balance check).
- **Documentation.** Automated throttle sweeps and logged runs that build the dataset.
- **Safety.** A contained test environment: hardware e-stop, fail-safe relay, pendant-link watchdog, and an arm interlock, so a transonic prop is developed without hurting anyone.

## Design basis (read this first)

The Gauntlet is a **hobby test rig**. It is not going through CE marking, SIL/PL assessment or formal certification. Standards (ISO 13850, IEC 60204-1, ISO 14119 and others) are referenced because they record failures someone already discovered the hard way, not because compliance is the goal.

**The filter every design decision goes through:**

> Can a plausible failure get someone hit by the prop or a projectile, expose someone to dangerous voltage, or start a fire?

If yes, it is a requirement. If no, it goes on the schematic checklist or into Tier 3 (see A16), not into the architecture.

**Hazard calibration for this rig:**
- **Projectiles and the spinning prop** are the dominant hazard. Containment, the guard, and not restarting unexpectedly matter most.
- **Fire** from a LiPo short or overloaded wiring is the second. Fusing at the source matters.
- **Shock from the battery side is a minor hazard:** 12S is 50.4V maximum, which is within the usual 60V DC extra-low-voltage limit for dry conditions. The battery-side hazards are sparks, burns, arc damage and fire, not electrocution.
- **Mains (230V)** is a real shock hazard, which is why A15 keeps the mains section small and uses certified parts.
- Much of A8, A13 and A14 goes beyond what a hobby rig strictly needs. It stays documented as the **target design and a record of the reasoning**; A16 sets the **build order**, so the first prototype is much lighter than the full document.

---

## Gauntlet V1: the minimum (build this first)

The rest of Part A describes the **target** rig. This section defines the first thing actually built. **The Gauntlet exists to build Longshot, not to become a commercial test stand.**

**Goal of V1:** a thrust / current / RPM curve for the **AMAX 2820 1000KV on 6S**, measured safely and repeatably.

**Hardware:** Tier 1 as defined in A16. Nothing from Tier 2 or 3.

**Software Tier 0 (matches hardware Tier 1):**
- Pico firmware: throttle, e-stop loop sensing, arm sequence, watchdog, output ceiling, precharge.
- Pendant firmware: encoder, buttons, TFT, RS-485 link.
- Pico streams measurements over USB serial.
- A small Python script on the Pi writes them to **CSV files**, one per run.
- **One plot** (thrust, current, RPM against throttle).
- **Not in V1:** Docker, Nginx, MQTT, TimescaleDB, FastAPI, Grafana, the live web page, session tokens and auth. The full stack (A4, A5) is right for later; it is not needed to produce the first curve. It comes after V1 has produced real data.

**V1 dataset (the minimum needed to make a motor decision):**
- Thrust
- RPM
- Current
- Voltage
- Motor temperature (PT1000)
- Ambient temperature and pressure, only for air-density correction

Thermal camera, AI camera, ESC telemetry beyond what the safety logic needs, and everything else wait.

**Milestones, in order:**
1. **Containment built** to full specification (A1, A13).
2. **Containment failure test:** a cheap, deliberately notched prop spun up on 6S inside the closed tunnel until it lets go, before any real testing. The containment is the biggest unknown in the whole design: electrical and software faults can be fixed and retested, a containment failure cannot. This is the one test that proves it.
3. **Stage 0 commissioning** (bench, no battery).
4. **Stage 1** (6S, prop off).
5. **Stage 2** (6S, prop on, stepped output ceiling).
6. **First curve** for the AMAX 2820 on 6S.

**Validation spin: keep it deliberately dumb.** Motor spins; current not insane; RPM roughly as expected; done. It exists to catch gross errors (wrong motor, wrong pole count), not to validate the powertrain. A simple check is one you can trust; a clever one becomes threshold tuning forever.

**Scope guard:**
- Anything not on this list waits until the first curve exists.
- The FMEA stopping rule stands (a failure ends the chain when it is self-revealing or caught by the proof test). A14 is analysis, not a to-do list.
- **Longshot airframe work runs in parallel.** Airframe design does not depend on the rig existing yet.

---

## A1. Physical enclosure

The Gauntlet lives in a dedicated **electrical box** mounted on the side of the containment tunnel.

### Outside face (operator panel)
- **Raspberry Pi Touch Display 2, 10.1" Portrait** (main dashboard, DSI ribbon to the Pi inside)
- **E-stop mushroom: LEB22-1-C 22mm 1NC latching, twist-reset** (prominent, right next to the screen, always in reach when watching the display)
- **GX16 panel-mount connector** (locking, for the wired pendant cable)

### Underside
- **Mains inlet: certified IEC C14 inlet module with double-pole switch and fusing suitable for IT networks** (see A15)
- **SHT31-D + BMP280 sensor bracket:** both ambient sensors on a plastic standoff in still air, shielded from prop wash and tunnel airflow

### Inside (DIN rail)
- **Mains compartment:** a separate covered section of the DIN rail holding only the inlet wiring, PE terminal and the PSU input. Finger-safe terminals, strain relief, mains physically segregated from all SELV/PELV wiring (A15)
- **Mean Well 24V DIN rail PSU** (mains to 24V DC bus; certified isolation, IT-network rated, built-in output overvoltage protection)
- **24V-to-12V DC-DC converter (contactor control rail)**, DIN-rail (Mean Well DDR class, 12V output): feeds the whole energy-path chain (e-stop loop, restart latch, contactor coil, precharge relay coil). See A8.
- **24V-to-5V DC-DC converter** with an overvoltage clamp / crowbar on its output (clean 5V rail for Pi, logic, and pendant power over the GX16 cable)
- **Raspberry Pi 5 (16GB)** on a DIN rail mount
- **Pico** (real-time controller / safety authority)
- **Custom PCBs** (safety circuit, relay driver, signal conditioning)
- **DAKEFPV H743 12S 120A stack, ESC board** (driven directly by the Pico; wired to the motor, fed through the contactor). The rig stack's FC board is not in the rig signal path (spare / optional).
- **DC contactor** (in the motor power path)
- **Terminal blocks** for all external wiring (load cell, PT1000, e-stop loop, contactor, pendant GX16, sensors). Nothing from outside hardwires directly into a PCB.

### Containment tunnel (the physical barrier)
- The tunnel is **safety containment**, not just an enclosure: it is what stops a shed blade or a part letting go. It is analysed like any other safety component (A14).
- **Access guard with interlock and guard locking** (A8): the tunnel's access opening (lid / door) has a solenoid guard-locking interlock switch: a positive-opening door contact in the NC e-stop loop and a power-to-release lock that holds it closed until the motor has run down and the bus is safe.
- **Designed against fragment energy** (A13): wall material and thickness chosen against the calculated energy of a blade fragment at maximum RPM, not by feel. Cable and air openings must not create a line-of-sight escape path.

### External (motor / battery side)
- **XT90 socket** where the active power module plugs in
- **Primary DC fusing lives in each power module (A10)**, as close to the batteries as practical, before the module's output lead. The contactor is a switch, not overcurrent protection; two 6S 160C packs can deliver thousands of amps into a short. Fusing at the source protects the packs, the module, the output lead and the XT90 plug, which a fuse inside the enclosure would not.
- **Optional secondary rig fuse** directly after the enclosure XT90 inlet, before the contactor and precharge branch.
- High current path: batteries -> **module fuse(s)** -> power module XT90 -> (optional rig fuse) -> contactor inside box -> ESC -> motor. Batteries and power module always stay outside.

---

## A2. Controller architecture

The base ESP32 has been removed. The pendant is wired, so no wireless link layer is needed.

- **ESP32-S3 = pendant only.** Drives the encoder, TFT screen, and function controls. Talks to the Pico over RS-485 down the GX16 cable (framed, CRC-checked packets, see A3). The pendant e-stop mushroom is electrically independent of the ESP32-S3: it is a hardware NC contact wired directly into the series safety loop via two dedicated GX16 pins. The MCU does not see, mediate, or control it.
- **Pico = real-time controller + safety authority.** Owns throttle generation (PIO bidirectional DShot, driving the ESC directly), the local RPM source (eRPM returned by bidirectional DShot), throttle-source arbitration, sweep execution, the pendant-link and Pi-link watchdogs, e-stop loop sensing, relay-enable logic, and the arm interlock. Takes high-level requests from the Pi and commands from the pendant ESP32-S3, but can refuse or cut throttle regardless. The hardware e-stop NC loop is electrically independent of the Pico: opening the loop removes coil power regardless of what the Pico is doing. Independent of the Pi's boot/crash state.
- **Raspberry Pi 5 (16GB) = the main brain.** Data, database, live dashboard, MQTT broker, AP, analysis, sweep configuration (execution happens on the Pico). **Outside the hard-safety and real-time actuator paths; participates only in supervisory control** (sweep configuration, start/abort requests, which the Pico validates). Linux is not real-time, and speed does not fix determinism.
  - **Storage:** M.2 NVMe SSD via the official PCIe M.2 HAT+ (low profile, clears the DSI ribbon). Database lives here, not on the SD card.
  - **Cooling:** separate active cooler. Cooler first, then HAT+ on taller standoffs (officially supported combo).
  - **Power:** fed from the 5V DC-DC rail inside the electrical box via the Pi's USB-C input.
  - **Future direction (justifies 16GB):** on-device thermal anomaly detection on camera frames. Not in v1 scope.

---

## A3. Pendant (wired)

A **3m coiled cable** connects the pendant to the electrical box via a **GX16 7-pin connector**. Coiled so it does not drag; natural rest length ~1m, stretches to 3m. No battery, no wireless stack, no link-quality management.

**GX16 pin assignment (7-pin connector, all pins used):**

| Pin | Signal |
|---|---|
| 1 | +5V (from DC-DC, feeds ESP32-S3 and TFT) |
| 2 | GND |
| 3 | RS-485 A (half-duplex data) |
| 4 | RS-485 B (half-duplex data) |
| 5 | E-stop NC loop out |
| 6 | E-stop NC loop return |
| 7 | Shield / drain |

**Pendant data link: RS-485 plus framed packets.** 3m of single-ended UART beside a 12S ESC and motor is asking for corrupted bytes. The pendant-link watchdog makes a dead link safe, but it cannot detect a corrupted packet that happens to look like a valid command. Two layers, not one:
- **RS-485 (differential, half-duplex)** on pins 3+4, via a transceiver at each end (MAX3485 / SN65HVD class, 3.3V). Makes noise-induced corruption unlikely.
- **Framed packets** catch what gets through anyway: start marker, message type, sequence number, heartbeat counter, payload, CRC. Bad CRC, wrong sequence, or unknown type = packet discarded (and counted; a rising error rate is a fault).
- **Bounds checking on content:** encoder delta per packet is capped, so even a valid-looking packet cannot jump the throttle. Arm state changes still require the fresh SAFE-to-ARM edge rule.

Pins 5+6 carry the pendant mushroom NC contact in the hardwired series safety loop. Two conductors, one contact, no MCU in the path.

**Pendant-link watchdog:** the Pico monitors the RS-485 link for a valid pendant heartbeat. No valid message within the timeout = throttle cut. This covers pendant link loss while the hardware safety loop remains intact (pendant crash, ESP32-S3 lockup, software fault on the pendant). Physical cable unplug is handled first and faster by the NC loop: unplugging the GX16 immediately breaks pins 5+6, drops the contactor, and stops the motor before any pendant-link watchdog timeout fires. The two mechanisms are independent and layered -- cable unplug triggers both in sequence; the pendant-link watchdog is not relied upon for that case.

**Throttle-source arbitration:** the Pico holds exactly one active throttle source at a time: pendant manual or automated sweep. Any pendant throttle or function input during a sweep aborts the sweep. When a sweep ends or aborts, throttle goes to zero, never to wherever the pendant knob happens to be. The rotary encoder is relative (it reports deltas, not an absolute position), so the Pico owns the throttle value and handing control back to the pendant causes no jump. A potentiometer would have had exactly that problem.

**Pendant controls:**
- **Throttle:** Adafruit seesaw I2C rotary encoder (clicky/detented, NeoPixel for status). Big vintage-amp-style knob. Support the knob's weight with a faceplate bearing so the encoder shaft is not side-loaded.
- **Arming:** missile-cover toggle. Maintained switch, so the Pico acts on a fresh SAFE-to-ARM transition, never on the level alone. After startup or any fault, the toggle must first be seen in SAFE, then moved to ARM. A toggle already in ARM at boot or when a fault clears is ignored.
- **Function buttons:** Tare, Log, Auto-sweep, Mode, Set.
- **E-stop:** side-mounted (frees the front face; ~50mm depth behind panel; recess/guard against accidental trips).
- **Display:** 2.4" ILI9341 SPI TFT (240x320). Backlight on GPIO for dimming.

---

## A4. Comms architecture (two planes)

- **Data plane = MQTT (via the Pi 5 broker, Mosquitto container).** Telemetry (thrust, RPM, current, temps), logging, dashboard, phone view. **MQTT is the data layer only. Nothing that can change motor output travels over MQTT.** Logging commands are fine. A command that begins or modifies an automated sweep goes Pi to Pico over the direct USB/UART control path, not via MQTT.
- **Sweep execution lives on the Pico.** The Pi uploads sweep parameters (ramp rate, dwell per step, ceiling). The Pico checks them against the profile latched at arm time, then runs the sweep deterministically. The Pi sends start and abort only. Linux stays out of the real-time control path even during automated runs.
- **No stale commands:** Pi commands are acted on only if they arrive while the rig is armed and fault-free. Anything received before arming, during a fault, or while the Pico was rebooting is discarded, never queued and executed later.
- **Arm-session number:** every **accepted arm request** creates a new arm-attempt number on the Pico (it becomes the active session number only if the arm sequence succeeds; on failure the number and the profile snapshot are destroyed). It is a fresh random nonce (at least 64 bits, from the RP2040 hardware entropy source) or a boot-epoch plus arm counter. A plain RAM counter is not acceptable, because it resets on reboot and can reuse a number across exactly the boundary this protects. The arm-session number is bound to the profile snapshot taken for that arm attempt (A9), so Pi commands always refer to the exact profile they were issued against. The Pico reports it to the Pi. Every sweep configuration and start command must carry the current number. A command carrying a number from a previous arm session is rejected even if it arrives while the rig is armed, which covers a command delayed in a USB/serial buffer across a disarm and re-arm.
- **Pi-link watchdog:** the Pico monitors a heartbeat from the Pi. If the heartbeat is lost during a sweep, the Pico aborts the sweep and sets throttle to zero. The operator has lost the display and database logging has stopped, so the run is not worth continuing. Manual pendant runs are unaffected by Pi loss (per the invariant below).
- **Control / safety plane = direct and deterministic. Never MQTT.**
  - Pendant ESP32-S3 to Pico: RS-485 over the GX16 cable, framed and CRC-checked (A3).
  - Pi to Pico: UART over USB (`/dev/gauntlet-pico`, fixed by udev rule keyed to the Pico's USB serial number). Carries sweep parameters and start/abort commands only; the Pi never streams throttle setpoints. Heartbeat monitored by the Pico's Pi-link watchdog.
  - Pico to ESC: bidirectional DShot (PIO). Throttle out, eRPM back on every frame, so the Pico has a deterministic local RPM source with no dependency on an FC or the Pi.
  - ESC to Pico: ESC serial telemetry wire (current, voltage, ESC temperature) into a Pico UART.
  - E-stop: hardware NC loop hard-inhibits both the main contactor and the precharge path. No protocol, no software.
- **The invariant:** if MQTT, the Pi, or the Docker stack dies, the dashboard goes dark but the rig still controls and stops safely. The safety plane depends on none of it.
- **Why MQTT is barred from safety:** non-deterministic latency, TCP/WiFi dependency, eventual-delivery QoS. Useless for an e-stop.
- **udev rules:** fixed symlinks by USB serial number. Always `/dev/gauntlet-pico`, never `/dev/ttyUSB0`.

---

## A5. Software stack (Pi 5, Docker Compose)

The entire Pi software stack runs in containers. `docker compose up` brings everything up on boot.

| Container | Job |
|---|---|
| Nginx | Reverse proxy, single entry point for all HTTP/WS traffic |
| Mosquitto | MQTT broker (data plane) |
| TimescaleDB | Single database: run metadata (relational tables) + sensor time-series (hypertables). One engine, one backup, proper SQL. |
| FastAPI | Business logic: MQTT-to-DB bridge, REST API, WebSocket for live data push, sweep configuration and start/abort requests to the Pico (execution happens on the Pico) |
| Grafana | Post-run analysis and comparison. Reads TimescaleDB via the native Postgres data source. |

**Internal Docker network:** containers talk internally. Only Nginx is exposed externally.

**URL layout (all via Nginx):**
- `/` - live run page
- `/grafana` - post-run analysis
- `/api/...` - REST API
- `/api/docs` - auto-generated interactive API docs (FastAPI/Swagger)

### TimescaleDB data model

- **runs table (relational):** run ID, motor profile, prop, battery profile, date, notes, valid flag, operator.
- **sensor_data hypertable (time-series):** timestamp, run ID (foreign key), thrust, RPM, current, voltage, motor temp, ESC temp, ambient temp, ambient pressure, throttle position.
- Derived metrics (power, efficiency, air density, temp rise over ambient) computed at query time or via continuous aggregates.
- "Give me the efficiency curve for run 47" = SQL join on run ID. One database, no bridging two engines.

### FastAPI service

- Subscribes to MQTT topics and writes sensor data to TimescaleDB in real time.
- REST endpoints for run management, motor/prop/battery profiles, calibration, sweep configuration.
- WebSocket endpoint pushes live sensor data to connected browsers.
- Calls system scripts for network mode switching and container restarts.
- **Access control:** the rule is simple -- reads are remote, mutations require local authority.
  - **Read operations** (GET -- live data, run history, status): open to any client on the AP. Phone/tablet view is served this way.
  - **Any state-changing operation** (POST, PUT, PATCH, DELETE -- start/stop, sweep config, profiles, calibration, network mode): requires a valid session token issued to the touchscreen at startup. Encoded as HTTP verb is a convenience; the real rule is "does this change state." A phone cannot perform these calls even if it knows the URL.
  - Session token is issued once at boot to the touchscreen session. Not a fixed IP -- a phone that takes the touchscreen's IP address still has no token. Bootstrap mechanism (how only the physically local touchscreen obtains the token) is an open implementation decision; see A13.

### Live run page

Built with **HTMX + Tailwind CSS + DaisyUI**. No build pipeline, no node_modules on the Pi. A single HTML page served by Nginx, updated in real time via FastAPI WebSocket.

**What it shows:**
- Live values: thrust (g), RPM, current (A), voltage (V), power (W), efficiency (g/W), motor temp, ambient temp
- Throttle position (mirror of the pendant)
- Run state: armed / running / stopped
- Safety chain status: relay state, e-stop loop state, latched faults, pendant-link and Pi-link watchdog status
- Current run log feed

**Responsive layout:** portrait touchscreen and phone are the primary views (stacked panels, big readable numbers). Laptop/desktop gets a wider layout via Tailwind breakpoint prefixes. One page, no separate mobile version.

**Access split:**
- **Touchscreen (on the electrical box):** local supervisory controller. May configure sweeps, and start or stop a sweep only after the Pico has accepted physical arming from the pendant and all interlocks are satisfied. May also tare, start/stop log, request a soft stop, and change settings (settings that affect the arm interlock are refused while armed). Cannot arm the rig, command manual throttle, bypass the Pico, or override either e-stop. The touchscreen cannot turn an unarmed rig into an armed one; it can only ask an armed Pico to run a defined operation.
- **Phone/tablet (on the AP):** view only. Live data, run status, no control. A phone has no hardware e-stop, so it gets no control authority.
- **Rule: control follows physical proximity to the hardware e-stop.**

### Grafana

Post-run analysis only. Not visible during active testing. Used for efficiency curves, motor comparison, thermal slope analysis, run history. Served at `/grafana` through Nginx.

### Settings page

Part of the FastAPI/HTMX app. Manages rig configuration:

- **Network:** field mode (AP + DHCP via hostapd + dnsmasq, isolated) / home mode (joins homelab, router handles DHCP). Toggle with confirmation dialog and auto-revert timeout (rolls back after 60s if connectivity is lost, same pattern as router firmware updates).
- **Profiles:** motor library (name, KV, stator size, **pole count** (needed to convert eRPM to mechanical RPM), notes), prop library, battery profiles. Tags every run in TimescaleDB automatically and feeds the arm interlock. Locked while armed (see A9).
- **Calibration:** load cell tare and calibration factor, PT1000 offset.
- **Sweep profiles:** throttle ramp rate, dwell time per step, max throttle ceiling.
- **Runtime limits (per motor + prop + battery combination, latched at arm):** max mechanical RPM (a property of the motor + prop combination, not the motor alone), max current, max ESC temperature, max motor temperature, minimum battery voltage. The throttle ceiling limits what is *asked for*; these limit what the hardware is *actually doing*. Enforced by the Pico on every run, manual or sweep, each mapped to a stop class (A8).
- **System:** Docker container status (up/down/restart each individually), NVMe disk usage, Pi CPU temp and load. Restart a misbehaving container from the touchscreen without a keyboard.

---

## A6. Raspberry Pi networking

- **Field mode (default):** Pi runs its own isolated AP (hostapd) and DHCP server (dnsmasq). Password-protected, closed. Cannot auto-associate with any external network.
- **Home mode:** AP stays on. Pi also joins the local homelab network. Home router handles DHCP for that interface.
- Mode switch via the settings page. Auto-revert if connectivity is lost after switching.
- **The Pi never joins a corporate or restricted network.** USB-C power only; no PoE, no ethernet to external networks. PoE was designed out to remove that route entirely.
- Phones and tablets connect to the Pi's AP for the live dashboard from anywhere on site.

---

## A7. Instrumentation

- **Thrust:** 10kg single-point Wheatstone-bridge load cell (bare 4-wire: red=E+, black=E-, white=O+, green=O-) into an **Olimex ADS1220** (24-bit, SPI, 128x PGA). Mount so thrust is axial, body clamped, wire-exit end unloaded, over-travel stop.
- **Motor temp: PT1000 RTD** (-70 to +400C, x3 bought), read by the **ADS1220** (reused, not a new MAX31865). Spring-loaded in the rig's motor-mount fixture against the stator base. Thermal paste, not epoxy. Nothing bonded to the motor. Repeatable contact, motors stay clean and reusable.
  - ADS1220 input budget: load cell + one PT1000 fills all four inputs. Spare PT1000s are future points.
- **Ambient reference:** SHT31-D + BMP280 mounted together on the underside of the electrical box in still air outside the tunnel.
  - **SHT31-D** (I2C 0x44/0x45, +-0.2C): accurate temperature reference. Used for temp-rise-over-ambient and as the temperature input to the air density calculation.
  - **BMP280** (Joy-It SEN-KY052, I2C 0x76/0x77): barometric pressure for air density. No address clash with encoder (0x36) or SHT31-D.
  - Air density = f(pressure from BMP280, temperature from SHT31-D). Each sensor does what it is best at.
- **Thermal imaging:** over the motor, for hotspot location and motor-to-motor comparison. Complements the PT1000 (surface vs. stator). Lives on the Pi, never in the safety path.
  - **Preferred: FLIR Lepton** (160x120 radiometric, PureThermal breakout, USB UVC). Export-controlled: source through proper channels, confirm rules before it crosses a border.
  - **Fallback: MLX90640** (32x24, I2C, no export baggage).
- **Visual + AI: Raspberry Pi AI Camera (Sony IMX500).** On-sensor inference, no Pi CPU load. Visual run record + on-sensor anomaly detection (prop shed, smoke, a part letting go). Never in the safety path. Supersedes WROVER-CAM for run recording.
- **Battery-side voltage sensor (Vbattery):** a voltage divider (or isolated voltage sensor) measuring pack voltage upstream of the contactor, read directly by the Pico. The FC/ESC sits downstream of the contactor and cannot report pack voltage before precharge, so it is not a valid source for the arm interlock. Provides the reference for the precharge threshold and the battery-voltage check in A9.
- **Independent standstill / RPM sensor (logic-powered) -- conditional:** an optical or Hall-effect sensor reading the motor bell, powered from the logic side, not from the DC bus. Bidirectional DShot eRPM comes from the bus-powered ESC, so after a hard disconnect it disappears while the rotor may still be turning. Whether this sensor is a **safety requirement** depends on the measured run-down time (A8 guard locking): if the worst-case run-down is short and repeatable, the guard uses a validated time delay and this sensor is an optional cross-check on eRPM (useful for catching a wrong pole count continuously). If the run-down is long, the guard unlocks on positive standstill evidence and this sensor becomes required. Either way, if fitted, a dead sensor must never read as stationary.
- **Bus-side voltage sensor (Vbus):** a voltage divider (or isolated voltage sensor) measuring the DC bus on the ESC side of the contactor. Required for: (1) precharge validation -- confirming Vbus rises correctly and reaches ~95% of Vbattery before the main contactor closes; (2) post-disconnect discharge confirmation -- after the contactor opens, Vbus should decay on the bleed resistor curve; (3) welded-contactor detection -- the fault signal is Vbus continuing to track Vbattery during the expected bleed decay, rather than decaying. Note: Vbus being high immediately after the contactor opens is normal (the cap bank is still charged); the fault is Vbus not decaying at all, indicating the contactor has not actually opened. After an opening during a run, Vbus will not follow the simple bleed curve while the motor is still spinning down (back-EMF and ESC behaviour), so the Vbus-based weld decision waits until RPM is low or a defined settling period has passed. Gross faults are still flagged immediately, and the contactor auxiliary contact (below) gives immediate position feedback in the meantime. Also detects a stuck-on precharge path: Vbus rising while nothing is commanded is a fault. Read by the Pico. Feeds both the arm interlock and the live dashboard.
- **Contactor position feedback (required):** the main contactor must have an auxiliary contact that mechanically mirrors the main contacts, read by the Pico as a direct contactor-state signal. It is the only prompt way to prove the contactor physically closed during arming (Vbus cannot: the precharge path holds the bus at battery voltage whether or not the contactor closed), and it detects a weld immediately without waiting for spin-down. Read back as a complementary pair (NO + NC) where the contactor provides both: both reading the same state = feedback fault.
- **Divider input protection:** both voltage dividers sit on a ~50V bus feeding a 3.3V ADC. A single divider component failure (lower resistor open, upper resistor shorted) must not turn a bad reading into a dead Pico: series current-limit resistance plus a clamp at the ADC input, and upper resistance split across two resistors so one short does not expose the input. Isolated sensing (isolated amplifier) is the alternative. A lower-resistor-open failure reads full scale permanently; the plausibility check "Vbus ~0 before precharge" catches it, so that check runs before **every** precharge.
  - **Measurement range:** the dividers scale the **whole intended range**, the 12S maximum of 50.4V plus margin (nominally 0 to 60V), into the ADC's valid window (roughly 0 to 3.0V against the Pico's 3.3V reference). The clamp is **inactive in normal operation** and only conducts under a fault or excessive input voltage. A divider scaled for 6S would saturate the day the rig moves to 12S.
- **Vbattery and Vbus together** make precharge a self-contained comparison on the Pico, with no dependency on the FC or the Pi. ADC choice (Pico onboard ADC vs a dedicated ADC) to be decided with the divider design.
- **Dedicated current sensor (Pico-read):** Hall-effect sensor, or a high-side shunt with a suitable amplifier / isolation, in the battery path (no low-side shunt: with a shared ground, other return paths partly bypass it and motor-current ground drop is injected into the logic reference), read directly by the Pico at a fast, known rate. Two jobs: (1) the **overcurrent SAFETY TRIP** needs a fast, deterministic measurement, which ESC serial telemetry (slow, polled, often an estimate) cannot guarantee; (2) **efficiency (g/W)** needs true battery current, not the ESC's estimate. ESC-reported current is kept as a cross-check.
- **Electrical (RPM, current, voltage, ESC temp):** RPM from bidirectional DShot eRPM, read by the Pico every frame. Battery current from the dedicated sensor. ESC-reported current, voltage and ESC temperature from the ESC serial telemetry wire into a Pico UART. No FC in the loop, so no Betaflight blackbox on the bench; the Pico SD log and TimescaleDB replace it.
- **Logging:** Pico local SD (always on, independent of Pi) + TimescaleDB on the NVMe (fed over MQTT via FastAPI).

---

## A8. Safety architecture (defence in depth, critical)

- **Hardware e-stop = the true authority.** An NC series loop runs through both mushroom contacts in the contactor-coil enable path. An intact loop is a necessary condition for the contactor to energise, but does not energise it by itself: the Pico must also permit the relay driver after the arm interlock has passed. Opening either mushroom, cutting a wire, unplugging the pendant cable, or losing control power physically removes coil power and commands a healthy contactor open regardless of software state. No software can keep a healthy contactor energised once the loop is broken. (A contactor that has physically welded is a separate fault, covered below.)
- **The NC loop inhibits both energy paths.** There are two routes from battery to ESC bus: the main contactor and the precharge path (A9). The NC loop hard-inhibits both. The precharge switch (a force-guided safety relay, A9) is powered from the same loop-fed supply as the contactor coil: its coil sits in the loop. The Pico may request precharge, but an open loop makes precharge electrically impossible, not just software-disabled. Without this, a stuck-on or misdriven precharge switch would keep recharging the ESC capacitor bank through the resistor after an e-stop, defeating the bus bleed. Invariant: **opening the e-stop loop removes every commanded energy path, independently of software.**
- **Failed-closed switching elements are a separate hardware fault class.** The loop removes coil and gate drive; it cannot open a contact that has welded or a MOSFET that has failed short. These are detected and contained, not prevented:
  - **Welded / failed-short precharge switch:** current is limited by the precharge resistor, but the failure is potentially sustained (up to ~25 W continuously into a shorted bus until disconnected, see A13 branch protection). Detected by Vbus rising or holding when no precharge is commanded. Fault latched, arming refused.
  - **Welded main contactor (the worse case):** the e-stop opens the loop but the contactor stays closed. The motor is still stopped by the signal-cut layer: the Pico senses the open loop and immediately sets throttle to zero, and the ESC stops driving. Detected by the auxiliary contact (immediately, if fitted) and by Vbus not decaying on the bleed curve after spin-down. Fault latched, arming refused until the contactor is inspected or replaced. The result is "motor stops via signal cut, rig locked out", not "motor keeps running". **Assumption:** this holds for a single welded-contactor fault with the Pico and ESC healthy. It is not a guarantee against simultaneous failures (welded contactor plus a frozen Pico or a faulty ESC); for those, the remaining protection is the power-module fusing and physically disconnecting the power module.
- **The NC series loop:**
  - **Panel mushroom (LEB22-1-C, 22mm 1NC latching):** NC contact block wired directly into the loop via terminal blocks inside the electrical box. Primary e-stop, physically next to the touchscreen.
  - **Pendant mushroom (LEB22-1-C, 22mm 1NC latching):** NC contact block on the pendant PCB. Its two terminals connect to two dedicated pins of the GX16 cable, which carries them back into the same loop inside the box. Mechanism is identical to the panel mushroom -- purely hardware, no MCU, no software, just a switch in a wire. The GX16 is only the cable that carries it.
  - **Cable pull = free e-stop:** unplugging the GX16 pendant cable physically breaks the NC loop. No extra logic needed.
  - Both contacts are in series: either one opening stops the motor.
  - **Guard switch (containment access):** a positive-opening safety interlock contact on the tunnel access, **in series with the same NC loop**. It must be a **dry NC contact** that sits in a series loop; many coded non-contact switches instead provide powered safety outputs that expect a safety relay, which would force a different topology. Recommended part class: a **solenoid guard-locking interlock switch** (tongue actuator) providing, in one certified device, the positive-opening door contact, a separate lock-monitoring contact, and the lock itself. Opening the guard acts exactly like an e-stop, in hardware: coil and precharge drive removed, restart latch drops, full re-arm required. A plain microswitch is not acceptable (too easy to defeat or to fail closed). **Containment closed is therefore an arm prerequisite by construction**: the loop cannot be intact with the guard open.
- **Guard locking.** Where run-down time can exceed the time it takes to open the lid and reach in, machinery practice (ISO 14119) calls for guard **locking**, not just interlocking. The lock also stops the lid opening while the rig is armed or the bus is live, which is worth having regardless of run-down time.
  - **Rough pre-test estimate only (replace with measured data):** with a prop fitted, aerodynamic drag is large: roughly 175 J stored at 48k RPM, about 0.2 s to half speed (three quarters of the energy gone) and about 2 s to 10% speed, then bearing friction and iron losses finish it. With the ESC still powered, active braking stops it almost instantly. **The worst case is a shed prop**: the bare bell has almost no drag and can freewheel for seconds, and that is exactly when someone wants to open the lid. 
  - **Loss of control power must leave the guard mechanically locked** (requirement). A power-to-lock guard would unlock on 24V loss, which is a credible event while the prop is coasting. Implementation: spring-lock, power-to-release. Nobody can be inside the tunnel, so there is no entrapment case; a key- or tool-operated manual escape release covers service, and its state is monitored.
  - **An e-stop does not unlock the guard.** It removes motor energy; the guard stays locked until the mechanical hazard is gone.
  - **Unlock method is decided by measurement.** Bench test: worst-case run-down with the ESC unpowered (hard disconnect), both with a prop fitted and with a bare bell. **Starting speed for the bare-bell test is the no-load speed, KV x maximum pack voltage**, which is a physical upper bound for an unloaded motor (about 50.4k RPM for the AMAX 1000KV at 50.4V, about 53k RPM for the V3115 1050KV). Use the fastest motor in the library. Measure with a temporary independent optical tachometer (eRPM disappears when the ESC loses bus power). **Re-validate whenever the rotating assembly changes** (motor, prop, bell, adapter), not only when the motor library changes. Then:
    - **Short, repeatable run-down:** unlock after a **validated time delay** (measured worst case plus margin) **and** Vbus valid and below the safe threshold. ISO 14119 explicitly allows time-delay guard unlocking on this basis.
    - **Long run-down:** unlock only on positive standstill evidence from the logic-powered standstill sensor (A7) **and** a safe Vbus.
  - **A designed delay is not a telemetry timeout.** A validated run-down delay is a legitimate unlock condition. Unlocking *because data went missing* is not: missing or implausible Vbus (or standstill data, if that method is used) = DO NOT UNLOCK. For the guard, the safe state is **locked**. Recovery if data is lost: the manual escape release, after physically verifying the rotor has stopped.
  - **Unlocking is deliberate, not automatic.** When the run-down condition and a safe Vbus are satisfied, the Pico enters UNLOCK PERMITTED. The operator then presses a **spring-return UNLOCK button on the electrical box**, wired **in series with the release driver**. A shorted release driver cannot unlock the lid on its own, and a stuck button cannot unlock it without Pico permission. The button lives on the box, so no extra pendant conductor is needed.
  - **Unexpected unlock while armed = SAFETY TRIP.** If the lock-monitoring contact goes from LOCKED to UNLOCKED while armed, a physical barrier has disappeared: throttle to zero, drop the master permit and contactor, de-energise the release coil so the spring lock tries to re-engage, latch the fault. An orderly spin-down is the wrong priority in that state.
  - **Closed and locked are two separate verified conditions.** The door contact proves closed (hardware, NC loop). The lock-monitoring contact proves the bolt is physically engaged, read by the Pico. It reads bolt position, not solenoid state.
- **E-stop reset never restarts the rig.** The Pico senses the NC loop state (opto-isolated or divider sense input downstream of both mushrooms). When the loop opens, the Pico immediately drops relay-enable and precharge-enable, sets throttle to zero, aborts any sweep, and latches a fault. Twisting the mushroom to reset closes the loop but does nothing else: relay-enable stays low. Returning to operation requires a deliberate full re-arm from the pendant (fresh SAFE-to-ARM transition, see A3), including the complete precharge sequence (A9), because the bus will have bled down. Without this, resetting the mushroom would re-energise the coil and slam the contactor onto a discharged bus with no precharge, no re-arm, and whatever throttle the Pico was last outputting. Principle as in ISO 13850 / IEC 60204-1: resetting an e-stop must not by itself cause a restart.
- **Hardware anti-restart (mandatory).** Pico loop sensing alone is not enough: a Pico frozen with its enable outputs high would re-energise the coil the moment the mushroom is reset. A hardware seal-in therefore sits in the energy-path enable chain. Once the loop opens, the seal-in drops and cannot re-latch until it receives a fresh start pulse. It covers both the main contactor and the precharge path. The start input is **edge-coupled** (capacitor-coupled or one-shot), so a level stuck high, from a frozen Pico or anything else, cannot re-latch it. Restart requires the seal-in to be re-latched by a fresh edge **and** the Pico to permit. **The latch is independent of the main contactor:** precharge runs for ~3 tau with the main contactor deliberately open, so the contactor's auxiliary contact has nothing to hold in during that phase. The latch is a small pilot relay with its own holding contact, or a discrete hardware latch. The contactor auxiliary contact is used for position feedback (A7), not for the restart latch. **Power-up safe:** the latch must power up unlatched. Supply ramp, brownout recovery, Pico boot, and restoration of control power must not create a valid restart pulse (power-on-reset gating on the start input, so edges during the supply ramp are ignored). Implementation choice (pilot relay vs discrete latch) at PCB stage; the requirements are architectural.
- **Every energy-path enable is fail-low in hardware.** Both relay-enable (main contactor) and precharge-enable have external pull-downs, so Pico reset, boot, unpowered GPIO, or a disconnected Pico leave both paths off. Any future output that can enable an energy path gets the same treatment.
- **Pendant-link watchdog:** no valid pendant packet within timeout = throttle cut. Covers pendant firmware crash or pendant power loss while the hardware NC loop remains intact. Cable unplug is handled faster and independently by the NC loop (pins 5+6 break immediately); the pendant-link watchdog does not need to cover that case.
- **The relay / contactor (final target):** 60-100A DC-rated, 60V or higher DC break-under-load, normally-open, coil on the 12V contactor control rail, mirror auxiliary contact (A7). **DC break rating must cover the SAFETY TRIP current at 12S voltage**, not just normal continuous bench current. DC contactor family (EV / solar / automotive).
- **V1 contactor (chosen): Hongfa HFE82V-60B/750-12-HL5.** Sealed EV contactor, 60A continuous, 750V rated, **12V coil**, lead wire + screw terminals, 64 x 33 x 53mm. From the HFE82V-60B datasheet: max break 600A at 450V DC (1 op); 600A for 0.6s; **no polarity on load or coil side**; release time 10ms or less. **No auxiliary contact**, so weld detection relies on Vbus after spin-down until the final contactor is fitted. 60A is ample for V1 / Tier 1 on 6S; it is marginal for full 12S static bench current, so expect to replace it with a 100A class part with aux before full 12S power (Tier 2 / Tier 3). Buy from a seller whose photos show the genuine Hongfa label (HF logo and manufacturer name) and check the label on arrival.
- **12V contactor control rail.** A 24V-to-12V DC-DC converter supplies the energy-path chain: e-stop NC loop, restart latch / master permit, contactor coil, precharge relay coil (12V automotive / pilot relays are cheap and plentiful). **The NC loop breaks the 12V side, directly in series with the coils, never the converter's input**: breaking the 24V input would leave the converter's output capacitors holding the coil in for a moment and slow drop-out. The 24V bus remains for the guard lock (Tier 2, 24V parts) and the 5V logic supply.
- **Protection hierarchy:**

```text
normal run
   | controlled stop (spin down, then open)
safety trip current
   | contactor must interrupt this safely at 12S
major short / beyond contactor break rating
   | module fuse clears the fault
   | contactor and wiring must survive until it does
```

  This is one **coordinated protection system**, not two independent parts: the contactor's breaking and short-time withstand ratings must be checked against the fuse's time-current / I2t curve (A13). A fuse that opens only after the contactor has tried to break ten times its rating is not the hierarchy described here.

- **Precharge-complete gate (hardware).** A single failed-short contactor driver MOSFET must not be able to bypass precharge: with the restart latch set before precharge, a shorted driver would energise the contactor onto an uncharged bus. So the main-contactor enable chain has a **second series switching element** driven by a hardware comparator (LM339 class). The contactor physically cannot energise until Vbus is within the precharge threshold of Vbattery, regardless of driver failure or a firmware bug commanding it early.
  - **Separate dividers:** the gate comparator has its own Vbus and Vbattery divider networks, not the ones the Pico reads. One divider fault must not fool both the software check and the hardware gate.
  - **Minimum Vbattery condition:** the gate permits only when Vbattery is above a minimum **and** Vbus is within threshold of it. Without this, "Vbus >= 95% of Vbattery" is true at 0 >= 0 and the gate reads permissive with no battery connected.
  - **Readback senses the switched coil path, not the comparator command.** The Pico reads the state *after* the gate's own series switching element (the effective coil node), so a series switch that has failed short is visible even while the comparator says "block". Reading only the comparator output would be commanded-not-confirmed. Used for the proof test (below) and sequence verification.
  - **Hysteresis and minimum dwell:** the comparator has defined hysteresis around the precharge-complete threshold, and the permit must hold for a minimum dwell time before it is valid. Bus ripple and comparator noise near the threshold must not make the permit chatter in series with a contactor coil. Values set at schematic stage.
  - **The gate only delays closure; it cannot remove permission afterwards.** Once closed, Vbus = Vbattery and the gate stays satisfied. A contactor driver that fails short *after* closure would hold the contactor in against a software stop. That is the job of the master safety permit (next).
- **Master safety permit (the restart latch).** The restart latch already sits in series with both energy paths. It is also the master permit: in addition to dropping when the e-stop loop opens, it has a **Pico-controlled release input that works in the safe direction** (the Pico must actively hold it; no drive = release). The Pico can always drop the latch, and only a fresh edge-coupled start can set it again. Consequences:
  - Main contactor driver fails short: the Pico releases the latch; coil power is removed. Needs two failures (driver short **and** latch welded) to lose software-controlled isolation.
  - Precharge driver fails short: after a normal stop the precharge relay would otherwise stay energised, keeping the bus tied to the battery through the resistor. Releasing the latch removes its coil power too.
  - Every stop (controlled stop and safety trip) ends with the latch released, not just the drivers switched off.
- **Precharge driver failing short** is not benign: before arming it is resistor-limited and detected by Vbus rising before precharge is commanded; after a stop it would keep the bus live through the resistor. Detected by force-guided relay feedback; contained by latch release.
- **Relay driver:** logic-level MOSFET switched by the Pico, with appropriate coil transient suppression. UF4007 available for prototyping; final suppression topology (plain diode, diode+Zener, or TVS) depends on the selected contactor's drop-out time requirements -- see A13. **Hardware undervoltage lockout (LM339N/LM2901 comparator):** watches the **12V contactor control rail** (the supply the coil actually depends on) and switches off both energy-path enables cleanly below a defined threshold, with hysteresis, independent of the MCU. Its job is to prevent **contactor chatter**: during a brownout the coil can sit between its drop-out and pull-in voltages and bounce the contacts under load. Threshold set above the contactor's drop-out voltage, from its datasheet. Battery undervoltage is *not* its job; that is a software runtime limit with a controlled stop.
- **Layered stop: two stop classes.** Signal cut is the primary gentle stop (ESC ramps down, no contactor wear); the contactor is the backstop.
  - **CONTROLLED STOP** (ordinary aborts): throttle to zero, wait for spin-down, then open the contactor. Triggers: touchscreen soft stop, pendant-link watchdog trip, Pi-link watchdog trip **during an automated sweep** (manual runs continue on Pi loss), sweep abort, arm-interlock fault, runtime limits on temperature (ESC, motor) and minimum battery voltage.
  - **SAFETY TRIP** (evidence that commanded zero is not stopping the motor): throttle to zero **and** open the contactor immediately, accepting the hard-disconnect transient. Uncontrolled torque wins the priority contest. Triggers: current or RPM not falling (or rising) after zero is commanded, runtime limits on current and RPM exceeded, contactor/precharge state contradicting commands, guard lock reading UNLOCKED while armed. The hardware e-stop is the hardware form of a safety trip.
  - **Involuntary hard disconnects** also happen at speed regardless of stop ordering: 24V control power loss, Pico power loss, RP2040 internal watchdog reset, and the undervoltage lockout. Stop ordering cannot prevent these, so the bus must be designed to survive a hard disconnect at full power (A13).
  - **Controlled-stop rule:** open the contactor when **RPM < threshold OR a conservative spin-down timeout expires**, whichever comes first. RPM comes from the Pico's own bidirectional DShot eRPM, so the sequence works with the Pi gone (Pi-link watchdog trip). Missing or invalid RPM data is a fault, but it never keeps the contactor closed: the timeout always wins.
- **Bus-live indicator (mandatory hardware).** A bus-powered indicator on the enclosure face, lit whenever the downstream DC bus is above a defined threshold, independent of the 24V control system (it must work with the Pico and touchscreen dead). The touchscreen mirrors it from Vbus but is not the primary indicator. **Light off is not proof of a dead bus** (a failed LED looks the same): confirm with Vbus or a meter before touching anything downstream.
- **Session-start proof test.** Some failures are latent: a mushroom contact that no longer opens, a guard switch that no longer opens, or a restart latch that no longer drops, is invisible until it is needed. Before each test session: **inspect the containment** (panels, window, fasteners, latch, baffles and openings; a panel damaged by yesterday's failure must not become today's containment), press each mushroom in turn, open and close the guard, **command the guard locked and physically confirm it cannot be opened** (safe to test: the rig is de-energised), and confirm via Pico loop sensing and latch readback that the loop opens and the latch drops. The Pico reads back the restart-latch state for this purpose. Failed proof test = no arming.
  - **A proof test must not rely on the function being tested to prevent hazardous energy.** Tests are designed so that if the tested item has failed, nothing dangerous happens during the test. Example: the precharge-complete gate is tested by reading its output back with the battery connected and the bus discharged; it must read "blocking". Nothing is commanded to close to find out. The master-permit release is tested by commanding release with no energy path active and confirming the latch drops. The bus clamp is **not** routinely tested by generating a full hard-disconnect transient (that relies on the clamp being healthy): routine clamp checks are at reduced energy, with an offline test pulse, or by component-level measurement. Full-power hard-disconnect testing belongs to initial characterisation only, with extra instrumentation and temporary external protection.
- **Three electrical domains** (see A15): 230V AC mains + PE (inlet and PSU input only), isolated 24V control (with a 12V contactor control rail and 5V logic derived from it), and the 12S drone battery (stack/motor only). Common ground reference inside the box, made **deliberately**: a single-point (star) connection between power ground and logic ground, so high motor-current edges do not flow through logic and sensor references. Exact topology at schematic stage.
- **Neither the Pi nor MQTT is ever in the safety path.**

---

## A9. Arm interlock (no spin without confirmation)

Before arming (owned by the Pico, not the Pi): cross-check **measured battery voltage** (battery-side sensor, A7) + **power-module ID** (auto-read from the dock) + **motor selection** (set via settings page). Arm only if all three agree with the loaded profile. Then the arm sequence runs as explicit states, each with its own check and timeout; any failure = refuse arm and latch a fault:

```text
SAFE
  | GUARD CLOSED (door contact proven via NC loop)
  | COMMAND + VERIFY GUARD LOCKED (lock-monitoring contact: bolt engaged)
  | fresh SAFE-to-ARM edge, proof test passed, interlocks agree
  | SNAPSHOT + LOCK motor / prop / battery profile (bound to arm-session number)
  | operator confirms that exact snapshot on the pendant screen
RESTART LATCH SET (hardware, edge-coupled)
  | Vbus plausibility: ~0 before precharge (catches a stuck-high sensor)
  | contactor aux reads open; precharge relay feedback reads open
PRECHARGE ON
  | Vbus >= closing threshold (nominally 95% of Vbattery, ~3 tau)
  | precharge time within [minimum, maximum] window (too fast = effective RC
  |   lower than expected: a shorted precharge resistor, or missing / degraded
  |   bus capacitance; too slow = drift or capacitance change)
  | fault timeout ~5 tau at worst-case capacitance + margin
  | hardware precharge-complete gate now permits the contactor
MAIN CONTACTOR COMMANDED ON
  | VERIFY PHYSICALLY CLOSED via aux contact (Vbus cannot prove it)
PRECHARGE OFF
  | VERIFY PRECHARGE OPEN via force-guided relay feedback
  |   (Vbus cannot prove it: both paths now see battery voltage)
  | verify states sane: no feedback contradictions
VALIDATION SPIN (restricted state)
  | hard low-power throttle ceiling; pendant throttle ignored / capped
  | RPM and current within the broad plausibility envelope for the profile
FULLY ARMED / MOTOR PERMITTED
```

Gates the relay. Not a cosmetic popup.

This is the **full target sequence** (all tiers built). Firmware runs the sequence for the current build tier: the **Tier 1 reduced arm sequence** is in A16, and Tier 2 and Tier 3 add their verification states as their hardware is fitted.

- **Pre-arm confirmation.** The pendant screen shows the snapshot of the selected motor and prop; the operator confirms it before any energy path is enabled. The motor + prop profile is otherwise trusted human input, so this is a deliberate check, not a formality. **For the prop, this confirmation is the actual control:** the validation spin reliably catches a wrong motor or pole count (gross errors), but two similar props can fall inside its deliberately broad envelope. If the prop is uncertain, use conservative limits. (Automatic identification by tag or keyed fixture is an open option.)
- **Validation spin (restricted state before full arming). Keep it deliberately dumb** (see Gauntlet V1). The contactor and precharge are verified, but full motor authority is not granted yet. The Pico runs a short spin at a low, fixed throttle under a hard power ceiling, ignoring or capping pendant throttle. Measured mechanical RPM must fall inside a **broad plausibility envelope** around KV x Vbattery x throttle fraction for the selected motor. With a prop attached, load and ESC timing make RPM non-linear with that expression, so the envelope is wide on purpose: it is there to catch a wrong motor or a wrong pole count (which scales the eRPM-to-RPM conversion by a whole factor), not to validate the motor precisely. Fail = CONTROLLED STOP and fault. Pass = FULLY ARMED.
- **Profile locked when the arm request is accepted, not when arming succeeds.** The Pico snapshots and locks the motor, prop, and battery profile as soon as a fresh arm request is accepted; every step of precharge and the validation spin uses that snapshot. If the arm attempt fails, the snapshot is released and a new confirmation is required. No profile mutation while ARMING **or** ARMED; any settings change that affects the interlock is refused, and sweep parameters are validated against the latched profile (ceiling, ramp rate).
- **Module ID supervised while armed.** The power-module ID is monitored continuously, not only checked at arming. If it disappears or changes unexpectedly while armed: CONTROLLED STOP and fault. The battery has not changed, but one of the assumptions the run was authorised under has been lost.
- **Arm starts from zero.** Arming requires throttle at zero, no active sweep, and a fresh SAFE-to-ARM transition of the arm toggle.
- **Re-arm after any fault** (e-stop loop open, pendant-link or Pi-link watchdog trip, failed precharge) repeats the full sequence above. No shortcut re-arm.

---

## A10. Power modules

- Swappable boards that adapt different battery configs to the rig (2x XT60 series = 12S, 2x XT60 parallel = 6S, single XT30, etc.).
- Each **self-identifies** via a resistor divider or 1-wire chip storing topology + max current + connector type. Read by the Pico at the dock.
- **Motor power always goes through the plugged XT90 into the contactor, never through dock pins.** Dock pins carry only the low-power ID signal.
- Strict, meter-verified polarity on every module (reversed = dead ESC).
- **Each module carries its own primary DC fusing**, placed as close to the batteries as practical, before the output lead, **in the positive (ungrounded) conductors**. Battery negative is bonded to PE at the star point (A15), so the negative side is the reference, not the fused side. Rated to interrupt at 60V DC or higher (DC rating, not AC), sized above maximum bench current with margin, **and with a DC breaking capacity (interrupt current) above the available LiPo fault current**. A fuse that melts but cannot safely interrupt the fault current is not a fuse.
  - **Series modules** (e.g. 2x 6S = 12S): **a fuse at each pack's lead**, not just one on the string output. Each pack is its own energy source; a short in module wiring directly across one pack, upstream of a single string fuse, would bypass it.
  - **Parallel modules:** each battery branch fused before the branches combine, so one pack cannot dump into a faulted parallel partner.
  - Fuse continuous rating and DC breaking capacity are part of the module's documented data (max current in its self-identified data).
- **Two stations:** storage rack (idle modules) and active dock (holds the module, reads its ID).
- Power modules always stay outside the electrical box.

---

## A11. Boards owned (no brains to buy)

2x Pico, 2x Pico W, 2x ESP32-S3, 2x ESP32-WROVER-CAM, assorted Arduinos, Raspberry Pi(s), Pi 5. Roles: ESP32-S3 = pendant; Pico = real-time controller/safety; Pi 5 = data brain/broker; WROVER-CAM = spare/optional second camera.

---

## A12. PCB plan (phase 2)

Custom PCBs are the right end-state (reliability, vibration tolerance, safety circuit on copper not jumpers).

- **Sequence: breadboard/proto first, prove the circuit, then PCB.**
- **Two boards:** safety-critical base (Pico + safety circuit) and the pendant. Separate by design.
- **KiCad**; fab at JLCPCB/PCBWay.
- **RS-485 pendant link details:** the Pico is the only bus master (the pendant replies only when polled), which removes half-duplex collision ambiguity. 120 ohm termination at both ends. Fail-safe biasing so an idle or disconnected line reads as a defined idle state, not noise. Transceiver driver-enable (DE/RE) held in receive by default through reset and boot (pull resistors), so neither side drives the line until firmware is running. Cable shield terminated at the enclosure end only.
- **Socket all removable modules** (Pico, ESP32-S3, ADS1220 breakout, encoder) on female headers.
- **Keyed connectors (JST-XH class)** for all polarity-sensitive lines. Never bare pin headers where a reverse would damage something.
- Embody the fail-safe logic on copper: NC e-stop in series, de-energise-to-stop, appropriate coil transient suppression per A13.

---

## A13. Open electrical design decisions

These are unresolved design questions that must be answered before the schematic is frozen or the prototype is built. Capturing them here prevents them from being hand-waved past.

- **RP2040 internal watchdog and enable fail-low:** the RP2040 internal watchdog protects against firmware hangs -- if firmware stops servicing it, the Pico resets and relay-enable goes low. However, the internal watchdog is not independent of the Pico itself; it does not cover every MCU failure mode (loss of power, GPIO floating during boot, disconnected Pico). Every energy-path enable (relay-enable and precharge-enable) must therefore be fail-low in hardware (see A8): an external pull-down resistor on the relay-driver control line ensures that Pico reset, boot, unpowered GPIO, or a physically disconnected Pico all result in relay-enable low and contactor off by default. The RP2040 internal watchdog does the job it is good at (firmware hang recovery); the pull-down covers everything else. An external watchdog supervisor IC can be added later if stronger independence is desired. **Firmware rule:** the RP2040 internal watchdog is serviced only from the main control loop, after all safety checks for that cycle have passed. Never from a timer interrupt, which could keep servicing it while the main loop is hung. Decide: RP2040 internal watchdog timeout value; pull-down resistor value sized against the Pico's drive strength.

- **Contactor precharge / closing inrush:** precharge is part of the architecture (see A9 -- main contactor closes only after Vbus reaches ~95% of Vbattery). The yes/no question is closed. Open implementation questions: final precharge resistor value, timeout, Vbus threshold, and power rating (switching element decided: force-guided safety relay with feedback contact, A9). These depend on measured ESC bus capacitance (external 63V 1000uF cap confirmed; total bus capacitance to be measured on the actual assembled unit) and the selected contactor's make-current capability. Size the resistor for a precharge time of roughly 5 time constants at worst-case capacitance; verify actual precharge curve on the bench before first arm. **Resistor sizing is energy-based, not wattage-based.** Select the part from its pulse-energy / overload curves and maximum working voltage, accounting for pulse duration, repetition rate, and ambient temperature. Nominal wattage alone is not the sizing number.
  - **Normal precharge energy:** approximately 1/2 x C x V^2, recalculated once actual bus capacitance is measured.
  - **Fault sizing:** if Vbus cannot rise because the downstream bus is shorted (for example a failed ESC), the resistor sees approximately Vbattery^2 / R for the full fault timeout.
  - **Series redundancy against resistor short (single-fault requirement):** the precharge resistance is **two resistors in series**, each sized so that it alone still limits current to something the branch (relay, wiring, capacitor) survives if the other fails short. A single shorted precharge resistor must not turn PRECHARGE ON into a direct battery-to-bus connection through the precharge relay. Detection of the first failure: precharge time is measured every arm (it is proportional to R x C), and a precharge that completes too fast means the effective RC is lower than expected: one resistor has shorted, or bus capacitance is missing or degraded. Either is a fault. Without that check, the redundancy would silently degrade to a single resistor. Fail = fault, refuse arm.
  - **Worked example (provisional 100 ohm total, 1 mF, 50.4V):**

| Quantity | Value |
|---|---|
| Initial precharge current | 0.504 A |
| Initial resistor power | 25.4 W |
| Time constant | 0.100 s |
| 5 time constants (charge time) | 0.500 s |
| Normal precharge energy | ~1.27 J |
| Shorted-bus fault, 0.5 s timeout | ~12.7 J |
| Shorted-bus fault, 1.0 s timeout | ~25.4 J |
| Shorted-bus fault, 2.0 s timeout | ~50.8 J |

  The fault case dominates by an order of magnitude, so the timeout value and the resistor choice are one decision. The precharge switch is loop-powered (A8).
  - **Closing threshold vs fault timeout are two different events.** 95% of Vbattery is reached at about 3 time constants (t = -ln(0.05) x RC, about 3.0 RC); 5 time constants is about 99.3%. Proposed: close at >=95% (~3 tau), fault timeout at ~5 tau at worst-case capacitance plus margin. At a 95% threshold the residual differential is about 2.5V. The residual make current is that differential divided by total loop resistance (battery, wiring, capacitor ESR, contact resistance), which is not yet known: at 2.5V it is 50A at 50 mohm, 100A at 25 mohm, 250A at 10 mohm, 500A at 5 mohm. With 8 AWG and low-ESR capacitors the low end of that resistance range is plausible. Calculate from bounded values or measure on the assembled rig, then check against the contactor's capacitive make rating; raise the threshold toward 99% if needed.
  - **Failed-closed precharge branch protection.** The timeout only works if the precharge switch can actually turn off. With the switch welded or failed short and the bus shorted, the resistor dissipates Vbattery^2 / R (about 25 W at 100 ohm) **continuously** until the power module is unplugged. Detection alone does not contain this. The branch needs its own protection that distinguishes by duration: normal precharge and the fault both start at the same current (about 0.5A) and differ only in how long it lasts. Options: thermal cutoff bonded to the resistor, a fusible resistor designed to open under sustained overload, a resistor with a defined safe open-circuit failure mode, or a suitably characterised time-delay fuse (its I2t curve must pass the normal decaying precharge pulse but open under sustained current before the resistor is damaged). Choose at component selection.

- **Coil suppression and drop-out speed:** a plain flyback diode (UF4007) across the 24V contactor coil protects the relay-driver MOSFET but allows the coil current to decay slowly, which slows contactor release. For an emergency stop, slower release = longer motor coast-down. A diode-plus-Zener in series (Zener clamps the voltage spike but lets current decay faster than a plain diode) is the standard fix. Decide: what is the actual release time spec of the chosen contactor? Is the plain diode adequate, or is a TVS/Zener suppressor needed to hit the required drop-out speed? Resolve after the contactor is sourced.

- **Hardware seal-in implementation:** now mandatory (A8). Must be independent of the main contactor (pilot relay with its own holding contact, or a discrete latch), cover both energy paths, use an edge-coupled start input, and power up unlatched with power-on-reset gating. Decide pilot relay vs discrete latch at PCB stage.

- **Hard-disconnect DC-bus transient:** opening the main contactor during a high-power run removes the battery, which was clamping the bus, while the motor is still spinning with considerable stored energy. Sources of bus overvoltage: (1) **ESC active braking / regenerative PWM** at zero throttle, which pumps energy back into the bus with no battery to absorb it (the main risk); (2) **wiring inductance** producing a spike when current is interrupted; (3) passive rectification of motor back-EMF through the ESC MOSFET body diodes, which is roughly bounded by the back-EMF itself and therefore below pack voltage at the moment of disconnect. Headroom is only 63V capacitor vs 50.4V full pack. Decide: check and set the ESC's stop/brake behaviour; characterise Vbus during emergency contactor opening on the bench (scope, starting at low power and walking up); confirm peak Vbus stays safely below the capacitor and MOSFET limits; decide the clamp design. **Some bus protection should be assumed necessary, not optional**, and the options are not interchangeable because they act on different timescales:
  - **TVS:** absorbs the fast wiring-inductance spike (microseconds). Small energy, fast edge.
  - **Active dump / brake resistor:** absorbs bulk regenerated energy over milliseconds or longer. A TVS alone is not sized for that.
  - **ESC set to coast rather than actively brake on stop:** removes most of the regenerated energy at the source. If this is sufficient, a TVS alone may cover the rest. Decide on the bench.
  - Likely outcome: TVS for the edge, plus either coast behaviour or a dump path for the energy underneath it.
  - **Clamp health is a latent fault, not a runtime-monitored one:** a microsecond spike passes straight through a normal Pico ADC and logging loop, so "Vbus peak logging" cannot detect clamp degradation. Initial characterisation with a scope (with extra protection); periodic re-test at reduced energy or component level (A8 proof-test rule); or add hardware peak detection if runtime monitoring is wanted.
  The reason protection is assumed necessary: the controlled-stop ordering reduces how often hard disconnects happen, but safety trips, e-stops, 24V loss, Pico power loss and RP2040 watchdog resets all disconnect at speed (A8). Characterise the worst case (full power, sudden disconnect) regardless of which path causes it.

- **ESC firmware support (open dependency):** the design relies on the DAKE ESC firmware supporting bidirectional DShot (eRPM) and the serial telemetry wire. Treat both as unconfirmed until verified on the bench; if either is missing, the RPM source and runtime limits need a fallback.

- **Touchscreen session token bootstrap:** the access control rule requires that only the physically local touchscreen obtains the privileged session token. If any browser loading `/` can request a token, the model collapses. Decide: how does the Pi know the first request at boot is from the local display and not a phone that connected first? Options include: token issued only via a loopback-bound endpoint (accessible only from the Pi itself, not the AP); token displayed as a QR code on the screen at boot that must be scanned locally; or a physical button press on the enclosure to issue the token. This does not need to be solved in the architecture document but must be decided before the FastAPI auth layer is coded.

- **Containment design against the worst credible projectile:** evaluate candidate projectiles, not just a blade fragment: blade sections, prop hub, spinner / prop nut, motor bell, **magnets detaching from the bell** (a known high-RPM failure), screws, adapters. Calculate kinetic energy at maximum RPM for the largest prop and fastest motor in the profile library, take the governing case, and choose tunnel wall material, thickness and fastening against it (polycarbonate alone may not be sufficient; layered construction or steel mesh are options). Check openings (cables, airflow) for escape paths. Guard lock is decided as power-to-release (A8); the remaining choice is the specific guard-locking switch.

- **Fuse and contactor protection coordination:** once both are chosen, compare the contactor's DC breaking capacity and short-time withstand against the fuse's time-current and I2t curves, across the whole range from safety-trip current to prospective LiPo short-circuit current. Confirm there is no current band where the contactor is asked to break more than it can and the fuse has not yet cleared.

- **Run-down measurement (decides the guard unlock method, A8):** worst-case run-down with the ESC unpowered, prop fitted and bare bell, the bare bell starting from no-load speed (KV x max pack voltage) of the fastest motor, measured with a temporary optical tachometer. Short and repeatable = validated time delay; long = standstill sensor required. If a standstill sensor is fitted: optical (reflective mark on the bell) vs Hall (bell magnets), mounting in the fixture, and how "sensor dead" is distinguished from "stationary" (periodic self-test, or a sensor with a status output).

### Schematic-stage checklist

Detailed-design items that do not change the architecture, to have open while drawing the schematic:

- **Current sensing and grounding:** Hall or high-side shunt only; single-point power/logic ground; route high-current returns away from sensor references.
- **Robust module ID:** the module returns a real identifier with integrity checking (for example a 1-wire ID chip with CRC), and the Pico maps that ID to trusted limits (topology, max current). Vbattery cross-checks voltage but cannot prove topology or current rating. If an analog resistor ID is used, define explicit invalid bands between valid states, so a noisy or out-of-tolerance reading is rejected rather than misread.
- **Pendant sending valid but wrong commands:** a faulty ESP32 or encoder can emit CRC-correct but unintended throttle deltas; link watchdogs and CRC do not catch this. Option: a spring-return hold-to-run enable for manual mode. To be effective it must **not** route through the ESP32 (a misbehaving MCU could fake it), which needs its own conductor: the 7-pin GX16 has none spare, so either move to an 8-pin connector or accept the guard interlock and containment as the mitigation. Decide.
- **Gate readback circuit, gate hysteresis and dwell values, divider protection values, RS-485 termination and biasing values:** as specified in A8, A7 and A12.

- **Post-disconnect stored energy (bus bleed):** opening the contactor disconnects the battery but does not discharge the ESC bulk capacitor. The downstream DC bus remains live at close to full pack voltage after the contactor opens. This is the natural complement to precharge. Needs: a bleed/discharge resistor path across the DC bus (sized for acceptable discharge time without excessive standby loss), a "DC bus live" indicator visible to the operator before touching anything downstream, and a characterised discharge time so the operator knows how long to wait. The Pico can monitor bus-side voltage (already available if the bus-voltage sensor from the precharge circuit is fitted) and indicate when the bus has discharged to a safe level. Decide: acceptable discharge time, bleed resistor value and power rating. The indicator is now decided: a bus-powered hardware indicator is mandatory (A8), with the touchscreen as a mirror.

---

## A14. Failure mode table (skeleton)

A systematic walk through every safety-relevant item: how it fails, what happens, how it is detected, and what contains it. Contactor-dependent rows get completed when the contactor is chosen. "Proof test" = the session-start test in A8.

| Item | Failure mode | Effect / hazard | Detection | Hardware containment | Software response | Recovery | Open dependency |
|---|---|---|---|---|---|---|---|
| NC e-stop loop | Mushroom pressed, wire break, cable unplugged | Coil and precharge drive removed | Pico loop sensing | Contactor and precharge de-energise; restart latch drops | Throttle 0, abort sweep, latch fault | Reset mushroom + full re-arm (fresh edge) | None |
| NC e-stop loop | Mushroom contact fails closed | That station's e-stop does nothing (latent) | Proof test | Other mushroom still in series | No arming if proof test fails | Replace contact block | None |
| Main contactor | Welded closed | Battery stays on bus after e-stop | Aux contact (immediate); Vbus after spin-down | Module fuse (overcurrent only) | Throttle 0 via DShot, latch fault, refuse arm | Inspect / replace | Aux contact type |
| Main contactor | Fails to close | Arm sequence would proceed on precharge alone | Aux contact only (Vbus cannot tell: precharge holds it at Vbattery) | None needed | Refuse arm at VERIFY CLOSED state | Inspect | Contactor with mirror aux contact |
| Contactor coil | Slow drop-out | Longer coast before disconnect | Bench measurement | Suppression choice | None | None | Release time spec |
| Precharge switch | Fails closed | Bus live when not commanded; sustained ~25 W into a shorted bus | Vbus rising / holding unexpectedly | Resistor limits current; branch thermal / time-delay protection | Latch fault, refuse arm | Replace | Protection part |
| Precharge resistor | One of the two series resistors fails short | Precharge current doubles (still limited by the other); redundancy lost (latent) | Precharge completes faster than the minimum time window | Second series resistor | Fault, refuse arm | Replace resistor | Resistor sizing |
| Precharge resistor | Both fail short (second failure) | Direct battery-to-bus path through the precharge relay: full inrush, relay may weld | Precharge far too fast; relay feedback | Module fuse | Fault, refuse arm | Replace branch | Covered by first-failure detection |
| Precharge-complete gate switch | Series switching element fails short | Gate cannot block early closure while comparator still says "block" (latent) | Readback of the switched coil path (after the switch) disagrees with comparator | Driver MOSFET still required | No arming if proof test fails | Repair | Coil-node readback circuit |
| Precharge switch / resistor | Fails open | Vbus never rises | Precharge timeout | None needed | Refuse arm, fault | Replace | None |
| ESC / DC bus | Short circuit | Precharge into short; overcurrent if contactor closed | Vbus fails to rise; current telemetry | Precharge protection; module fuse | Refuse arm / SAFETY TRIP if running | Replace ESC | Fuse sizing |
| Module fuse | Blown | No power | Vbattery reads ~0 | None needed | Refuse arm | Replace fuse, find cause | Fuse part |
| Module fuse | Fails to interrupt when required (wrong fuse fitted, interrupt capacity exceeded, holder fault, poor coordination) | Fault current persists; contactor asked to break beyond rating; wiring and pack damage, fire | Not detectable at runtime | Correct fuse selection and coordination (A13); fuse rating is part of the module's documented data | None effective | Inspect after any severe fault | Fuse-contactor coordination |
| Restart latch | Stuck latched | Mushroom reset could restart the rig (latent) | Proof test via latch readback | Pico enable still required; fail-low enables | No arming if proof test fails | Replace | Latch design |
| Restart latch | Latches at power-up | Would bypass reset discipline | Latch readback at boot | Power-on-reset gating | Fault if latched at boot | Power cycle / repair | POR circuit |
| Pico | Firmware hang | Outputs frozen; watchdog reset = involuntary hard disconnect at speed | RP2040 internal watchdog (main-loop serviced) | Fail-low enables; restart latch; NC loop | Reset to safe state | Fresh arm edge | Watchdog timeout |
| Pico | Power loss / disconnected | Outputs float; DShot stops; involuntary hard disconnect at speed | ESC signal loss | Pull-downs: both energy paths off; ESC disarms on signal loss | None | Full re-arm | ESC signal-loss behaviour |
| Pendant link | Link lost | No pendant commands | Pendant-link watchdog | NC loop independent of the link | Throttle 0, CONTROLLED STOP | Full re-arm | Timeout value |
| Pendant (ESP32 / encoder) | Emits valid, CRC-correct but unintended throttle commands | Unwanted motor demand | Not detectable by link checks; runtime limits bound the result | Runtime limits; containment; guard interlock; optional hold-to-run not routed through the ESP32 | Runtime-limit trips | Inspect pendant | Hold-to-run decision |
| Pendant link | Corrupted packets | Wrong command | CRC, sequence, bounds checks | RS-485 differential signalling | Discard; rising error rate = fault | None | Termination / biasing |
| Pi link | Pi dies or hangs | Display and DB logging lost | Pi-link watchdog | None needed | Abort sweep, CONTROLLED STOP; manual runs continue | Restart Pi | None |
| Pi link | Stale / delayed command | Wrong operation | Arm-session number | None needed | Reject | None | None |
| Vbattery sensor | Wrong or open reading | Interlock check wrong | Plausibility vs module ID; Vbus = Vbattery after close | None | Fault, refuse arm | Repair | Divider / ADC design |
| Vbus sensor | Reads falsely high | Precharge false pass: close onto discharged bus | Plausibility: Vbus ~0 before precharge after bleed; rise must follow expected curve | Contactor make rating as backstop | Fault, refuse arm | Repair | Divider failure direction |
| RPM source (bidir DShot) | Missing / invalid eRPM | Spin-down cannot be confirmed; RPM limit blind | Telemetry timeout / invalid frames | None needed | Controlled stop via timeout fallback; fault | None | Spin-down timeout; ESC firmware support |
| ESC | Desync / drive failure; still driving after zero commanded | Erratic motor or uncontrolled torque | RPM / current not falling after zero; RPM vs throttle mismatch | E-stop, module fuse | SAFETY TRIP (immediate contactor open) | Inspect | None |
| ESC braking on hard disconnect | Bus overvoltage | Capacitor / MOSFET damage | Bench scope characterisation (runtime logging catches only slow overvoltage) | ESC set to coast; TVS for the edge; dump resistor if needed | None | Inspect | A13 characterisation |
| Arm toggle | Left in ARM | Unintended re-arm | Fresh-edge rule | None | Ignore level | Toggle SAFE then ARM | None |
| Power-module ID | Wrong ID read | Wrong interlock profile | Cross-check with Vbattery | None | Refuse arm | Inspect dock | ID scheme |
| Power-module ID | Lost or changed while armed | Run no longer matches its authorisation | Continuous ID supervision | None | CONTROLLED STOP, fault | Inspect dock contacts | ID scheme |
| Bus bleed resistor | Fails short | Effectively a bus short | Vbus fails to rise at precharge | Precharge resistor limits; module fuse if contactor closed | Refuse arm / SAFETY TRIP | Replace | None |
| Bus bleed resistor | Open | Bus stays live after stop | Vbus not decaying while aux contact says open (distinguishes from weld) | Bus-live indicator | Fault | Replace | None |
| Mains PE | Open (earth connection lost) | Insulation fault leaves exposed metal live; RCD cannot see the fault | Periodic PE continuity check | Class I PSU; RCD-protected socket; double insulation where possible | None at runtime | Repair | Inspection interval |
| Mains / SELV segregation | Mains wiring contacts low-voltage side | Hazardous voltage on the 24V, 5V or battery side | None at runtime | Separate covered mains compartment; certified PSU isolation; 0V bonded to PE (fault trips protection) | None | Repair | Compartment design |
| 24V PSU | Primary-secondary isolation fault | Mains on the control side | None at runtime | Certified PSU isolation; 0V bonded to PE so the fault trips the RCD / fuse | None | Replace PSU | PSU choice |
| 24V PSU | Output overvoltage | Coil, comparators and 5V converter overstressed | PSU built-in OVP; optional 24V monitor | PSU OVP | Fault if monitored | Replace PSU | PSU OVP spec |
| 24V-to-12V converter | Fails high (24V on the 12V rail) | 12V coils overheat; eventually burn open (contactor drops: safe direction) | Optional 12V rail monitor | Optional TVS / crowbar on the 12V rail | Fault if monitored | Replace converter and any cooked coils | Clamp decision |
| 24V-to-12V converter | Fails low / off | Coil and loop lose power: contactor drops, precharge off | UVLO; Pico sees loop / coil state | Inherent (de-energise to stop) | Latched fault | Replace converter | None |
| 24V-to-5V converter | Fails high (output overvoltage) | Pi, Pico, sensors damaged together; Pico outputs undefined | 5V monitor (optional) | Overvoltage clamp / crowbar on the 5V rail; fail-low enables | None | Replace | Clamp design |
| 24V control power | Lost | All logic off; involuntary hard disconnect at speed | Inherent | Coil and precharge drive lost; contactor opens; ESC loses DShot; bus clamp absorbs transient | None | Latch powers up unlatched; fresh arm | Bus clamp design |
| 24V control power | Brownout (sags, does not vanish) | Coil between drop-out and pull-in: contactor chatter under load | Undervoltage lockout comparator | UVLO removes both enables cleanly, with hysteresis | Fault | Fix supply; fresh arm | UVLO threshold vs contactor drop-out |
| Undervoltage lockout | Fails to trip | Chatter possible in brownout (latent) | Bench test of threshold | None | None | Repair | Periodic test method |
| Undervoltage lockout | False trip | Involuntary hard disconnect at speed | Fault log (UVLO state readback) | Bus clamp | Fault | Inspect | UVLO readback |
| E-stop loop sense input | Stuck reads "closed" | Pico misses an e-stop: throttle not zeroed (contactor still opens in hardware) | Proof test (press each mushroom, sense must change) | Hardware loop still opens contactor and precharge | No arming if proof test fails | Repair | Sense circuit design |
| E-stop loop sense input | Stuck reads "open" | Rig cannot arm | Inherent (safe) | None needed | Refuse arm | Repair | None |
| Restart-latch feedback | Stuck reads "dropped" | Proof test passes falsely; stuck latch hidden | Complementary-pair readback disagrees; latch must read "set" during arm | Pico enable still required | Fault | Repair | Latch with complementary contacts |
| Contactor aux feedback | Stuck reads "open" | Arm always refused | Inherent (safe); complementary pair disagrees | None needed | Refuse arm | Repair | None |
| Contactor aux feedback | Stuck reads "closed" | Weld detection blind; closure unverified | Complementary pair disagrees; must read open before precharge | Vbus after spin-down still catches weld | Fault, refuse arm | Repair | Contactor provides NO + NC aux |
| Voltage divider component | Lower resistor open / upper shorted | ADC overvoltage (Pico damage); reading stuck at full scale | Plausibility: Vbus ~0 before every precharge | Series resistance + clamp; split upper resistor | Fault, refuse arm | Repair | Divider design |
| Bus-live indicator | LED failed open | Bus looks dead while live | Touchscreen mirror from Vbus disagrees | None | Warn | Replace | None |
| Bus-live indicator | Fails short | Bus load / short via indicator circuit | Vbus fails to rise; indicator current limit | Indicator current-limit resistor; precharge limits; module fuse | Refuse arm | Replace | Indicator circuit design |
| Runtime limit | Current / RPM exceeded | Overload, mechanical risk | Pico telemetry vs latched limits | None | SAFETY TRIP | Inspect | Limit values per profile |
| Contactor driver MOSFET | Fails short | Would energise contactor as soon as the restart latch sets, before precharge | Aux reads closed before commanded | Precharge-complete gate (second series element) blocks the coil until Vbus is precharged | Fault, refuse arm | Replace driver | Gate circuit design |
| Contactor driver MOSFET | Fails open | Contactor never closes | Aux fails VERIFY CLOSED | None needed | Refuse arm | Replace | None |
| Precharge driver / relay | Fails short / welds | Before arm: precharge on as soon as latch sets (resistor-limited). After a stop: bus stays tied to battery through the resistor | Vbus rising before commanded; force-guided feedback | Resistor; branch thermal protection; master permit (latch release) removes coil power | Release latch, fault, refuse arm | Replace relay | Relay DC rating |
| Precharge-complete gate | Stuck permitting | Driver-short protection before closure lost (latent) | Proof test by output readback: battery connected, bus discharged, gate must read "blocking" (nothing is commanded closed) | Driver MOSFET still required | No arming if proof test fails | Repair | Gate readback circuit |
| Precharge-complete gate | Divider fault reads Vbus falsely high | Gate permits early closure | Separate dividers: Pico plausibility disagrees with gate readback | Driver MOSFET still required | Fault, refuse arm | Repair | Separate divider networks |
| Master permit (restart latch) | Release input fails (latch cannot be released by the Pico) | Shorted driver could hold contactor in (latent) | Proof test: command release with no energy path active, latch must drop | Driver MOSFETs; e-stop loop still drops the latch | No arming if proof test fails | Repair | Release circuit design |
| Master permit (restart latch) | Latch contact welded | Master permit lost (latent) | Proof test (latch readback must show drop); complementary readback | Driver MOSFETs; precharge gate | No arming if proof test fails | Replace relay | Latch relay choice |
| Precharge-complete gate | Stuck blocking | Contactor never closes | Aux fails VERIFY CLOSED | None needed | Refuse arm | Repair | None |
| Bus clamp (TVS / dump) | Fails short | Bus shorted: overcurrent | Vbus fails to rise at precharge; current | Precharge protection; module fuse | Refuse arm / SAFETY TRIP | Replace clamp | Clamp design |
| Bus clamp (TVS / dump) | Fails open or degraded / undersized | Hard-disconnect protection lost (latent) | Periodic re-test at reduced energy or component level (never by a full hard disconnect); optional hardware peak detector | None | None at runtime unless peak detector fitted | Replace | Characterisation; peak-detect decision |
| Series power module | Short across one pack in module wiring | Pack dumps into the short | Inherent (fuse clears) | Per-pack fuse at each pack lead | Fault if Vbattery drops | Repair module | Fuse selection |
| Dedicated current sensor | Lost, frozen, or implausible | Overcurrent SAFETY TRIP blind; efficiency data wrong | Cross-check vs ESC-reported current; frozen-value detection | None | SAFETY TRIP if lost while running; refuse arm | Repair | Sensor choice |
| ESC serial telemetry | Lost, frozen, or implausible | ESC temperature limit blind | Telemetry timeout; frozen-value detection | None | CONTROLLED STOP | Inspect | ESC firmware support |
| PT1000 / ADS1220 | Open, shorted, frozen, or implausible reading | Motor temperature limit blind | Range check (open/short read as out-of-range); frozen-value detection; ADS1220 SPI health | None | CONTROLLED STOP | Repair | None |
| Motor / prop profile | Wrong motor selected (or wrong pole count) | Wrong limits and RPM conversion | Pre-arm confirmation; validation spin (broad RPM envelope) | None | CONTROLLED STOP, fault | Correct profile | Identification method |
| Motor / prop profile | Wrong prop selected | Wrong max RPM limit (prop-dependent) | Pre-arm confirmation of the snapshot (the actual control). Validation spin is a plausibility aid only: similar props fall inside its envelope | Conservative limits if uncertain | CONTROLLED STOP if spin data is grossly off | Correct profile | Prop identification method (open) |
| Runtime limit | Temperature or min battery voltage exceeded | Thermal damage, pack damage | Pico telemetry vs latched limits | None | CONTROLLED STOP | Cool down / swap pack | Limit values per profile |
| Containment guard | Opened while armed | Access to a spinning prop | Guard switch in the NC loop (hardware) | Loop opens: contactor and precharge drop; guard lock should prevent this while spinning | Latched fault | Close guard, full re-arm | None |
| Containment guard switch | Fails closed / defeated | Rig can run with access open (latent) | Proof test (open guard, loop must open) | Positive-opening / coded switch resists this | No arming if proof test fails | Replace switch | Switch type |
| Guard lock | Fails to lock | Access possible during spin-down | Lock-monitoring contact (bolt position) | Guard switch still opens the loop (stops drive, not the coasting prop) | Refuse arm if lock not confirmed | Repair | Lock type |
| Guard-lock driver | Fails short (release energised) | Would unlock the guard | Lock-monitoring contact | UNLOCK button in series with the driver: nothing happens until the operator presses it | Refuse arm; SAFETY TRIP if it unlocks while armed | Repair | Driver design |
| UNLOCK button | Stuck pressed | Would unlock as soon as permitted | Button state read at arm; must read released | Pico permission still required (series with driver) | Refuse arm | Repair | None |
| Guard lock | Reads UNLOCKED while armed (any cause) | Physical barrier gone during a run | Lock-monitoring contact | Guard door contact in NC loop if the lid opens | SAFETY TRIP | Inspect | None |
| Guard-lock driver | Fails open (cannot release) | Cannot open containment after stop | Inherent | Manual escape release | None | Manual release, repair | None |
| Guard-lock wiring | Breaks | Power-to-release lock stays locked (safe direction) | Inherent | Spring lock | None | Manual release, repair | None |
| Lock-monitoring feedback | Stuck reading "locked" while the bolt is not engaged | Sequence proceeds with guard unlocked (the dangerous case) | Session-start physical lock check (guard must not open); feedback must change state on every lock / unlock cycle | Guard door contact still in NC loop | No arming if proof test fails; fault if feedback never changes | Repair | Switch with bolt-position monitoring |
| Lock-monitoring feedback | Stuck reading "unlocked" | Rig cannot arm | Inherent (safe) | None needed | Refuse arm | Repair | None |
| Manual escape release | Left engaged / not reset | Guard unlocked while armed | Release state monitored | Guard door contact still in NC loop | Refuse arm while engaged | Reset release | Monitored release |
| Standstill sensor (if used for unlock) | Dead, or reads stationary while spinning | Guard could unlock during run-down | Cross-check vs eRPM while ESC is powered; sensor self-test / status; dead must never read as zero | Unlock also requires valid Vbus | DO NOT UNLOCK on missing / implausible data | Manual release after physical verification | Run-down measurement; sensor choice |
| Guard unlock delay (if time-delay method) | Delay too short (run-down longer than validated: new motor, lighter prop, bare bell) | Guard unlocks while rotor still turning | Re-validate run-down whenever the rotating assembly changes | Margin on the measured worst case; bare-bell case included in validation | None at runtime | Re-measure, lengthen delay | Run-down measurement |
| Guard lock | Fails locked | Cannot open containment | Inherent | Manual escape release | None | Manual release, repair | Escape release design |
| Containment wall / panel | Penetrated or cracked by fragment | Projectile escapes | Inspection after any shed event | Wall designed against fragment energy | None | Replace panel | Fragment energy calculation |
| Containment fasteners / latch | Fail under impact | Lid or panel opens under impact | Inspection | Fastening designed against fragment load | None | Repair | Fastener design |
| Containment openings | Cable / air opening forms an escape path | Projectile escapes | Design review | Baffled or shielded openings | None | Redesign | Opening design |
| Prop / mechanical | Prop shed, part lets go | Projectile | Pico-local only: sudden eRPM rise toward the limit, current drop, thrust collapse (ADS1220). IMX500 annotates the event but is never required for the trip | Tunnel containment | SAFETY TRIP (unloaded motor overspeeds) | Inspect | Tunnel design |

---

## A15. Mains supply and electrical domains

The box contains 230V mains as well as the 12S battery. The mains side gets its own architecture, kept deliberately small.

```text
230V AC mains + PE    (inlet module + PSU input only, separate compartment)
        |
   Mean Well PSU       (certified isolation)
        |
isolated 24V control  (guard lock, converters)
        |-- 12V contactor control rail (e-stop loop, latch, contactor coil, precharge relay, UVLO)
        |-- 5V logic           (Pi, Pico, sensors, pendant)

12S battery power     (power modules -> contactor -> ESC -> motor)
```

- **Minimise DIY mains work:** certified parts only on the mains side: a fused and switched IEC C14 inlet module and the DIN-rail PSU. Very little field wiring between them.
- **IT networks:** many Norwegian installations are IT grids (230V line-to-line, no neutral, each conductor roughly 130V to earth). A single fuse in one conductor can leave the other live. Requirements: **double-pole switch**, fusing that protects both conductors (or an inlet module rated for IT networks), and a PSU rated for IT networks.
- **RCD:** operate only from an RCD-protected socket.
- **Protective earth:** PE from the inlet to the PE terminal, bonded to any metal enclosure parts and the DIN rail if metal. Class I PSU earthed.
- **Low-voltage reference:** 0V is bonded to PE at the single star point (a PELV arrangement), not left floating, so an insulation fault from mains trips protection instead of leaving the low-voltage side at an unknown potential. The battery negative shares this reference through the star point.
- **Segregation:** mains in its own covered compartment, finger-safe terminals, strain relief on the inlet cable, physical separation from all low-voltage wiring.
- **Overvoltage:** PSU with built-in output OVP; overvoltage clamp / crowbar on the 5V rail, since a 5V converter failing high takes out the Pi and the Pico together. A crowbar needs the upstream source current-limited or fused, otherwise it trades overvoltage for overheating.
- **Test equipment grounding:** with battery negative / 0V bonded to PE, a mains-earthed oscilloscope's ground clip **is** battery negative. Clipping it to a non-ground node (the series midpoint of a 2x 6S module, a motor phase, the high side of a shunt) shorts that node through the earth path. Only ground-referenced nodes with a mains-earthed scope; everything else with a differential probe or a battery-powered scope.
- FMEA rows for PE open, segregation failure, PSU isolation fault, 24V overvoltage and 5V converter failing high are in A14.

---

## A16. Build tiers (build order)

The full document is the target design. The tiers decide what is built first, using the design-basis filter (projectile / dangerous voltage / fire). Everything in a lower tier is required before the conditions of the next tier. **Each tier must be executable on its own**: nothing in Tier 1 may depend on hardware from Tier 2 or 3.

**The build tier is a firmware setting, not a UI setting.** The tier and its output ceiling are compile-time constants (changed only by reflashing the Pico), so moving up a tier is a deliberate act and cannot be done from the touchscreen or the settings page.

### Tier 1: before the first powered spin (low power, 6S, manual only)
- **Containment** sized against the worst credible projectile, openings checked, **guard door switch in the NC loop** (A1, A8). Built to full specification **before the first prop run** (Stage 2 below), not as a temporary version: even on 6S the credible maximum is the no-load speed, about 25k RPM, which is still a very fast prop. The Tier 1 throttle ceiling is not a verified mechanical RPM ceiling.
- **V1 contactor: Hongfa HFE82V-60B/750-12-HL5** (60A, 12V coil, no aux) on the 12V contactor control rail, with the NC loop on the 12V side. Deliberately a V1 part: replaced by a 100A class contactor with a mirror aux contact before full 12S power.
- **24V-to-12V DC-DC converter** for the contactor control rail.
- **Hardware e-stop NC loop** with both mushrooms, **contactor**, de-energise-to-stop (A8).
- **Power-module fusing** at the source, correct DC interrupt rating and breaking capacity, in the positive conductors (A10).
- **Mains section**: certified inlet module and PSU, double-pole switching, PE, segregation, RCD socket (A15).
- **Single 6S power module only.** A physical ceiling on top of the software one: no-load speed roughly halves (about 25k RPM for a 1000KV motor at 25.2V) and stored rotational energy drops to about a quarter (energy scales with RPM squared). Move to 12S only when Tier 2 is in.
- **6S enforced in firmware too (`TIER_1_VBATT_WINDOW`):** Tier 1 refuses to arm unless measured Vbattery is inside a defined 6S window, nominally **19.8 to 25.4V** (3.3 to 4.23 V/cell). A 12S pack never overlaps it (even a nearly flat 12S is about 39.6V), so an accidentally connected 12S module is always rejected. **Limitation:** a well-discharged **8S** pack (around 3.15 V/cell, about 25.2V) can fall inside the window, so voltage cannot prove pack identity; the physical "6S module only" rule still applies in Tier 1.
- **Low-battery stop:** CONTROLLED STOP when Vbattery under load falls below a defined per-cell threshold for 6S, so a long test cannot overdischarge the pack.
- **Enforced output ceiling (`TIER_1_MAX_OUTPUT`):** a hard **throttle / DShot output ceiling** in the Pico that the pendant cannot exceed, whatever it sends. Starts conservative, raised deliberately by reflashing. Note: this is an output ceiling, **not a power limit**: the same throttle draws different power with different motors and props, and Tier 1 has no current sensor. Measured current and power limits arrive in Tier 2.
- **Before raising the ceiling significantly:** characterise the stop behaviour and bus transient at low energy (scope on Vbus during an e-stop at the current ceiling). 6S voltage headroom helps, but it is not proof that every disconnect transient is harmless.
- **Basic Vbattery + Vbus sensing:** two protected dividers into the Pico (series resistance + clamp), and the "Vbus ~0 before precharge" plausibility check (A7). Moved here from Tier 2 because Tier 1 precharge cannot work without them.
- **Basic precharge**: resistor + relay, contactor closes only after Vbus reaches threshold within the timeout (A9). Single resistor acceptable at this stage, **with basic branch thermal protection** (thermal cutoff bonded to the resistor, or a fusible resistor): a welded precharge relay into a shorted ESC would otherwise heat the resistor continuously (about 6.4 W at 25.2V through 100 ohm; smaller than at 12S, still enough to cook a small resistor). Fire protection now; redundancy and diagnostics later (Tier 3).
- **Coil-energised LED** across the contactor coil drive: an **indication of coil-drive state only**. It does not prove the main contacts are open (a welded contactor can have a de-energised coil) and a failed LED can stay dark while the coil is powered. Used in the recovery procedure as an extra check, alongside physically disconnecting the module and the Vbus check.
- **Bus-live indicator**, bus-powered LED with threshold and current limit (A8). Moved here from Tier 2: an LED and a couple of resistors. Until fitted, verify with a meter.
- **Bus bleed resistor** (A13).
- **Fail-low pull-downs** on every energy-path enable (A8). Cheap: resistors.
- **Pico loop sensing**: throttle to zero and latched fault on loop open; **reset never restarts**; fresh SAFE-to-ARM edge; arm from zero throttle (A3, A8, A9).
- **Pendant link**: RS-485, framed packets with CRC, pendant-link watchdog, bounded throttle deltas (A3). Cheap once the pins are decided.
- **Coil suppression** on the contactor (A8).
- **Procedures**: session-start e-stop and guard check, containment inspection, scope-ground rule (A8, A15).
- **Procedural substitutes for Tier 2 hardware** (you are standing at the rig during Tier 1 manual tests):
  - **No hardware restart latch yet: recovery after any e-stop or guard-loop trip:**
    1. Disconnect the power module.
    2. Arm toggle to SAFE.
    3. Power-cycle the 24V (reboots the Pico; pull-downs hold every enable off meanwhile).
    4. Reset the mushroom / close the guard.
    5. Confirm the coil-energised LED stays **off** (coil drive indication, not proof of open contacts).
    6. Reconnect the module and run the full Tier 1 arm sequence. If Vbus rises unexpectedly on reconnection (before precharge is commanded), the arm sequence must not proceed: disconnect and investigate (possible welded contactor or precharge relay).
    If the coil LED comes on at any point outside the arm sequence, or the controller is unresponsive, leave the module disconnected and investigate.
  - **No guard lock yet:** to open the containment: throttle zero, SAFE, rotor visibly stopped, power module disconnected, bus verified discharged (bus-live LED off **and** meter check), then open.

**Tier 1 reduced arm sequence:**

```text
TIER 1 SAFE
  | guard closed (door contact in NC loop)
  | fresh SAFE-to-ARM edge, throttle = 0
  | Vbattery inside the 6S window (TIER_1_VBATT_WINDOW)
  | Vbus ~0 (plausibility)
PRECHARGE ON
  | Vbus reaches threshold of Vbattery within timeout
CONTACTOR ON
PRECHARGE OFF
MANUAL LOW-POWER MODE
  | pendant throttle, capped by TIER_1_MAX_OUTPUT (output ceiling, not a power limit)
  | any fault / loop open: throttle 0, contactor off, latched fault
  | low battery: CONTROLLED STOP
```

### Tier 1 commissioning plan (staged bring-up)

Tests the wiring actually built, not just the behaviour the firmware expects. Each stage passes before the next starts.

**Commissioning firmware build.** A separate compile-time build used only for Stage 0. It can energise the contactor coil and the precharge relay without the normal arm sequence, for wiring tests, and it enforces the **inverse** of the normal battery check: it refuses to energise anything if Vbattery is above about 1V. This check **supplements** physical disconnection; it does not prove it (a disconnected Vbattery sense wire reads about zero with a pack connected). Stage 0 therefore requires, **by procedure**: power module physically removed and the battery connector confirmed empty **before** the commissioning build is enabled. The firmware check catches a mistake; it is not the safeguard on its own.

**Stage 0: bench, no battery, no prop.**
Before any Stage 0 step: power module removed, battery connector confirmed empty, prop off.

1. **Divider and ADC input test** (relay and contactor operation disabled; normal Tier 1 firmware, not the commissioning build) with a current-limited bench supply: ramp the Vbattery and Vbus inputs through the 6S window and up to about 55V. Confirm readings are **accurate and monotonic** across the whole range, the ADC input stays inside its permitted range, **no clamp carries significant current**, and the firmware refuses to arm outside the 6S window. (Test the wrong-battery case with the bench supply, never by plugging in the wrong LiPo.) Afterwards: disconnect the bench supply and confirm the inputs and bus have discharged before moving on.
2. **E-stop path test** (commissioning build, battery module disconnected throughout):
   - Energise the contactor coil through its normal control circuit.
   - Press the **panel** mushroom: confirm the coil supply is removed and the contactor releases (audible / visible, and coil LED).
   - Repeat with the **pendant** mushroom, then by **opening the guard**, then by **unplugging the pendant cable**.
   - After each trip, reset it and confirm the contactor does **not** re-energise without a fresh arm sequence.
3. **Precharge relay and branch** (commissioning build, no supply on the battery side): with the relay commanded closed, measure the branch with a multimeter: it reads the precharge resistance (about 100 ohm); with the relay open, it reads open circuit. Confirm the thermal protection part is fitted in the path. No voltage on the battery side, so no conflict with the 1V check. The real charging curve at 6S is verified in Stage 1 with normal firmware.
4. **Pendant link**: CRC rejection, watchdog trip on unplug / ESP32 reset, throttle ceiling cannot be exceeded.
5. **Restart behaviour with the normal Tier 1 firmware** (battery still disconnected): the commissioning build's reset behaviour proves nothing about the normal firmware. Flash normal Tier 1 firmware; trip and reset each mushroom, the guard, and the pendant cable; confirm the coil LED never lights. (The normal firmware cannot complete an arm with no battery, since the 6S window fails; that is fine, the test is that nothing energises.)

**Stage 1: 6S, prop OFF, containment closed.**
- Motor direction, throttle response, signal-loss behaviour (DShot stops = motor stops), controlled stop, e-stop at low output.
- First low-energy scope capture of Vbus during an e-stop (stop and transient characterisation begins here).
- Low-battery stop and 6S window checked with a real pack.
- **Precharge charging curve at 6S** with normal firmware: Vbus reaches threshold within the timeout, precharge relay drops after contactor close.

**Stage 2: 6S, prop ON, containment closed and at full specification.**
- Very short, low-output runs first.
- Raise `TIER_1_MAX_OUTPUT` in steps, with the Vbus transient re-captured at each step before going higher.

### Tier 2: before full power, 12S, or automated sweeps
- **Run-down measurement** with a temporary tachometer: **before the guard-lock unlock logic is commissioned**, not before the first spin (the bare-bell test starts from no-load speed, so it is itself a powered test). Walk the bare motor up in steps inside the containment. Decides time delay vs standstill sensor (A8, A13).
- **Guard locking** (power-to-release) with the **UNLOCK button** and lock monitoring; unexpected unlock = safety trip (A8).
- **Hardware restart latch / master permit** with fail-safe Pico release (A8).
- **Dedicated current sensor** and **runtime limits** mapped to CONTROLLED STOP and SAFETY TRIP (A5, A7, A8).
- **Bus protection** (TVS, and ESC coast setting or dump resistor) sized from hard-disconnect characterisation (A13).
- **Fuse-contactor coordination** check once both parts are chosen (A13).
- **Undervoltage lockout** on the 12V contactor control rail (A8).
- **Final contactor** (100A class, mirror aux contact) replacing the V1 HFE82V-60B before full 12S power.
- **Sweep safety**: sweep execution on the Pico, Pi-link watchdog, arm-session nonce, stale-command rejection (A4).
- **Module ID** (at least a basic scheme) and pre-arm motor/prop confirmation; validation spin (A9).
- **Full arm sequence states** from A9 enabled as their hardware exists (restart latch, guard lock, module ID, validation spin).

### Tier 3: good practice, add at the PCB stage or when convenient
- **Precharge-complete hardware gate** with its own dividers, readback of the switched coil path, hysteresis (A8).
- **Redundant series precharge resistors** with the precharge-time window (A13).
- **Force-guided precharge relay** with VERIFY PRECHARGE OPEN (A9).
- **Contactor mirror auxiliary contact** for immediate weld detection (A7). Until then, weld detection relies on Vbus after spin-down.
- **Complementary (NO + NC) readbacks** on diagnostic channels (A14).
- **Standstill sensor**, only if the run-down measurement calls for it (A7).
- **Robust module ID** with integrity checking (A13 checklist).
- **Hold-to-run** pendant enable, if ever wanted (A13 checklist).
- **Hardware bus-transient peak detector** (A13).

Tier 3 items are not wrong or pointless: they protect equipment and close second-order failure paths. They are simply not what stands between the operator and a prop, a mains shock or a fire on a carefully operated first prototype.

---
---

# PART B: PROJECT LONGSHOT (the drone)

A single-12S, fully 3D-printed, load-bearing monocoque speed quad. The name fits the ambition: a long shot at real speed, and a long, slim airframe.

## B1. Goal

**Go fast.** 400+ km/h is a **floor, not a target**; find the ceiling over successive versions. The approach is method-led: instrumented testing, measured data, and a documented iterative process, rather than chasing a single lucky top-speed pass.

v1 target: a **credible, repeatable 400+ km/h** that sets up a later assault on top speed. Early flights only reaching ~300 km/h count as a pass.

## B2. Design philosophy

- **Single 12S**, for thermal/efficiency headroom, not for more speed.
- **Speed from prop pitch, not RPM.** Lower KV, higher pitch, calmer mechanics.
- **Weight discipline always on, but not the master constraint yet.** v1 is for function and measurability; weight optimisation comes later, with data.
- **Freeze the battery first as the design datum**, then wrap a minimal monocoque around it.
- **Fully 3D-printed load-bearing monocoque** (CF-filled filament, H2D), not a printed shell over carbon. This is the differentiator and the biggest unknown.
- **Reference builds are reference numbers, not templates.**
- **Test method:** run full 12S from the start, cap power in software (`motor_output_limit`, bench-verified props-off), and walk the limit up. One frozen battery/airframe acts as a controlled test article.

## B3. Powertrain spec

### Battery / power path
- **2x Gens Ace Tattu R-Line V6.0 6S 1600mAh 160C**, wired in series = 12S (~50.4V full charge). ~240g each.
- Native small high-C 12S packs effectively don't exist and are hard to get into Norway; series 2x 6S is the buyable answer. Long term, a custom pack built from cells is the likely path.
- Re-terminate each pack XT60 to **XT90**. Anti-spark on **one** pack only (the last one plugged in). Never two anti-spark connectors in a series loop.
- Series junction lives in a 3D-printed captured socket block, hardwired to the ESC pads on **8AWG**. No output connector in the high-current path.
- Re-termination technique: hot iron (SUGON T61), one lead at a time, heat-sink clip on the lead, never short.

### Stack (FC + ESC)
- **DAKEFPV H743 12S 120A 4-in-1** (x2 ordered: one flight, one for the rig). H743 over F722 for headroom, UARTs, logging.
- Betaflight, DJI direct plug, ICM42688P gyro, barometer, 16MB blackbox flash.
- **CONFIRMED with seller (2026-09-22):** main capacitor is **63V** (correct for 12S; comfortable margin over 50.4V in flight, where the battery always clamps the bus. On the rig, a hard contactor opening under power removes that clamp: see A13 hard-disconnect transient), and the **120A is per-channel** (~3.4x headroom over ~35A per-motor flight draw). The stack is a genuine 12S board and will not be the thermal limiter.

### Motors
- **Flight: AMAX 2820 1000KV x4.** 28mm stator, 3-12S rated, ~50k rpm no-load at 12S.
- Math-confirmed: ~1000KV on 12S with a high-pitch ~5.3" prop hits 400 km/h at a sane RPM with big thrust and current margin (~35A/motor).
- **All motors (flight and bench) have 5mm shafts.** One prop adapter / sleeve size covers every motor on the rig.
- **Bench comparison motors** (not for flight): T-Motor V3115 1050KV x4, T-Motor V3120 700KV x4, MAD 3120 1000KV x1.

### Props
- **HQProp 5.3x8, 2-blade, glass-fibre-reinforced nylon.** Both rotations: 5.3x8 (CW) + 5.3x8R (CCW), 2 of each per set.
- Glass-nylon is the right material class for near-transonic tips. Bench-verify for vibration before a full send.
- Diameter is a tuning lever.
- **Tip speed check (use helical tip speed, not rotation alone):** combine blade tip speed from rotation with forward speed. For 5.3" at 48k RPM: rotational tip ~338 m/s (~Mach 0.99); with 400 km/h (111 m/s) forward, helical tip ~356 m/s (**~Mach 1.04**). Target helical tip speed below about Mach 0.9, which for 5.3" at 400 km/h means roughly 40k RPM or less. This pushes further toward the "speed from pitch, not RPM" philosophy: more pitch, fewer RPM.
- **Candidate speed prop: APC 7x15, cropped.** The approach used by the Blackbird (B5): start from a very high-pitch prop that is too large and crop it down (Blackbird: 7x15 cropped to about 6", ~30k RPM for 600+ km/h).
  - **Genuine APC 7x15E (standard) and 7x15EP (pusher)** for both rotations. APC lists a **5mm bore** for the 7x15E, matching our 5mm shafts, so **no adapter ring should be needed** (the included LPAR18SF / LPARM12SF rings only reduce the bore, for smaller shafts). Verify on arrival with calipers or a 5mm pin gauge: snug with no play = fit directly; loose = APC ring or machined sleeve; tight = ream carefully to 5mm, never force it on. About 10g. Available in Europe at about EUR 4 each (Rotorama, Hoellein).
  - **Lookalikes** (e.g. "Vortex 7X15E/7X15R", 6.35mm bore, generic ring set) are for practising the cropping jig only, **never for flight** (unknown source and quality).
  - **If a ring is ever needed, it is the weak point at 30-40k RPM:** any play or eccentricity becomes imbalance. Rings must press in snugly with no play, and runout must be checked (dial indicator, or the rig's vibration data). Preferred for flight props: a **machined tight-fitting sleeve / prop adapter** (CNC), ideally with a machined spinner that doubles as the prop nut and centres the prop, as on the Blackbird.
  - **Cropping method (CNC):** hold the prop on its hub bore in a jig; cut both blades in the same setup, indexing 180 degrees, so they are identical; shape the tip rather than leaving a square cut; sharp cutter, light passes and air blast (glass-filled nylon melts easily); rebalance afterwards.
  - **Pitch after cropping:** nominal pitch is usually specified at about 75% radius, so a cropped prop's effective pitch is no longer exactly the label value. Work from pitch-speed calculation and measured flight data, not the label.
  - **RPM limit caveat:** APC publishes a maximum-RPM guideline for its thin electric props (recalled as roughly 145,000 / diameter in inches, i.e. about 21k RPM at 7" and 24k at 6"; **verify on APC's site**). Speed builds run these props beyond that guideline. Therefore every cropped prop is proof-spun on The Gauntlet, inside the containment, **above its planned flight RPM**, before it flies.

## B4. Airframe (the untouched frontier, next chapter)

Not started. Plan:
1. Freeze the battery datum (done: 2x R-Line 6S 1600 in series, long-slim in-line layout).
2. Wrap a minimal-frontal-area monocoque cross-section around it.
3. Set the aero / drag target.
4. Design cooling ducts (stack and DJI air unit are the two heat sources).
5. Map load paths into the print (motor mounts, wall thickness, print orientation along load lines).

Design to be **upgraded, not rebuilt**: cavity fits frozen pack and a future custom pack; mounts fit v1 motor and a likely v3 motor; instrumented from day one.

## B5. Reference builds (numbers only, not templates)

- **"SF drone"** (father-son world record, 657 km/h): fully 3D-printed PA6-CF airframe on an H2D. Confirms a printed monocoque survives well past our target. Sized for 650 km/h, a class above us.
- **DAVE_C 400 km/h build:** 8S, HQ 5.3x8 prop, printed shell over CNC carbon subframe. Our speed analog, but uses carbon structure.
- **"Blackbird" (Ben, 626 km/h average record, 655 km/h best run):** 14S (two custom 7S packs, over 60V full), AMAX 2826 motors (Sami's 557 km/h record used the 2820, our flight motor), **APC 7x15 cropped to about 6"** on CNC at ~30k RPM, APD 200F3 ESCs bussed side by side with a 3mm copper rod. Lessons worth taking:
  - **Build and iterate** rather than design the perfect drone first (his first prototype was "completely useless").
  - **Tip speed**: he cropped props specifically to keep helical tip speed below about Mach 0.9.
  - **Frame resonance**: a ~50 Hz vibration spike needed a gyro notch filter; gluing the frame stiffened it and invalidated the tune. A printed monocoque will have its own resonances that shift with material and print settings: plan to tune from blackbox data.
  - **Keep D gain above zero** (zero D caused a violent oscillation at ~450 km/h).
  - **Video dropout when flying past the pilot** (digital link) nearly lost the drone twice: plan antenna placement and pilot position for speed runs.
  - **Puller props** (props ahead of the arms: mass balance against flutter, clean air, motor cooling) and **Meredith-effect spinners** (machined aluminium, ducting air over the windings: lower drag and cooler motors) are worth considering for Longshot.
  - Speed claims are averaged over upwind and downwind runs.

---
---

# SHOPPING LIST

**Project Longshot, bought / in cart:**
- 2x Gens Ace Tattu R-Line V6.0 6S 1600 160C (battery)
- AMAX 2820 1000KV x4 (flight motors)
- T-Motor V3115 1050KV x4, V3120 700KV x4, MAD 3120 1000KV x1 (bench comparison)
- 2x DAKEFPV H743 12S 120A stack (flight + rig)
- HQProp 5.3x8 + 5.3x8R (props)
- **To order:** genuine APC 7x15E + 7x15EP, about 10 of each rotation (cropping experiments and proof tests)
- XT90 connectors (anti-spark + plain), 8AWG silicone wire, strip tool

**The Gauntlet, ordered from Electrokit:**
- Raspberry Pi Touch Display 2, 10.1" Portrait (base dashboard, DSI)
- Raspberry Pi AI Camera (IMX500)
- Adafruit seesaw I2C rotary encoder (throttle)
- 2.4" ILI9341 SPI TFT x1 (pendant)
- LEB22-1-C 22mm 1NC mushroom x2 (panel + pendant)
- Load cell 10kg (bare bridge)
- PT1000 RTD x3
- Olimex ADS1220
- LM339N DIP x2 (proto) + LM2901D SO-14 x2 (PCB stock)
- UF4007 diode x10, 1uF cap x4, 10k resistor x10
- Low-profile 2.54mm female header strips x5

**Still to source:**
- **V1 contactor: Hongfa HFE82V-60B/750-12-HL5** (chosen; check for genuine Hongfa label)
- **24V-to-12V DIN-rail DC-DC converter** (Mean Well DDR class, 12V output) for the contactor control rail
- Final contactor (100A class, 12V coil, mirror aux contact): before full 12S power
- Mean Well 24V DIN rail PSU (size to load: Pi + logic + relay coil + guard lock + margin; IT-network rated; built-in OVP)
- 24V-to-5V DC-DC converter (DIN rail or board mount, clean rail for Pi)
- Certified IEC C14 inlet module with double-pole switch and fusing suitable for IT networks (electrical box underside)
- Mains compartment cover, finger-safe terminals, PE terminal, inlet strain relief
- 5V rail overvoltage clamp / crowbar
- Independent standstill sensor (optical or Hall, logic-powered) -- only if run-down measurement shows it is needed; otherwise optional cross-check
- GX16 7-pin connectors x2 (panel-mount + cable-mount)
- 3m coiled cable (multicore: power, RS-485 pair (twisted), e-stop NC pair, shield)
- Thermal camera: FLIR Lepton + PureThermal breakout (export-controlled, source through proper channels) OR MLX90640 fallback
- RS-485 transceivers x2 (MAX3485 / SN65HVD class, 3.3V) for the pendant link
- Contactor requirement: mirror-type auxiliary contact (NO + NC if available) for position feedback
- Bus-powered bus-live indicator (LED + threshold + current limit)
- Bus protection: TVS for the fast edge, plus dump/brake resistor if ESC coast behaviour is not sufficient (size after characterisation)
- Divider input protection (series resistors, ADC clamps)
- Power-module DC fuses + holders (60V DC or higher interrupt rating; one per pack lead in series modules, one per branch on parallel modules; sized above max bench current)
- Optional secondary rig fuse at the enclosure XT90 inlet
- Precharge branch thermal protection (thermal cutoff or fusible resistor)
- Seal-in parts (pilot relay with holding contact, or discrete latch components; power-on-reset gating)
- Relay-driver MOSFET (logic-level, 12V coil compatible)
- Precharge: **two pulse-rated resistors in series** (each able to limit current alone) + **force-guided safety relay** (loop-powered coil) whose feedback contact proves the precharge switch opened. Candidate families (Omron G7SA, Finder 50) are **not** interchangeable: the exact variant must be checked against a DC breaking-capacity curve for the governing duty, which is the **worst permitted single-fault case: one series precharge resistor shorted, maximum battery voltage, worst-case resistor tolerance**. In the provisional example that is 50.4V across ~50 ohm, so **breaking ~1A at ~50V DC, resistive**. Normal duty is light (make at ~0.5A, break at near zero current once the contactor has equalised the bus). Resistor sized after bus capacitance is measured
- Dedicated current sensor for the battery path (Hall-effect, or high-side shunt + amplifier/isolation; no low-side shunt), sized for safety-trip current
- Precharge-complete gate parts (comparator stage with its own dividers, minimum-Vbattery condition, output readback, series switching element)
- Restart latch / master permit: pilot relay with holding contact, complementary feedback, and a fail-safe Pico release input
- Bus bleed resistor (sized for discharge time vs standby loss)
- Voltage-divider parts for Vbattery and Vbus sensing
- Dock ID contacts (pogo) + module ID chips (1-wire with CRC, or equivalent)
- Containment guard: **solenoid guard-locking interlock switch** (tongue actuator, spring-lock / power-to-release, dry positive-opening door contact, separate bolt-position monitoring contact, monitored key / tool escape release)
- Containment wall material (sized against fragment energy)
- Remaining PCB parts when phase 2 starts (JST-XH connectors etc.)

**Already owned / from home stock:**
- Raspberry Pi 5 (16GB) + official PCIe M.2 HAT+ + active cooler
- M.2 NVMe SSD (confirm NVMe/PCIe M-key, not SATA; 2230/2242 fit cleanly, 2280 may overhang)
- SHT31-D ambient temperature sensor
- BMP280 (Joy-It SEN-KY052) ambient pressure sensor
- Microcontrollers: 2x Pico, 2x Pico W, 2x ESP32-S3, 2x WROVER-CAM, assorted Arduinos
- Raspberry Pi(s) (earlier models)
- Switches (missile/arm/buttons), PSUs

---

# RULES / LESSONS CAPTURED

- **Shop by voltage first.** A voltage rating is a pass/fail gate before size or KV. Same-numbered parts exist in wildly different voltage classes.
- **Rated voltage is a proxy for RPM survivability, not an absolute.** Over-volting a 6S motor to 12S is standard; what matters is RPM within mechanical limits and managing heat.
- **Check the ESC capacitor voltage, not the amp rating.** 63V cap = 12S; 35V = 6S.
- **XT60 is undersized** for this current path; use XT90 or 8mm for the full loop.
- **Anti-spark on the last-plugged connection only**, never two in series.
- **E-stop = normally-closed, in series** (a broken wire also stops the motor). Fail-safe = de-energise-to-open.
- **DC relays: check the DC break-under-load rating at your voltage**, not the AC or carry number.
- **A relay/contactor coil needs appropriate transient suppression.** A plain flyback diode is not automatically the right answer when fast drop-out matters; final topology depends on the contactor's release time spec.
- **Read the grey/variant text on multi-option listings**, not the title.
- **MQTT is the data layer only.** Safety and real-time control go on direct deterministic links. The rig must stay controllable and stoppable even if MQTT, WiFi, or the Pi dies.
- **The Pico is the safety authority; the Pi is never in the safety/control path.** Linux is not real-time.
- **Safety authority never originates from a soft or touch surface.** Hardware arming and manual throttle originate only from the physical pendant (interpreted by the ESP32-S3 and forwarded over the RS-485 link; not hardwired, but no screen can initiate them). The local touchscreen may issue supervisory commands, including starting or stopping an automated sweep, but only after the Pico has accepted physical arming and all interlocks are satisfied. The phone/tablet is view-only. The true e-stop is hardwired with no software, MCU, or protocol in its path. No exceptions, no cheat codes.
- **Resetting an e-stop never restarts anything.** Reset clears the hardware; operation resumes only through a deliberate full re-arm.
- **An e-stop removes every commanded energy path, not just the obvious one.** Any route from battery to bus (contactor, precharge, anything added later) must be inhibited by the NC loop. Switches that fail closed are a separate fault class: detect them via Vbus, contain them (resistor limit, signal cut), latch a fault, refuse to arm.
- **Size precharge resistors by energy, not wattage.** The shorted-bus fault over the full timeout is the sizing case, and a failed-closed switch makes it continuous, so the branch needs thermal protection.
- **A contactor is a switch, not a fuse.** Every battery-fed circuit gets DC-rated fusing, at the source.
- **Anti-restart must not trust the MCU.** Hardware seal-in with an edge-coupled start; a stuck-high output cannot restart anything.
- **Every energy-path enable is fail-low in hardware.** No exceptions for new outputs.
- **Commands carry the arm session they belong to.** Anything from a previous arm session is rejected.
- **Fuse at the source.** A fuse protects only what is downstream of it; put the primary fuse next to the batteries, and fuse each parallel branch.
- **Never wait forever on telemetry.** Any sequence that waits for a measurement also has a timeout, and the timeout moves toward the safe state. **The safe state depends on the function:** for the contactor it is open; for the guard lock it is locked. A telemetry timeout never unlocks a guard; a **validated run-down delay** (measured worst case plus margin) is a designed condition, not a timeout, and may.
- **Proving standstill needs a sensor that survives the stop.** Anything powered by the bus disappears when the bus is disconnected.
- **Every build tier must run on its own.** If a tier's sequence needs hardware from a later tier, the tier list is wrong.
- **A firmware check supplements a physical safeguard; it never replaces it.** A sensor that reads zero may simply be disconnected.
- **Protection clamps are for faults, not for normal measurement.** Scale every measurement for the final configuration, not the first one.
- **Commission the wiring, not the firmware.** The first e-stop test proves the coil actually drops, with the battery physically disconnected so a wiring fault cannot spin anything.
- **A detector must survive the fault it detects.** The wrong-battery check has to withstand the wrong battery.
- **Assume the operator will plug in the wrong battery.** If a tier depends on a pack configuration, the firmware checks what it can (voltage window) and the document states what it cannot.
- **"Low power" is enforced, not promised.** A ceiling in firmware and a lower pack voltage, not a note in a procedure.
- **Break the coil side, not the converter input.** An e-stop that cuts a DC-DC converter's input leaves its output capacitors holding the coil in.
- **The Gauntlet exists to build Longshot.** Anything not needed for the next real measurement waits.
- **Software tiers follow hardware tiers.** CSV and one plot before databases, dashboards and auth.
- **A static bench characterises motors, not speed props.** High-pitch props are stalled at zero airspeed; choose them by pitch speed and flight data.
- **Modified props are proof-spun above flight RPM before they fly.**
- **Check helical tip speed** (rotation plus forward speed), not rotation alone.
- **Test the containment before trusting it.** It is the one failure you cannot fix and retest.
- **Apply the filter.** Projectile, dangerous voltage, or fire: requirement. Anything else: checklist or Tier 3. This is a hobby rig, not a certification exercise.
- **Never clip a mains-earthed scope ground to a non-ground node.** With 0V bonded to PE, the scope ground is battery negative.
- **Measure before designing around a physical assumption.** Run-down time decides the guard unlock method; neither intuition nor an AI estimate is the design basis.
- **Model every energy source in the box, not just the interesting one.** Mains gets its own domain, protection and FMEA rows.
- **Every FMEA row that says "detected by X" or "contained by X" needs a row for X failing.** Diagnostics and containment are components too; read them back as complementary pairs where possible. **The chain stops** when a failure is self-revealing (it fails toward the safe state and is noticed) or is covered by the session-start proof test.
- **Every energy-path driver gets a failed-short row.** A single shorted driver must not bypass precharge or the interlocks.
- **Commanded is not confirmed.** Every state change that matters (contactor closed, precharge open) is verified by a feedback that can actually tell, not inferred from a voltage that cannot.
- **Two stop classes.** Controlled stop spins down first; safety trip disconnects immediately. Evidence of uncontrolled torque always gets a safety trip.
- **Design the bus for the hard disconnect you cannot prevent.** Stop ordering reduces hard disconnects; it does not eliminate them.
- **Light off is not proof of dead.** Confirm with a measurement before touching.
- **A proof test must not rely on the function under test to prevent hazardous energy.** If the item has failed, the test itself must still be safe.
- **Every physical barrier written in the "containment" column gets its own row.** A wall, a lid, a fastener and a guard switch fail too.
- **Containment closed is an arm prerequisite, enforced in hardware.** The guard sits in the NC loop; the software never gets a vote.
- **When a fix changes a part's duty, re-check every requirement that referenced the old duty.** (The redundant precharge resistor doubled the relay's fault current.)
- **Every safety circuit you add is a new set of parts that can fail.** Give its components FMEA rows the day it is added, including the failure of the part that does the limiting (a resistor can short).
- **Redundancy needs detection, or it is only a delay.** A redundant element whose first failure is invisible degrades silently to a single point of failure.
- **Lock inputs when the request is accepted, not when it succeeds.** Everything the sequence uses must be immutable for the whole sequence.
- **Delaying permission is not removing it.** A gate that prevents early closure does nothing after closure; something independent must always be able to drop the energy path.
- **Independent means independent sensing too.** A hardware check sharing a sensor with the software check it backs up is not independent.
- **Latent failures need proof tests.** A safety contact that no longer opens is invisible until needed; test the e-stops and restart latch at the start of every session.
- **Watchdogs catch dead links, not lying ones.** Any control link needs framing and CRC; a long cable beside a motor gets a differential physical layer too.
- **Controlled software stops spin down first. Safety trips, hardware e-stops and involuntary control-power failures may disconnect under power, so the bus must survive the worst case.** A hard disconnect removes the battery clamp from a bus that a spinning motor can still pump.
- **Edge-triggered is not enough for a restart latch; it must also be power-up safe.** A rising supply is an edge too.
- **Act on edges, not levels, for maintained switches.** A toggle already in ARM after a fault or reboot is not a request to arm.
- **Measure what you compare.** If an interlock compares two voltages, both need a sensor the Pico reads directly.
- **One throttle source at a time.** Pendant manual or sweep, never both. A sweep ending goes to zero, not to the knob.
- **Three watchdogs, three names.** Pendant-link watchdog (pendant heartbeat over RS-485), Pi-link watchdog (Pi heartbeat, aborts sweeps), RP2040 internal watchdog (Pico firmware hang). Never write just "watchdog".
- **Control follows physical proximity to the hardware e-stop.** Touchscreen (next to the panel e-stop) = local supervisory control. Phone (no e-stop) = view only.
- **Export-controlled parts must be sourced through proper channels** and cleared before crossing a border.
- **The Pi runs network-isolated by default: its own AP, never joined to a restricted network.** USB-C power only; PoE was designed out.
- **Prototype before you fab a PCB.** DIP for proto, SMD for the board.
- **A load cell "with HX711 built in" is not a bare bridge cell.** For the ADS1220 you need the raw 4-wire bridge.
- **Verify a "4-channel 24-bit SPI ADC" is actually an ADS1220** (128x PGA), not an ADS1256 (64x).
- **Measure motor temp without ruining the motor.** Sensor in the rig fixture (spring-loaded, stator base), thermal paste not epoxy. The bell spins, the stator does not.
- **Split ambient sensing by job.** SHT31-D for accurate temperature (+-0.2C), BMP280 for pressure. Air density uses both.
- **One database engine for everything.** TimescaleDB covers relational run metadata and time-series sensor data. One connection string, one backup, proper SQL for both.
- **Wired beats wireless for a 3m pendant.** No battery, no link management, no wireless-link watchdog complexity; a simple heartbeat (pendant-link watchdog) remains over RS-485. The e-stop NC loop runs down the same cable.
- **Fix USB device names with udev rules.** Keyed to the USB serial number, not plug order. Always `/dev/gauntlet-pico`, never `/dev/ttyUSB0`.

---

*Design summary and decision log. Open work: airframe (Part B4), DC contactor sourcing, PSU and enclosure parts, rig physical build-out.*

---

# VERSION HISTORY

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-21/22 | Design worked out across an initial session: goals and philosophy; full Project Longshot powertrain (battery, stack, motors, props); The Gauntlet test rig (three-tier architecture, two-plane comms, full safety design, instrumentation, controls, power modules, PCB plan); names settled; DAKE stack confirmed; base compute settled (Pi 5 16GB, M.2 HAT+ NVMe, active cooler, 10.1" portrait touchscreen); Pi AI Camera (IMX500) added; PT1000 RTD motor temp via ADS1220, spring-loaded non-destructive mount. Style: no em dashes. |
| 1.1 | 2026-09-22 | Structure: The Gauntlet leads, Project Longshot follows (build order). Ambient sensing split: SHT31-D (temp) + BMP280 (pressure) as a pair. Pi 5, M.2 HAT+, NVMe, SHT31-D, BMP280 confirmed from stock. |
| 2.0 | 2026-09-22 | Full software stack decided (Docker Compose: Nginx, Mosquitto, TimescaleDB, FastAPI, Grafana). Live run page (FastAPI WebSocket + HTMX + Tailwind + DaisyUI). Access control split (touchscreen = full control, phone = view only). Settings page defined. Networking modes formalised (field AP / home). Pendant goes wired (3m coiled, GX16). Base ESP32 dropped. Physical enclosure: electrical box on tunnel side, DIN rail inside, Mean Well 24V PSU, operator panel on outside face (screen + e-stop + GX16). udev rules for stable device names. |
| 2.1-2.10 | 2026-09-22 | Design review pass. Authority model: pendant owns hardware arming and manual throttle; touchscreen is a local supervisory controller (sweeps only after physical arming, session-token auth, mutations require local authority); phone view-only. E-stop: NC loop mechanism documented, GX16 pin map, Pico loop sensing with latched fault, reset never restarts. Contactor energisation needs an intact loop plus Pico permission. Precharge made architectural (Vbus to ~95% before close); bus bleed and bus-side voltage sensing added. Sweeps execute on the Pico. Three named watchdogs (pendant-link, Pi-link, RP2040 internal); relay-enable fail-low via pull-down. Single throttle source; profile latched at arm. MQTT carries nothing that changes motor output. Open electrical decisions collected in A13. |
| 3.0 | 2026-09-22 | NC loop now inhibits the precharge path as well as the main contactor. Battery-side voltage sensor added (A7). Fresh SAFE-to-ARM edge required after startup or any fault (A3/A9). Stale Pi commands discarded (A4). Precharge resistor sized to survive a shorted bus for the full timeout (A13). Version history moved to the end; 2.1-2.10 collapsed into one row. |
| 3.1 | 2026-09-22 | Precharge resistor sizing rewritten in energy terms with worked example (A13). A4 e-stop line updated to cover both energy paths. A8 invariant narrowed to commanded energy paths; failed-closed switching elements (welded precharge switch, welded main contactor) documented as a separate detected-and-contained fault class. Rules updated to match. |
| 3.2 | 2026-09-22 | Main DC fuse added after the XT90 inlet (A1). Hardware anti-restart seal-in made mandatory, covering both energy paths, with an edge-coupled start input (A8). Fail-low extended to every energy-path enable, including precharge-enable. RP2040 internal watchdog serviced only from the main loop. Precharge closing threshold (>=95%, ~3 tau) separated from fault timeout (~5 tau + margin) (A9/A13). Failed-closed precharge branch gets thermal/energy-responsive protection (A13). Arm-session number on Pi commands (A4). Welded-contactor behaviour states its single-fault assumption (A8). |
| 3.3 | 2026-09-22 | Restart latch made independent of the main contactor (its aux contact is open during precharge) and power-up safe (A8). Contactor aux contact repurposed as position feedback for immediate weld detection (A7). Residual make-current guess replaced with a loop-resistance calculation to be measured (A13). New A13 item: hard-disconnect DC-bus transient (ESC braking, wiring inductance, back-EMF) against the 63V capacitor; B3 margin statement qualified. Software stops now spin down before opening the contactor (A8). Vbus weld decision waits for spin-down (A7). Time-delay fuse restored as a precharge-branch option (A13). Pendant link moved to RS-485 with framed, CRC-checked packets and bounded deltas; GX16 pins 3+4 now RS-485 A/B (A3). |
| 3.4 | 2026-09-22 | ESC drive path decided: the Pico drives the ESC directly with bidirectional DShot, giving a local RPM source; ESC telemetry wire to the Pico; FC not in the rig signal path (intro, A1, A2, A4, A7). Primary fusing moved into each power module, per branch on parallel modules; optional secondary rig fuse (A1, A10). Stop rule: RPM below threshold or spin-down timeout, whichever first (A8). A8 wording tightened (healthy contactor; precharge failure potentially sustained). Session-start proof test with restart-latch readback (A8). RS-485 implementation details (A12). New A14: failure mode table skeleton. |
| 3.5 | 2026-09-22 | A14 walked row by row. Arm sequence made explicit as states, including VERIFY CONTACTOR CLOSED before PRECHARGE OFF (A9); mirror aux contact now a contactor requirement (A7). Two stop classes: CONTROLLED STOP and SAFETY TRIP (A8). Involuntary hard-disconnect paths listed; bus clamp assumed necessary (A8, A13). Divider input protection and pre-precharge plausibility check (A7). Bus-powered bus-live indicator mandatory (A8). Undervoltage lockout given a defined job: prevent contactor chatter on 24V brownout (A8). Motor pole count in profiles; runtime limits per profile mapped to stop classes (A5). Fuse DC breaking capacity (A10). ESC firmware support for bidirectional DShot and telemetry as open dependency (A13). A14 rows added for diagnostic channels, UVLO, dividers, indicator, runtime limits. |
| 3.6 | 2026-09-22 | Hardware precharge-complete gate added in series with the contactor driver, so a failed-short driver cannot bypass precharge (A8). Precharge switch becomes a force-guided safety relay; VERIFY PRECHARGE OPEN added to the arm sequence (A9). Dedicated Pico-read current sensor for the overcurrent safety trip and true efficiency data (A7). Pre-arm motor/prop confirmation and first spin-up plausibility check; max RPM tied to motor + prop (A5, A9). Arm-session number must be boot-unique (A4). Pi-link trip triggers a controlled stop only during sweeps (A8). Contactor DC break rating must cover safety-trip current; protection hierarchy documented (A8). A14 rows for driver MOSFETs, gate, bus clamp, telemetry, PT1000/ADS1220, wrong profiles. FMEA chain-stopping rule and stale stop rule updated. |
| 3.7 | 2026-09-22 | Restart latch becomes the master safety permit with a fail-safe Pico release input, fixing the post-closure hold-in trap of a shorted contactor or precharge driver (A8). Precharge gate gets separate dividers, a minimum-Vbattery condition and output readback; gate only delays closure (A8). Proof-test safety rule; gate tested by readback, not by commanding closure (A8). Validation spin made a restricted state before full arming, with a broad plausibility envelope (A9). Per-pack fusing in series modules (A10). TVS vs dump resistor vs ESC coast separated by timescale; clamp health treated as latent (A13). Precharge relay rating defined by the fault-case duty (shopping list). A14 rows for the master permit, gate divider fault, series module short. |
| 3.8 | 2026-09-22 | Precharge resistance made two series resistors (single-fault protection against a resistor short), with a precharge-time window that detects the first failure (A13, A9). Gate readback now senses the switched coil path after the gate's own series switch; gate hysteresis and minimum dwell (A8). Profile snapshot locked when the arm request is accepted and bound to the arm-session number (A9, A4). Module ID supervised while armed (A9). Prop-shed trip uses Pico-local measurements only; camera annotation only (A14). Clamp re-test at reduced energy per the proof-test rule (A8, A13). Wrong-prop detection downgraded to a plausibility aid; human confirmation is the control (A9). A14 rows for precharge resistor short, gate switch short, bleed and indicator shorts, module ID loss. |
| 3.9 | 2026-09-23 | Containment guard switch added in series with the NC loop, making containment-closed an arm prerequisite in hardware; guard locking until spin-down and safe bus (A1, A8, A9). Containment tunnel analysed as a safety component: A14 rows for guard, guard switch, guard lock, walls, fasteners, openings; A13 item for fragment-energy design. Fuse-contactor protection coordination (A8, A13) and a fuse-fails-to-interrupt row. Precharge relay rated for the single-fault current (~1A). New A13 schematic-stage checklist: current sensing and star grounding, robust module ID, pendant valid-but-wrong commands (hold-to-run option). Pi wording: supervisory control only (A2). Precharge-too-fast wording covers low capacitance. |
| 3.10 | 2026-09-23 | New A15: mains supply and three electrical domains (IT-network double-pole switching and fusing, certified mains parts in a separate compartment, PE bonding, 0V bonded to PE, RCD, 5V overvoltage clamp) with A14 rows. Guard lock must stay locked on control-power loss (power-to-release); e-stop does not unlock it; closed and locked verified separately in the arm sequence; physical lock check at session start (A8, A9). Guard unlock method decided by a bench run-down measurement: validated time delay if run-down is short and repeatable (ISO 14119), logic-powered standstill sensor if it is long (eRPM disappears when the bus-powered ESC browns out); missing data never unlocks, a validated delay may (A7, A8). Guard switch must be a dry NC contact; solenoid guard-locking interlock switch recommended. Worst-credible-projectile design basis incl. detaching magnets; containment inspection at session start (A13). Arm nonce generated at the accepted arm request (A4). A14 rows for guard-lock driver, wiring, feedback, escape release, standstill sensor. |
| 3.11 | 2026-09-23 | Design basis added: hobby rig, not certification; every decision passes the projectile / dangerous voltage / fire filter; hazard calibration (12S is within the 60V DC extra-low-voltage band). New A16: build tiers (Tier 1 before first spin, Tier 2 before full power / sweeps, Tier 3 good practice). Guard unlocking made deliberate via an UNLOCK button in series with the release driver; unexpected unlock while armed = SAFETY TRIP (A8). Bare-bell run-down test starts from no-load speed (KV x max voltage), temporary tachometer, re-validate on any rotating-assembly change. Run-down figures marked as pre-test estimates. Fuses in positive conductors (A10). Crowbar needs a current-limited source; scope-ground rule (A15). |
| 3.12 | 2026-09-23 | Tier 1 made executable on its own: basic Vbattery + Vbus sensing and the bus-live LED moved into Tier 1; documented Tier 1 reduced arm sequence; enforced TIER_1_MAX_OUTPUT ceiling in the Pico; Tier 1 runs on a single 6S module (physical ceiling: about half the RPM, a quarter of the energy); build tier set in firmware, not the UI. Run-down measurement moved to before guard-lock commissioning (Tier 2). Procedural substitutes for the restart latch and guard lock during Tier 1. A9 marked as the full target sequence. |
| 3.13 | 2026-09-23 | Tier 1: 6S voltage window enforced in firmware (rejects any 12S pack; a discharged 8S can pass, so the 6S-only rule stays); low-battery controlled stop; precharge branch thermal protection moved into Tier 1; coil-energised LED; step-by-step recovery procedure after e-stop / guard trip; TIER_1_MAX_OUTPUT renamed as an output ceiling, not a power limit; low-energy stop/transient characterisation required before raising the ceiling. |
| 3.14 | 2026-09-23 | Tier 1 commissioning plan added (A16): commissioning firmware build that refuses to energise anything if Vbattery > ~1V; Stage 0 bench tests (divider/ADC with bench supply up to ~55V, e-stop path test proving the coil actually drops for panel, pendant, guard and cable unplug, precharge branch, pendant link); Stage 1 6S prop off; Stage 2 6S prop on with stepped ceiling. Coil LED described as coil-drive indication only; unexpected Vbus rise on reconnection blocks arming. Final contactor bought with the mirror aux contact from day one. Containment at full specification before the first prop run. |
| 3.15 | 2026-09-23 | Commissioning clarifications: the 1V check supplements physical battery removal rather than proving it; precharge branch tested with a multimeter (no battery-side voltage, no conflict with the 1V check), real charging curve moved to Stage 1; dividers scaled for 0-60V so the 55V test confirms accurate, monotonic readings with no clamp current (A7, A16); restart behaviour verified with the normal Tier 1 firmware as well as the commissioning build. Design phase closed; next: component selection. |
| 3.16 | 2026-09-23 | New section "Gauntlet V1: the minimum": first goal is a thrust/current/RPM curve for the AMAX 2820 on 6S; Tier 1 hardware with software Tier 0 (Pico and pendant firmware, USB serial to CSV, one plot; full software stack deferred); minimum dataset (thrust, RPM, current, voltage, motor temp, ambient for density); milestones with an early deliberate containment failure test; validation spin kept deliberately simple; scope guard; Longshot airframe work in parallel. |
| 3.17 | 2026-09-23 | V1 contactor chosen: Hongfa HFE82V-60B/750-12-HL5 (60A, 12V coil, no aux, non-polar, release 10ms or less); final 100A class contactor with aux before full 12S power. New 12V contactor control rail from a 24V-to-12V DIN DC-DC, feeding e-stop loop, latch, contactor and precharge coils; NC loop breaks the 12V side, never the converter input; UVLO moved to the 12V rail (A1, A8, A15, A16). A14 rows for the 24-to-12V converter failing high / low. |
| 3.18 | 2026-09-23 | Bench role narrowed: flight parity covers motor and ESC behaviour; a static bench cannot rank speed props; the bench proof-spins and balance-checks modified props. Longshot props: helical tip speed check (5.3" at 48k RPM and 400 km/h is ~Mach 1.04; target below ~0.9); APC 7x15E/EP cropping path with CNC method, adapter-ring fit and runout check (machined sleeve preferred), pitch caveat, RPM-limit caveat and mandatory proof spin; lookalikes for jig practice only (B3). Blackbird added to reference builds with lessons (B5). Shopping list: APC 7x15E/EP. |
