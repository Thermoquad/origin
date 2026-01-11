Communication Patterns
######################

This document describes the communication patterns used in the :term:`Fusain`
protocol. For implementation details including timeouts and buffer handling,
see :doc:`implementation`.


Command-Response
****************

:ref:`Controllers <fusain-device-roles>` send commands; :ref:`appliances <fusain-device-roles>` may respond with errors.

.. code-block:: text

   Controller → Appliance:  [STATE_COMMAND: Set HEAT mode]
   Appliance → Controller:  (Success: no response)
                            OR
                            [ERROR_INVALID_CMD: Invalid parameter]

**Key Points:**

- Successful commands do NOT receive an acknowledgment
- Only errors trigger a response
- See :ref:`Control Commands <acknowledgment-philosophy-control>` for rationale


Periodic Telemetry
******************

Appliances broadcast telemetry at configured intervals when enabled.

.. code-block:: text

   Controller → Appliance:  TELEMETRY_CONFIG (enable=1, interval_ms=100)

   [After enabling, appliance broadcasts at configured interval:]

     Every 100ms:   MOTOR_DATA (per motor) + TEMPERATURE_DATA (per sensor)
     Every 250ms:   STATE_DATA (2.5× interval)

**Broadcast Mode Timing:**

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - Interval
     - Motor/Temperature Rate
     - State Rate
   * - 100ms
     - Every 100ms
     - Every 250ms
   * - 500ms
     - Every 500ms
     - Every 1250ms

For appliances with multiple motors or sensors, separate messages are sent for
each peripheral.

**Important:**

- Telemetry is **disabled by default** on boot
- Controller MUST explicitly enable with :ref:`msg-telemetry-config`
- Telemetry auto-disables when communication timeout elapses


Event-Driven Updates
********************

Some messages are sent when events occur, not on a timer.

.. code-block:: text

   PUMP_DATA:  Sent on pump cycle events (start, pulse end, cycle end)
   GLOW_DATA:  Sent when glow plug turns on or off

**Behavior:**

- Event-driven messages are only sent when telemetry is enabled
- In polling mode (``interval_ms=0``), event-driven messages are NOT sent
  automatically—use :ref:`msg-send-telemetry` to request them


Heartbeat
*********

Controllers check connectivity with periodic pings.

.. code-block:: text

   Controller → Appliance:  PING_REQUEST
   Appliance → Controller:  PING_RESPONSE (with uptime)

**Purpose:**

- Verify appliance is responsive
- Reset communication timeout timer on appliance
- Maintain subscription timeouts on routers

**Recommended Interval:** 10–15 seconds (well below default 30-second timeout)


Timeout Mode
************

Appliances automatically transition to safe state if communication is lost.

.. code-block:: text

   Normal operation:
     Controller → Appliance:  PING_REQUEST (every 10-15 seconds)
     Appliance → Controller:  PING_RESPONSE
     Appliance → Controller:  [Telemetry broadcasts continue]

   Timeout condition (30 seconds without PING_REQUEST):
     Appliance: Automatically transitions to IDLE mode
     Appliance: Automatically disables telemetry broadcasts
     [No further telemetry until controller re-enables]

**Default Behavior:**

- Timeout mode is **enabled by default** (30-second timeout)
- Only :ref:`msg-ping-request` resets the timer (other commands do not)
- See :ref:`msg-timeout-config` to configure

**Safety Rationale:**

- Ensures appliance doesn't operate indefinitely without supervision
- Critical for burner systems where communication loss requires safe shutdown
- IDLE mode performs proper cooldown if temperature is elevated
- **Exception:** E_STOP suspends timeout behavior; see :ref:`impl-estop`


Polling Mode
************

Controllers explicitly request telemetry instead of receiving broadcasts.

.. code-block:: text

   Controller → Appliance:  TELEMETRY_CONFIG (enable=1, interval_ms=0)

   [Telemetry enabled but not broadcasting. Controller must poll:]

   Controller → Appliance:  SEND_TELEMETRY (type=MOTOR, index=0xFFFFFFFF)
   Appliance → Controller:  MOTOR_DATA (motor 0)
   Appliance → Controller:  MOTOR_DATA (motor 1)  [if multiple motors]

   Controller → Appliance:  SEND_TELEMETRY (type=STATE, index=0)
   Appliance → Controller:  STATE_DATA

**Use Cases:**

- Multi-appliance networks (prevents broadcast collisions on RS-485). See
  :doc:`physical-layer` for RS-485 configuration.
- Bandwidth-constrained links (request only needed data)
- Power-sensitive applications (reduces transmissions)

**Important:**

- :ref:`msg-send-telemetry` does NOT reset communication timeout
- Controller MUST still send periodic :ref:`msg-ping-request`


Controller Routing
******************

Controllers can route packets between physical layers. See :doc:`packet-routing`
for detailed routing documentation.

.. code-block:: text

   Remote Controller (WiFi)
       |
       | Fusain over WiFi
       |
   Router Controller (WiFi + LIN)
       |
       | Fusain over LIN
       |
   Appliance (LIN)

**Summary:**

1. :ref:`Client <fusain-device-roles>` connects to router and performs discovery handshake
2. Client subscribes to appliance data via :ref:`msg-data-subscription`
3. Client sends commands with appliance address; router forwards to bus
4. Router forwards telemetry from appliance to subscribed clients
