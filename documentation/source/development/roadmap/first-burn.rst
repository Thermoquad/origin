First Burn
##########

:Target: Complete first diesel heater burn cycle with Helios firmware
:Status: Planning

Overview
********

The **First Burn** milestone represents a critical project checkpoint: executing
a complete diesel heater burn cycle using Helios firmware with validated,
real-world parameters.

Before Helios can safely control the heater, we must extract accurate hardware
parameters from the original "Universal Diesel Air Heater Controller" and verify
Helios behavior matches expectations.

Hardware Architecture
*********************

.. code-block:: text

   ┌──────────────────┐
   │   Test Stand     │◄── USB Serial ──┐
   │     Utility      │                 │
   │      (Go)        │                 │
   ├──────────────────┤                 │
   │ - GoCV (webcam)  │                 │
   │ - Prometheus     │                 │
   │ - Local logging  │                 │
   └────────┬─────────┘                 │
            │                           │
            ▼                           │
   ┌─────────────────┐          ┌───────▼──────┐
   │     Grafana     │          │     Stan     │
   │  Visualization  │          │   (Pico 2W)  │
   └─────────────────┘          │   Firmware   │
                                └───────┬──────┘
                                        │ GPIO/ADC
                                        │
                                ┌───────▼──────┐
                                │  Controller  │
                                │  (Original   │
                                │  OR Helios)  │
                                └───────┬──────┘
                                        │
                                ┌───────▼──────┐
                                │    Heater    │
                                │  Components  │
                                └──────────────┘

**Components:**

- **Stan** - Test stand firmware (Pico 2W) that captures signals and can
  generate fake signals for verification
- **Test Stand Utility** - Go application using GoCV for webcam capture,
  receives data from Stan, correlates with CV-read temperatures, logs data,
  and exposes Prometheus metrics
- **Grafana** - Visualization of captured data via Prometheus endpoint

Project Structure
*****************

.. code-block:: text

   apps/stan/
   ├── firmware/     # Zephyr firmware for Pico 2W
   ├── tooling/      # Go utility with GoCV
   └── CLAUDE.md

Parameters to Extract
*********************

The following parameters in the diesel fuel profile are currently estimates.
We need to extract accurate values from the original controller.

**Thermistor Lookup Table**

Helios's current thermistor table is only accurate to 150°C. The heater operates
up to 275°C (emergency stop threshold). We need a complete °C-to-ohms mapping.

Approximate Steinhart-Hart constants for validation (R25 ≈ 45kΩ):

.. code-block:: text

   A = 0.8546971786e-3
   B = 2.190553900e-4
   C = 1.237951846e-7

These can be used to cross-check extracted values against predicted resistance.

Extraction method:

1. Observe voltage across thermistor (bypassing controller)
2. Voltage divider on Stan scales 5V to 3.3V for RP2350 ADC
3. Correlate ADC readings with temperature displayed on manufacturer app
4. Convert voltage readings to resistance
5. Build complete lookup table

Voltage divider circuit (3x 2.2kΩ resistors):

.. code-block:: text

   Thermistor node (0-5V)
           │
          ┌┴┐
          │ │ 2.2kΩ (R1)
          └┬┘
           ├──────► ADC input (0-3.33V)
          ┌┴┐
          │ │ 2.2kΩ ┐
          └┬┘       │ R2 = 4.4kΩ
          ┌┴┐       │
          │ │ 2.2kΩ ┘
          └┬┘
           │
          GND

**Motor RPM**

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - Parameter
     - Current Estimate
     - Extraction Method
   * - Ignition RPM
     - 2500
     - Intercept tach signal
   * - Preheating RPM
     - 2800
     - Intercept tach signal
   * - Cooldown RPM
     - 2500
     - Intercept tach signal
   * - Min/Max heating RPM
     - Unknown
     - Intercept tach signal

Motor tach signal: normally high, pulled low once per rotation.

**Pump Timing**

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - Parameter
     - Current Estimate
     - Extraction Method
   * - Ignition pulse duration
     - Unknown
     - Monitor pump signal
   * - Ignition pulse period
     - 500ms
     - Monitor pump signal
   * - Preheating pulse duration
     - Unknown
     - Monitor pump signal
   * - Preheating pulse period
     - 250ms
     - Monitor pump signal

Chinese controllers only expose period (in Hz). Helios treats duration and
period as separate configurable parameters. We need to extract both.

Pump continuity check: 10K resistor across contacts fakes pump being plugged in.

**Glow Plug Timing**

- Warmup duration before fuel injection (estimated 60s)
- On/off transitions during cooldown phase

**Temperature Thresholds**

Once we have accurate thermistor readings, we can observe when the original
controller transitions between phases and extract the actual temperature
thresholds.

Stan Serial Protocol
********************

Stan uses a simple text protocol over USB serial.

**Streaming (Stan → Utility):**

.. code-block:: text

   THERM:0.523,TACH:1234,PUMP_DUR:45,PUMP_PER:250,GLOW:0\n

**Commands (Utility → Stan):**

.. code-block:: text

   SET_THERM:0.400\n

Used during verification phase to simulate temperatures via PWM.

Prometheus Metrics
******************

The test stand utility exposes individual metrics:

- ``thermistor_voltage`` - Raw ADC voltage reading
- ``thermistor_temperature`` - CV-read temperature from app
- ``motor_rpm`` - Calculated from tach pulses
- ``pump_duration_ms`` - Pulse duration
- ``pump_period_ms`` - Time between pulses
- ``glow_plug_state`` - On/off (1/0)

Prioritized Work Order
**********************

Issues organized by dependency tier. Work on any issue in a tier once its
dependencies (previous tiers) are complete. Issues in the same tier can be
worked in parallel.

.. list-table::
   :header-rows: 1
   :widths: 10 20 50 20

   * - Tier
     - Issue
     - Description
     - Depends On
   * - 0
     - `stan#1 <https://github.com/Thermoquad/stan/issues/1>`_
     - Test Stand Hardware
     - —
   * - 0
     - `stan#7 <https://github.com/Thermoquad/stan/issues/7>`_
     - Monitoring Stack (Docker)
     - —
   * - 0
     - `heliostat#1 <https://github.com/Thermoquad/heliostat/issues/1>`_ ✅
     - Packet encoder
     - — *(Completed 2026-01-15)*
   * - 0
     - `slate#1 <https://github.com/Thermoquad/slate/issues/1>`_
     - WebSocket server
     - —
   * - 1
     - `stan#2 <https://github.com/Thermoquad/stan/issues/2>`_
     - Stan Firmware
     - stan#1
   * - 1
     - `heliostat#2 <https://github.com/Thermoquad/heliostat/issues/2>`_ ✅
     - Command builders
     - heliostat#1 *(Completed 2026-01-15)*
   * - 2
     - `stan#3 <https://github.com/Thermoquad/stan/issues/3>`_
     - Test Stand Utility
     - stan#2
   * - 2
     - `stan#6 <https://github.com/Thermoquad/stan/issues/6>`_
     - Verification Support
     - stan#2
   * - 2
     - `heliostat#3 <https://github.com/Thermoquad/heliostat/issues/3>`_
     - Controller mode
     - heliostat#1, #2
   * - 3
     - `stan#5 <https://github.com/Thermoquad/stan/issues/5>`_
     - Parameter Extraction
     - stan#1, #2, #3, #7
   * - 4
     - `stan#4 <https://github.com/Thermoquad/stan/issues/4>`_
     - Data Post-Processing
     - stan#5
   * - 5
     - `helios#1 <https://github.com/Thermoquad/helios/issues/1>`_
     - Update Helios Parameters
     - stan#4
   * - 6
     - `helios#2 <https://github.com/Thermoquad/helios/issues/2>`_
     - Helios Verification
     - helios#1, stan#6
   * - 7
     - `helios#3 <https://github.com/Thermoquad/helios/issues/3>`_
     - First Burn Execution
     - helios#2, heliostat#3, slate#1

Milestones
**********

Ready To Burn
=============

All hardware, software, and parameter extraction must be complete.

Task 1: Test Stand Hardware
---------------------------

`stan#1 <https://github.com/Thermoquad/stan/issues/1>`_

Design and build the breadboard circuit for parameter extraction.

- Voltage divider to scale thermistor voltage (5V) to 3.3V for RP2350 ADC
- Motor tach signal interception (normally high, pulled low per rotation)
- Pump signal monitoring (duration and period measurement)
- Pump continuity fake (10K resistor)
- Glow plug state detection (on/off)

Task 2: Stan Firmware
---------------------

`stan#2 <https://github.com/Thermoquad/stan/issues/2>`_

Create ``apps/stan/firmware/`` for Pico 2W.

- USB serial communication (no WiFi)
- Continuous data streaming (THERM, TACH, PUMP_DUR, PUMP_PER, GLOW)
- Thermistor ADC reading
- Motor tach pulse counting and RPM calculation
- Pump pulse duration and period measurement
- Glow plug state detection

Task 3: Test Stand Utility
--------------------------

`stan#3 <https://github.com/Thermoquad/stan/issues/3>`_

Create ``apps/stan/tooling/`` - pure Go utility using GoCV.

- USB serial communication with Stan
- Webcam capture using GoCV
- OpenCV temperature reading from manufacturer app display
- Data correlation (thermistor voltage ↔ CV temperature)
- Local file logging (CSV)
- Prometheus endpoint with individual metrics

Task 4: Parameter Extraction
----------------------------

`stan#5 <https://github.com/Thermoquad/stan/issues/5>`_

Run the original controller through full burn cycles.

Uses `stan#7 <https://github.com/Thermoquad/stan/issues/7>`_ (Monitoring Stack) for visualization.

- Connect original controller to test stand
- Point webcam at phone running manufacturer Bluetooth app
- Execute full burn cycles (preheat → ignition → heat → cooldown)
- Capture all data via test stand utility
- Visualize in Grafana during extraction
- Repeat as needed for consistent data

Task 5: Data Post-Processing
----------------------------

`stan#4 <https://github.com/Thermoquad/stan/issues/4>`_

Process captured data into usable parameters.

- Build °C-to-ohms thermistor lookup table
- Document motor RPM at each phase
- Document pump duration and period at each phase
- Document glow plug timing
- Identify temperature thresholds from phase transitions
- Update diesel fuel profile with accurate values

Task 6: Update Helios Parameters
--------------------------------

`helios#1 <https://github.com/Thermoquad/helios/issues/1>`_

Apply extracted parameters to Helios firmware.

- Update thermistor lookup table (extend to 275°C+)
- Update motor RPM defaults
- Update pump timing defaults
- Update temperature thresholds
- Verify configuration can be applied via Fusain commands

Task 7: Helios Verification
---------------------------

`helios#2 <https://github.com/Thermoquad/helios/issues/2>`_

Verify Helios behavior using the test stand with simulated signals.

Requires `stan#6 <https://github.com/Thermoquad/stan/issues/6>`_ (Stan Verification Support).

- Connect Helios (on Pico 2) to test stand
- Stan generates fake thermistor signals via PWM
- Simulate temperature sequences through burn phases
- Verify Helios state transitions match expectations
- Verify motor RPM targets
- Verify pump timing
- Verify glow plug behavior
- Document any discrepancies

Task 8: Heliostat Controller Mode
---------------------------------

`heliostat#1 <https://github.com/Thermoquad/heliostat/issues/1>`_ ✅ *Completed 2026-01-15*,
`heliostat#2 <https://github.com/Thermoquad/heliostat/issues/2>`_ ✅ *Completed 2026-01-15*,
`heliostat#3 <https://github.com/Thermoquad/heliostat/issues/3>`_

Add controller capability to Heliostat for sending commands to Helios.

- ✅ Add packet encoder to pkg/fusain (inverse of decoder) — **heliostat#1**
- ✅ Add command builder functions (STATE_COMMAND, PING_REQUEST, etc.) — **heliostat#2**
- Add controller mode with interactive TUI — **heliostat#3**
- Support serial (direct) and WebSocket (via Slate) connections
- Implement automatic keep-alive (PING_REQUEST)
- Add Prometheus metrics endpoint for Grafana

Task 9: Slate WebSocket Bridge
------------------------------

`slate#1 <https://github.com/Thermoquad/slate/issues/1>`_

Add WebSocket server to Slate for bridging Heliostat to Helios.

- WebSocket endpoint at port 80 (``/fusain``)
- Bidirectional packet forwarding (WebSocket ↔ Serial)
- Support multiple concurrent clients
- mDNS responder (``thermoquad-XXXX.local``)
- WiFi credential storage (NVS)
- Shell command ``wifi_save <ssid> <password>``

Task 10: Stan Verification Support
----------------------------------

`stan#6 <https://github.com/Thermoquad/stan/issues/6>`_

Add verification mode to Stan for simulating thermistor signals.

- Parse SET_THERM command from utility
- Generate PWM output for fake thermistor voltage
- Low-pass RC filter circuit (PWM → analog)
- Required for Task 7 (Helios Verification)

Task 11: Monitoring Stack
-------------------------

`stan#7 <https://github.com/Thermoquad/stan/issues/7>`_

Set up Prometheus and Grafana for data visualization.

- Docker Compose configuration
- Persistent volumes for test data between restarts
- Prometheus scrape config for test stand utility
- Grafana dashboard for parameter extraction
- Used during Task 4 (Parameter Extraction)

First Burn
==========

Execute the actual burn with Helios controlling real hardware.

Task 12: First Burn Execution
-----------------------------

`helios#3 <https://github.com/Thermoquad/helios/issues/3>`_

- Connect Helios to real heater (not test stand)
- Use Heliostat to control and monitor
- Execute full burn cycle (preheat → ignition → heat → cooldown)
- Capture all telemetry throughout burn
- Document results and observations
- Analyze data for issues
- Create burn report in documentation

Success Criteria
****************

The First Burn milestone is successful when:

1. **Parameters extracted** - Complete thermistor table (to 275°C+), accurate
   RPM values, pump timing, and temperature thresholds documented

2. **Helios verified** - All state transitions, motor control, pump timing,
   and glow plug behavior validated against test stand

3. **Complete cycle** - Heater completes preheat, ignition, stable heat, and
   cooldown phases without emergency stops or failures

4. **Telemetry captured** - All data logged throughout the burn with no gaps

5. **Documented** - Burn results, observations, and analysis recorded

GitHub Tracking
***************

Progress is tracked via GitHub milestones and issues:

- **Milestone:** `Ready To Burn <https://github.com/Thermoquad/origin/milestone/5>`_
- **Milestone:** `First Burn <https://github.com/Thermoquad/origin/milestone/6>`_
