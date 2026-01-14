Physical Layer
##############

This document specifies the physical layer options for Fusain protocol
communication. For network-based transports, see :doc:`tcp` and :doc:`websocket`.


Overview
********

Fusain uses UART-based packet framing and supports multiple physical layer
transports. The protocol remains unchanged across all physical layers - transceiver
ICs handle the adaptation between Fusain packets and the physical bus. For
routing between physical layers, see :doc:`packet-routing`.

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 40

   * - Physical Layer
     - Baud Rate
     - Status
     - Use Case
   * - Plain UART
     - 230400
     - Development only
     - Lab bench testing, prototyping
   * - :term:`RS-485`
     - 115200
     - Production
     - Multi-appliance networks, long distances
   * - :term:`LIN`
     - 19200
     - Production (default)
     - Single-wire retrofits


Transceiver Architecture
************************

The MCU firmware operates identically regardless of physical layer. Transceiver
ICs handle all physical layer details:

.. code-block:: text

   MCU (Fusain firmware)
       |
       | UART (Fusain packets)
       |
   [Transceiver IC]
       |
       | LIN / RS-485 / None (plain UART)
       |
   Physical Bus

The transceiver IC handles:

- Physical layer signaling (LIN break/sync, RS-485 differential, etc.)
- Frame fragmentation/reassembly (for LIN's 8-byte frame limit)
- Fusain packet buffering and delivery
- Direction control (RS-485 DE/RE pins)


UART Configuration
******************

All physical layers use the same UART parameters at the MCU level:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Parameter
     - Value
   * - Data bits
     - 8
   * - Parity
     - None
   * - Stop bits
     - 1
   * - Flow control
     - None
   * - Baud rate
     - Physical layer dependent (see below)

**Baud Rates by Physical Layer**

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - Physical Layer
     - Baud Rate
     - Notes
   * - Plain UART
     - 230400
     - Direct MCU-to-MCU connection
   * - RS-485
     - 115200
     - Via RS-485 transceiver
   * - LIN
     - 19200
     - Via UART-to-LIN transceiver


Plain UART
**********

**Status:** Development and prototyping only

Plain TTL UART provides direct connection between devices for development and
testing. Not recommended for production due to limited distance and noise
immunity.

**Specifications**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Parameter
     - Value
   * - Maximum distance
     - ~15m (depends on cable quality and environment)
   * - Multi-drop support
     - No (point-to-point only)
   * - Noise immunity
     - Low
   * - Wiring
     - TX, RX, GND (3 wires minimum)

**Use Cases**

- Initial firmware development
- Protocol validation and debugging
- Bench testing with short cables


RS-485
******

**Status:** Production (multi-appliance networks)

RS-485 provides robust, differential signaling suitable for long distances and
multi-drop networks. Use RS-485 when:

- Multiple :ref:`appliances <fusain-device-roles>` share one bus
- Cable runs exceed 40m
- Environment has high electrical noise

**Specifications**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Parameter
     - Value
   * - Maximum distance
     - 1200m (ideal conditions)
   * - Typical distance
     - 100m (guaranteed with CAT5)
   * - Maximum nodes
     - 32-256 (depends on transceiver unit load)
   * - Noise immunity
     - High (differential signaling)
   * - Wiring
     - A, B, GND (twisted pair)


Transmit Enable Control
-----------------------

RS-485 transceivers require driver enable (DE) and receiver enable (RE) control.

- Assert DE before transmission begins
- Wait for UART TX shift register to empty before de-asserting DE
- RE is typically active-low and tied inverse to DE

**Timing Requirements**

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Parameter
     - Value
   * - Inter-packet gap
     - 500μs minimum (prevents collisions)
   * - Turn-around time
     - 1-10μs (transceiver dependent)


Multi-Drop Topology
-------------------

RS-485 supports multiple devices on a single bus.

**Daisy-Chain (Recommended)**

.. code-block:: text

   [Controller]---[Appliance 1]---[Appliance 2]---[Appliance 3]
       120Ω                                            120Ω

- Devices connected in series along the main bus
- Short stubs (<30cm) from bus to each device
- Termination resistors (120Ω) at both physical ends only
- Best signal integrity and maximum distance

**Star Topology (Not Recommended)**

Star topology creates reflections at branch points and is not recommended for
RS-485. If unavoidable, keep branch stubs under 30cm.


Termination
-----------

**Requirements**

- 120Ω resistors at both physical ends of the bus
- 1/4W or greater power rating
- Must match cable characteristic impedance

**Correct Termination**

.. code-block:: text

   [Controller]---[Appliance 1]---[Appliance 2]---[Appliance 3]
       120Ω                                            120Ω

**Incorrect Termination**

.. code-block:: text

   [Controller]---[Appliance 1]---[Appliance 2]---[Appliance 3]
       120Ω            120Ω            120Ω            120Ω
   (Problem: Over-termination causes signal attenuation)

   [Controller]---[Appliance 1]---[Appliance 2]---[Appliance 3]
       120Ω
   (Problem: Reflections from unterminated far end)


Cable Specifications
--------------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Parameter
     - Specification
   * - Cable type
     - Twisted pair (required)
   * - Characteristic impedance
     - 120Ω
   * - Recommended cables
     - Industrial RS-485 cable, CAT5/CAT6 (one pair)
   * - Avoid
     - Parallel untwisted wires

**Shielding**

- **Shielded (STP):** Recommended for industrial/noisy environments
- **Unshielded (UTP):** Acceptable for typical installations


Grounding and Biasing
---------------------

**Common-Mode Ground**

RS-485 is differential, but common-mode voltage must stay within ±7V of device
ground. Connect all device grounds together, or use isolated transceivers for
ground isolation up to 2.5kV.

**Fail-Safe Biasing**

When no drivers are active, the bus is in an undefined state. Add bias resistors
at the controller:

- A to +5V: 560Ω (pulls A high)
- B to GND: 560Ω (pulls B low)

This ensures the bus idles in a defined state.


Collision Avoidance
-------------------

Fusain requires software collision avoidance on RS-485. See
:doc:`communication-patterns` for polling mode usage.

1. **Discovery Phase**

   :ref:`msg-discovery-request` causes all appliances to respond. Each appliance
   MUST wait a random delay (0-50ms) before sending :ref:`msg-device-announce`.

2. **Normal Operation**

   Use addressing to send commands to specific appliances. Each appliance only
   responds when addressed.

3. **Polling Mode**

   For multi-appliance networks, use polling mode
   (:ref:`TELEMETRY_CONFIG <msg-telemetry-config>` with ``interval_ms=0``) to
   prevent broadcast collisions.


Node Capacity
-------------

RS-485 node limits depend on transceiver unit load (UL):

.. list-table::
   :header-rows: 1
   :widths: 25 25 25 25

   * - Transceiver Type
     - Unit Load
     - Max Nodes
     - Example ICs
   * - Standard
     - 1 UL
     - 32
     - MAX485, SN75176
   * - Low-power
     - 1/2 UL
     - 64
     - MAX3485
   * - High-density
     - 1/4 UL
     - 128
     - ADM2682E
   * - Ultra-high-density
     - 1/8 UL
     - 256
     - MAX22501E


Recommended Hardware
--------------------

**Transceivers**

- MAX485: Basic, low cost
- MAX3485: Low power, 1/2 unit load
- ADM2682E: Isolated, 1/4 unit load

**Cables**

- Belden 3105A: Industrial RS-485
- CAT5e: Budget option (use one twisted pair)

**Connectors**

- RJ45: Ethernet-style
- Phoenix Contact: Industrial terminal blocks


LIN
***

**Status:** Production (default for single-appliance networks)

LIN (Local Interconnect Network) is the default physical layer for Fusain in
production environments. Most heaters being retrofitted only have one wire
available for appliance-to-controller communication.

.. warning::

   LIN MUST NOT be used for multi-appliance networks. LIN's single-master
   architecture and limited collision handling make it unsuitable for multi-drop
   configurations. Use RS-485 for multi-appliance networks.

**Specifications**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Parameter
     - Value
   * - Maximum distance
     - ~40m
   * - Maximum nodes
     - 2 (Fusain restriction: 1 controller + 1 appliance)
   * - Noise immunity
     - Medium-high
   * - Wiring
     - LIN (single wire) + GND
   * - Termination
     - Built into transceiver ICs


Transceiver Architecture
------------------------

The Fusain protocol runs unchanged at the MCU level. A UART-to-LIN transceiver
IC handles all LIN physical layer requirements:

.. code-block:: text

   Controller MCU                          Appliance MCU
       |                                        |
       | UART (19.2 kbaud)                      | UART (19.2 kbaud)
       |                                        |
   [UART-to-LIN IC]                        [UART-to-LIN IC]
       |                                        |
       +--------------- LIN Bus ----------------+
              (single wire + ground)

**Transceiver Responsibilities**

- **Voltage Translation:** Converts UART logic levels to LIN bus levels
- **Slew Rate Control:** Limits signal transitions for EMC compliance
- **Wake/Sleep:** Handles LIN bus wake-up and sleep modes


Recommended Transceiver ICs
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 20 20 40 20

   * - IC
     - Interface
     - Features
     - Use Case
   * - TJA1027
     - UART
     - Transparent UART-to-LIN, low standby current, automotive-grade
     - **Recommended**
   * - TLIN1029
     - UART
     - Diagnostic features, integrated regulator
     - Industrial
   * - MCP2003B
     - UART
     - Low-cost, basic functionality
     - Budget builds
   * - NCV7321
     - SPI
     - SPI-controlled, flexible
     - Complex systems


Performance Characteristics
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - Metric
     - Value
     - Notes
   * - Baud rate
     - 19.2 kbaud
     - LIN standard
   * - Telemetry interval
     - 500ms minimum
     - See :ref:`link-bandwidth-requirements`


Configuration
-------------

1. **Baud Rate:** 19.2 kbaud at MCU UART

2. **Telemetry Intervals:** 500ms minimum. See :ref:`link-bandwidth-requirements`.

3. **Discovery:** Random delay (0-50ms) MUST be applied before responding to
   :ref:`DISCOVERY_REQUEST <msg-discovery-request>` for protocol consistency.


Wiring
------

- **LIN Bus:** Single wire (12V nominal, automotive-grade wire)
- **Ground:** Separate ground return
- **Termination:** Built into most transceiver ICs (no external resistors)
- **Connectors:** Automotive-grade (Deutsch, AMP, or equivalent)


Benefits
--------

- **Single-wire topology:** Ideal for retrofits with limited wiring
- **Automotive-grade reliability:** Proven in harsh environments
- **Built-in collision avoidance:** LIN centralized scheduling
- **Lower cost:** Simpler than RS-485
- **Standard compliance:** LIN 2.0+ specification


Comparison Summary
******************

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 20 20

   * - Feature
     - Plain UART
     - RS-485
     - LIN
     - Notes
   * - Distance
     - ~15m
     - ~1200m
     - ~40m
     -
   * - Baud rate
     - 230400
     - 115200
     - 19200
     -
   * - Multi-drop
     - No
     - Yes (32-256)
     - No
     - Fusain restriction
   * - Wiring
     - 3-wire
     - 2-wire differential
     - 1-wire + GND
     -
   * - Noise immunity
     - Low
     - High
     - Medium-high
     -
   * - Recommended use
     - Development
     - Production
     - Production
     -
   * - Typical use case
     - Lab testing
     - Multi-appliance
     - Single-wire retrofit
     -


Recommendations
***************

Production Deployments
----------------------

1. **Single-wire retrofits (most common):** LIN

   - Use TJA1027 or equivalent transceiver
   - Configure 19.2 kbaud at MCU UART
   - Set telemetry interval to 500ms minimum
   - Ideal for heater retrofits with existing single-wire infrastructure

2. **Multi-appliance networks or long distances (>40m):** RS-485

   - Implement DE/RE control in firmware
   - Use daisy-chain topology with 120Ω termination at both ends
   - Add random delays (0-50ms) to DISCOVERY_REQUEST responses
   - Use polling mode for multi-appliance networks

Development and Prototyping
---------------------------

- Use plain UART for initial development
- Direct UART connection between devices
- Suitable for bench testing and firmware development


Protocol Configuration Summary
******************************

.. list-table::
   :header-rows: 1
   :widths: 25 25 25 25

   * - Environment
     - Physical Layer
     - Baud Rate
     - Telemetry Interval
   * - Production (single-appliance)
     - LIN
     - 19200
     - 500ms
   * - Production (multi-appliance)
     - RS-485
     - 115200
     - 100-500ms
   * - Development
     - Plain UART
     - 230400
     - 100ms

All configurations use identical packet format. See :doc:`packet-format` for
framing, CRC, and addressing. See :doc:`implementation` for buffer sizing and
timeouts.
