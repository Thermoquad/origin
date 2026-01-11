Message Types
#############

This document specifies the behavior, validation, and semantics of each Fusain
message type. For payload encoding details, see :doc:`packet-payloads`. For
communication patterns, see :doc:`communication-patterns`.

.. note::

   Field types in this document (e.g., ``bool``, ``int``, ``decimal``, ``enum``)
   represent high-level types as decoded in implementation structs. For the
   type mapping between high-level and wire-level types, see :doc:`overview`.
   For wire-level encoding of each field, see :doc:`packet-payloads`.


Configuration Commands (0x10–0x1F)
**********************************

Configuration commands are sent from :ref:`controllers <fusain-device-roles>` to :ref:`appliances <fusain-device-roles>` to set
persistent parameters. Changes SHOULD persist across power cycles.

.. _acknowledgment-philosophy-config:

**Acknowledgment**

Configuration commands do NOT receive explicit acknowledgments. Controllers
SHOULD maintain the desired configuration state and periodically send commands
to reconcile any differences. This provides built-in retry capability.


.. _msg-motor-config:

MOTOR_CONFIG
------------

Configure motor controller parameters including PWM, PID gains, and RPM limits.

| **Payload:** :ref:`MOTOR_CONFIG <payload-motor-config>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - motor
     - int
     - Motor index (0 to motor_count-1)
   * - pwm_period (optional)
     - int
     - PWM period in nanoseconds. If absent, use firmware default.
   * - pid_kp (optional)
     - decimal
     - Proportional gain for :term:`PID controller`. If absent, use firmware default.
   * - pid_ki (optional)
     - decimal
     - Integral gain for :term:`PID controller`. If absent, use firmware default.
   * - pid_kd (optional)
     - decimal
     - Derivative gain for :term:`PID controller`. If absent, use firmware default.
   * - max_rpm (optional)
     - int
     - Maximum achievable RPM. If absent, use firmware default.
   * - min_rpm (optional)
     - int
     - Minimum stable RPM. If absent, use firmware default.
   * - min_pwm_duty (optional)
     - int
     - Minimum PWM pulse width in nanoseconds. If absent, use firmware default.

**Default Values**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Field
     - Default
   * - pwm_period
     - 50000 ns (20 kHz)
   * - pid_kp
     - 4.0
   * - pid_ki
     - 12.0
   * - pid_kd
     - 0.1
   * - max_rpm
     - 3400
   * - min_rpm
     - 800
   * - min_pwm_duty
     - 10000 ns

**Validation**

- motor MUST be within device capability (0 to motor_count-1)
- At least one optional field MUST be present (motor index alone is invalid)
- If present, pwm_period MUST be > 0
- If both present, max_rpm MUST be > min_rpm
- If both present, min_pwm_duty MUST be < pwm_period
- PID gains MAY be 0 (disables that term)
- PID gains MUST NOT be NaN or Infinity

**Errors**

- Invalid motor index: ERROR_INVALID_CMD (code 2)
- Invalid parameter value: ERROR_INVALID_CMD (code 1)


.. _msg-pump-config:

PUMP_CONFIG
-----------

Configure fuel pump parameters including pulse duration and recovery time.

| **Payload:** :ref:`PUMP_CONFIG <payload-pump-config>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - pump
     - int
     - Pump index (0 to pump_count-1)
   * - pulse_ms (optional)
     - int
     - Solenoid pulse duration in milliseconds. If absent, use firmware default.
   * - recovery_ms (optional)
     - int
     - Recovery time after pulse in milliseconds. If absent, use firmware default.

**Default Values**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Field
     - Default
   * - pulse_ms
     - 50
   * - recovery_ms
     - 50

**Validation**

- pump MUST be within device capability (0 to pump_count-1)
- At least one optional field MUST be present (pump index alone is invalid)
- If present, pulse_ms MUST be > 0
- If present, recovery_ms MUST be > 0
- Minimum pump rate = pulse_ms + recovery_ms

**Errors**

- Invalid pump index: ERROR_INVALID_CMD (code 2)
- Invalid parameter value: ERROR_INVALID_CMD (code 1)

**Rationale**

- Pulse duration controls fuel delivery per cycle
- Recovery time prevents solenoid overheating and ensures complete valve closure


.. _msg-temperature-config:

TEMPERATURE_CONFIG
------------------

Configure temperature controller PID gains.

| **Payload:** :ref:`TEMPERATURE_CONFIG <payload-temperature-config>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - thermometer
     - int
     - Temperature sensor index (0 to thermometer_count-1)
   * - pid_kp (optional)
     - decimal
     - Proportional gain for :term:`PID controller`. If absent, use firmware default.
   * - pid_ki (optional)
     - decimal
     - Integral gain for :term:`PID controller`. If absent, use firmware default.
   * - pid_kd (optional)
     - decimal
     - Derivative gain for :term:`PID controller`. If absent, use firmware default.

**Default Values**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Field
     - Default
   * - pid_kp
     - 100.0
   * - pid_ki
     - 10.0
   * - pid_kd
     - 5.0

**Validation**

- thermometer MUST be within device capability (0 to thermometer_count-1)
- At least one optional field MUST be present (thermometer index alone is invalid)
- PID gains MAY be 0 (disables that term)
- PID gains MUST NOT be NaN or Infinity

**Errors**

- Invalid thermometer index: ERROR_INVALID_CMD (code 2)
- Invalid parameter value: ERROR_INVALID_CMD (code 1)

**Rationale**

- PID gains tuned for inverted control (higher temperature → higher RPM)


.. _msg-glow-config:

GLOW_CONFIG
-----------

Configure :term:`glow plug` parameters.

| **Payload:** :ref:`GLOW_CONFIG <payload-glow-config>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - glow
     - int
     - Glow plug index (0 to glow_count-1)
   * - max_duration (optional)
     - int
     - Maximum allowed glow duration in milliseconds. If absent, use firmware default.

**Default Values**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Field
     - Default
   * - max_duration
     - 300000 ms (5 minutes)

**Validation**

- glow MUST be within device capability (0 to glow_count-1)
- At least one optional field MUST be present (glow index alone is invalid)
- If present, max_duration MUST be > 0
- RECOMMENDED: max_duration ≤ 300000 ms (5 minutes) for safety

**Errors**

- Invalid glow index: ERROR_INVALID_CMD (code 2)
- Invalid parameter value: ERROR_INVALID_CMD (code 1)

**Rationale**

- Prevents indefinite glow plug operation
- Safety timeout for preheat phase


.. _msg-data-subscription:

DATA_SUBSCRIPTION
-----------------

Subscribe to receive data messages from a specific appliance.

| **Payload:** :ref:`DATA_SUBSCRIPTION <payload-data-subscription>`

**Direction:** Controller → Controller (for routing scenarios)

This is NOT a Controller → Appliance command. This command is sent from one
controller (subscriber) to another controller (router) to establish data
forwarding over a point-to-point connection.

**Address Field**

Client controllers MUST use the stateless address (``0xFFFFFFFFFFFFFFFF``) in
the ADDRESS field when sending DATA_SUBSCRIPTION to a router.

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - appliance_address
     - int
     - Address of the appliance to subscribe to (64-bit)

**Behavior**

When a controller (router) receives this command:

1. Associates the subscription with the connection on which it was received
2. Adds the connection to the routing table for the specified appliance_address
3. Forwards telemetry data messages (``0x30``–``0x34``) from that appliance

PING_RESPONSE (``0x3F``) and DEVICE_ANNOUNCE (``0x35``) are NOT forwarded.
DEVICE_ANNOUNCE is only sent during discovery, not as ongoing telemetry.

**Validation**

- appliance_address MUST NOT be the broadcast address (0x0000000000000000)
- Router MAY limit the number of active subscriptions (RECOMMENDED minimum: 10)
- Duplicate subscriptions refresh the timeout

**Notes**

- Subscriptions are NOT persistent (lost on power cycle or connection close)
- Routers SHOULD implement subscription timeout (60 seconds without PING_REQUEST)
- Subscriber controllers SHOULD send PING_REQUEST periodically to maintain routing


.. _msg-data-unsubscribe:

DATA_UNSUBSCRIBE
----------------

Remove a previously established data subscription.

| **Payload:** :ref:`DATA_UNSUBSCRIBE <payload-data-unsubscribe>`

**Direction:** Controller → Controller (for routing scenarios)

**Address Field**

Client controllers MUST use the stateless address (``0xFFFFFFFFFFFFFFFF``) in
the ADDRESS field when sending DATA_UNSUBSCRIBE to a router.

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - appliance_address
     - int
     - Address of the appliance to unsubscribe from (64-bit)

**Behavior**

When a controller (router) receives this command:

1. Identifies the connection on which the command was received
2. Removes the subscription for the specified appliance_address
3. No longer forwards data messages from that appliance to the connection

If no subscription exists, the command is silently ignored.


.. _msg-telemetry-config:

TELEMETRY_CONFIG
----------------

Enable or disable periodic telemetry broadcasts. See :doc:`communication-patterns`
for usage patterns.

| **Payload:** :ref:`TELEMETRY_CONFIG <payload-telemetry-config>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - enabled
     - bool
     - Enable (true) or disable (false) telemetry broadcasts
   * - interval_ms
     - int
     - Broadcast interval in milliseconds (0 = polling mode)

**Default State**

Telemetry broadcasts are **disabled** on boot.

**Validation**

- interval_ms range: 0 or 100-5000 ms
- Values 1-99 SHALL be clamped to 100 ms
- Values above 5000 SHALL be clamped to 5000 ms

**Behavior**

When enabled is true:

- **Broadcast mode** (interval_ms > 0): Appliance automatically sends telemetry
  at the configured interval
- **Polling mode** (interval_ms = 0): Appliance only sends telemetry in response
  to :ref:`SEND_TELEMETRY <msg-send-telemetry>` commands

**Auto-Disable**

Telemetry broadcasts are automatically disabled when the communication timeout
elapses (see :ref:`TIMEOUT_CONFIG <msg-timeout-config>`).

**Rationale**

- Disabling on boot prevents synchronization issues during initial connection
- Auto-disable on timeout reduces power consumption when controller is absent
- Polling mode useful for multi-appliance topologies to prevent bus collisions

**Routed Topologies**

In routed topologies, clients connected to a router SHOULD NOT send
TELEMETRY_CONFIG directly to appliances. The bus controller (router) is
responsible for telemetry configuration. See :doc:`packet-routing` for details.


.. _msg-timeout-config:

TIMEOUT_CONFIG
--------------

Configure communication timeout behavior.

| **Payload:** :ref:`TIMEOUT_CONFIG <payload-timeout-config>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - enabled
     - bool
     - Enable (true) or disable (false) timeout mode
   * - timeout_ms
     - int
     - Timeout interval in milliseconds

**Default State**

Timeout mode is **enabled** on boot with 30-second timeout.

**Validation**

- timeout_ms MUST be within range: 5000-60000 ms (5-60 seconds)
- Values outside range SHALL be clamped to nearest valid value

**Behavior**

When enabled:

- Appliance tracks time since last :ref:`PING_REQUEST <msg-ping-request>` received
- **Only PING_REQUEST resets the timeout timer** (other commands do not)
- If timeout_ms elapses without PING_REQUEST:

  - Appliance automatically transitions to IDLE mode
  - Telemetry broadcasts are automatically disabled

**Exception:** The communication timeout is suspended in E_STOP state. E_STOP can
only be cleared by power cycle or hardware reset, not by timeout. See
:doc:`implementation` for E_STOP behavior.

**Warning**

Disabling timeout mode (enabled=false) removes a critical safety feature.
Use with caution.

**Rationale**

- Ensures appliance doesn't operate indefinitely without controller supervision
- Critical for burner systems where loss of communication requires safe shutdown
- IDLE mode performs proper cooldown if temperature is elevated


.. _msg-discovery-request:

DISCOVERY_REQUEST
-----------------

Request device capabilities from all appliances on the bus.

| **Payload:** :ref:`DISCOVERY_REQUEST <payload-discovery-request>` (0 bytes)

**Appliance Behavior**

- Addressed (non-broadcast) DISCOVERY_REQUEST SHALL be ignored
- Appliances respond with :ref:`DEVICE_ANNOUNCE <msg-device-announce>`
- Appliances MUST wait a random delay (0-50ms) before responding

**Controller Behavior**

- Controller MUST send with broadcast address (0x0000000000000000)
- Controllers MUST wait at least 100ms to receive all appliance responses

**Router Behavior**

When a router receives DISCOVERY_REQUEST from a client:

1. Responds with DEVICE_ANNOUNCE for each accessible appliance
2. Sends a final DEVICE_ANNOUNCE with stateless address
   (``0xFFFFFFFFFFFFFFFF``) and all fields set to 0

This end-of-discovery marker allows clients to know when discovery is complete.
For routing details, see :ref:`session-initiation`.

**Response**

:ref:`DEVICE_ANNOUNCE <msg-device-announce>` from each appliance (and router).


Control Commands (0x20–0x2F)
****************************

Control commands provide real-time operational control without changing
persistent configuration.

.. _acknowledgment-philosophy-control:

**Acknowledgment**

Control commands do NOT receive explicit acknowledgments. Success is inferred
from subsequent telemetry:

- STATE_COMMAND success: STATE_DATA shows expected state
- MOTOR_COMMAND success: MOTOR_DATA shows expected RPM

Absence of an error response does NOT guarantee the command was received.
For critical operations, verify success via telemetry or use retry strategies.
Emergency stop uses explicit retry until confirmed via STATE_DATA.


.. _msg-state-command:

STATE_COMMAND
-------------

Set system operating mode.

| **Payload:** :ref:`STATE_COMMAND <payload-state-command>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - mode
     - enum
     - Operating mode (see values below)
   * - argument (optional)
     - int
     - Mode-specific parameter. If absent, use mode-specific default (see below).

**Mode Values**

.. list-table::
   :header-rows: 1
   :widths: 15 20 65

   * - Value
     - Mode
     - Argument (if present)
   * - 0
     - IDLE
     - Ignored
   * - 1
     - FAN
     - Target RPM (0 or min_rpm to max_rpm). If absent, use last commanded RPM
       or motor's default RPM.
   * - 2
     - HEAT
     - Pump rate in milliseconds. If absent, use last commanded rate or pump's
       default rate.
   * - 255
     - EMERGENCY
     - Ignored

**Mode → State Mapping**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - Command Mode
     - Resulting States
     - Notes
   * - IDLE (0)
     - IDLE, COOLING
     - COOLING first if temperature is elevated
   * - FAN (1)
     - BLOWING
     - Direct transition
   * - HEAT (2)
     - PREHEAT → PREHEAT_STAGE_2 → HEATING
     - Progresses through heating sequence
   * - EMERGENCY (255)
     - E_STOP
     - Requires power cycle to exit

**Validation**

- mode values 3–254: ERROR_INVALID_CMD (code 1)
- FAN mode: argument 1 to (min_rpm-1) is invalid: ERROR_INVALID_CMD (code 1)
- HEAT mode: argument 1 to (pulse_ms + recovery_ms - 1) is invalid: ERROR_INVALID_CMD (code 1)
- HEAT mode: argument > max_pump_rate is invalid: ERROR_INVALID_CMD (code 1)

The maximum pump rate is defined in the fuel profile. See
:doc:`/specifications/fuel-profiles/diesel` for fuel-specific limits (e.g., 5000ms for diesel).


.. _msg-motor-command:

MOTOR_COMMAND
-------------

Control motor (fan) speed.

| **Payload:** :ref:`MOTOR_COMMAND <payload-motor-command>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - motor
     - int
     - Motor index (0 to motor_count-1)
   * - rpm
     - int
     - Target RPM (0 = stop, min_rpm to max_rpm = run)

**Validation**

- motor MUST be within device capability (0 to motor_count-1)
- rpm MUST be 0 OR within motor's configured min/max range
- rpm values 1 to (min_rpm-1) are INVALID

**Errors**

- Invalid motor index: ERROR_INVALID_CMD (code 2)
- Invalid rpm value: ERROR_INVALID_CMD (code 1)


.. _msg-pump-command:

PUMP_COMMAND
------------

Control fuel pump rate.

| **Payload:** :ref:`PUMP_COMMAND <payload-pump-command>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - pump
     - int
     - Pump index (0 to pump_count-1)
   * - rate_ms
     - int
     - Pulse interval in milliseconds (0 = stop)

**Validation**

- pump MUST be within device capability (0 to pump_count-1)
- rate_ms MUST be 0 OR within valid range: (pulse_ms + recovery_ms) to max_pump_rate
- rate_ms values 1 to (pulse_ms + recovery_ms - 1) are INVALID

The maximum pump rate (max_pump_rate) is defined in the fuel profile. See
:doc:`/specifications/fuel-profiles/diesel` for fuel-specific limits (e.g., 5000ms
for diesel).

**Errors**

- Invalid pump index: ERROR_INVALID_CMD (code 2)
- Invalid rate_ms value: ERROR_INVALID_CMD (code 1)


.. _msg-glow-command:

GLOW_COMMAND
------------

Control glow plug heating.

| **Payload:** :ref:`GLOW_COMMAND <payload-glow-command>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - glow
     - int
     - Glow plug index (0 to glow_count-1)
   * - duration
     - int
     - Burn duration in milliseconds (0 = extinguish)

**Behavior**

- Glow plug automatically turns off when duration expires
- Duration expiry triggers :ref:`GLOW_DATA <msg-glow-data>` with lit=false
- Duration MUST NOT be extended while lit
- Sending duration=0 to a lit glow plug immediately extinguishes it

**Validation**

- glow MUST be within device capability (0 to glow_count-1)
- duration MUST be 0 to max_duration

**Errors**

- Invalid glow index: ERROR_INVALID_CMD (code 2)
- Invalid duration: ERROR_INVALID_CMD (code 1)
- Attempting to light already lit glow plug: ERROR_INVALID_CMD (code 1)
- Glow controlled by state machine (HEAT mode): ERROR_STATE_REJECT


.. _msg-temperature-command:

TEMPERATURE_COMMAND
-------------------

Configure temperature controller operation.

| **Payload:** :ref:`TEMPERATURE_COMMAND <payload-temperature-command>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - thermometer
     - int
     - Temperature sensor index (0 to thermometer_count-1)
   * - type
     - enum
     - Command type (see values below)
   * - motor_index (optional)
     - int
     - Motor to control. Required for WATCH_MOTOR; ignored otherwise.
   * - target_temperature (optional)
     - decimal
     - Target temperature in Celsius. Required for SET_TARGET_TEMPERATURE;
       ignored otherwise.

**Command Types**

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - Type
     - Description
   * - 0
     - WATCH_MOTOR
     - Associate temperature sensor with motor
   * - 1
     - UNWATCH_MOTOR
     - Stop monitoring motor
   * - 2
     - ENABLE_RPM_CONTROL
     - Enable PID control
   * - 3
     - DISABLE_RPM_CONTROL
     - Disable PID control
   * - 4
     - SET_TARGET_TEMPERATURE
     - Set temperature target

**Validation**

- type values > 4: ERROR_INVALID_CMD (code 1)
- thermometer MUST be within device capability
- motor_index (for WATCH_MOTOR) MUST be valid
- target_temperature MUST NOT be NaN or Infinity

**SET_TARGET_TEMPERATURE Restriction**

SET_TARGET_TEMPERATURE (type=4) MAY only be sent when appliance is in HEATING
state. Other states return ERROR_STATE_REJECT.


.. _msg-send-telemetry:

SEND_TELEMETRY
--------------

Request specific telemetry data from an appliance (polling mode). Polling mode
is recommended for multi-appliance RS-485 networks; see :doc:`physical-layer`.

| **Payload:** :ref:`SEND_TELEMETRY <payload-send-telemetry>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - telemetry_type
     - enum
     - Type of telemetry to request (see values below)
   * - index (optional)
     - int
     - Peripheral index. If absent or 0xFFFFFFFF, request all peripherals
       of the specified type.

**Telemetry Types**

.. list-table::
   :header-rows: 1
   :widths: 15 20 65

   * - Value
     - Type
     - Response Message
   * - 0
     - STATE
     - :ref:`STATE_DATA <msg-state-data>` (index ignored)
   * - 1
     - MOTOR
     - :ref:`MOTOR_DATA <msg-motor-data>`
   * - 2
     - TEMPERATURE
     - :ref:`TEMPERATURE_DATA <msg-temperature-data>`
   * - 3
     - PUMP
     - :ref:`PUMP_DATA <msg-pump-data>`
   * - 4
     - GLOW
     - :ref:`GLOW_DATA <msg-glow-data>`

**Behavior**

- If telemetry is disabled or in broadcast mode, command is **silently ignored**
- If index=0xFFFFFFFF, appliance sends one message per peripheral of that type
- If index is valid (0 to count-1), appliance sends one message for that peripheral
- **Does NOT reset communication timeout timer** (only PING_REQUEST does)

**Validation**

- telemetry_type > 4: ERROR_INVALID_CMD
- index out of range (and not 0xFFFFFFFF): ERROR_INVALID_CMD


.. _msg-ping-request:

PING_REQUEST
------------

Connectivity check and heartbeat. See :doc:`communication-patterns` for
heartbeat and timeout behavior.

| **Payload:** :ref:`PING_REQUEST <payload-ping-request>` (0 bytes)

**Response**

:ref:`PING_RESPONSE <msg-ping-response>` with uptime.

**Addressing**

PING_REQUEST uses different addressing depending on the target:

- **To appliances:** Use the appliance's address (addressed, not broadcast).
  Appliances respond with PING_RESPONSE.
- **To routers:** Use the broadcast address (``0x0000000000000000``). Routers
  respond with PING_RESPONSE using stateless address.

On multi-appliance networks, the controller MUST ping each appliance
individually by address to avoid bus collisions.

**Appliance Behavior**

PING_REQUEST resets the communication timeout timer. If timeout mode is enabled
and no PING_REQUEST is received within the configured interval, the appliance
automatically transitions to IDLE mode and disables telemetry.

**Client Controller Behavior**

Controllers acting as clients (connected to a router) MUST broadcast
PING_REQUEST using the broadcast address (``0x0000000000000000``). This allows
routers to track client health and maintain subscription timeouts.

**Router Behavior**

Controllers acting as routers MUST respond to PING_REQUEST with PING_RESPONSE
using the stateless address (``0xFFFFFFFFFFFFFFFF``). Routers do NOT
forward PING_REQUEST to appliances—ping is handled at each hop independently.


Telemetry Data (0x30–0x3F)
**************************

Telemetry messages are sent from appliances to controllers. They are only sent
when telemetry is enabled via :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`.

**Exception:** In E_STOP state, appliances broadcast all telemetry every 250ms
regardless of TELEMETRY_CONFIG state. See :doc:`implementation` for E_STOP
behavior details.


.. _msg-state-data:

STATE_DATA
----------

System state and error status.

| **Payload:** :ref:`STATE_DATA <payload-state-data>`
| **Send Rate:** 2.5× telemetry interval (250ms at default 100ms interval)

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - error
     - bool
     - Error flag (true = error occurred)
   * - code
     - int
     - Error code (see values below)
   * - state
     - enum
     - Current system state (see values below)
   * - timestamp
     - int
     - Milliseconds since boot

**State Values**

These states correspond to phases of the :doc:`/specifications/burn-cycle/index`.

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - State
     - Description
   * - 0
     - INITIALIZING
     - System startup in progress
   * - 1
     - IDLE
     - Ready for operation
   * - 2
     - BLOWING
     - Fan running without combustion (FAN mode)
   * - 3
     - PREHEAT
     - Glow plug warming (see :ref:`burn-cycle-ignition`)
   * - 4
     - PREHEAT_STAGE_2
     - Combustion starting, not yet stable (see :ref:`burn-cycle-preheating`)
   * - 5
     - HEATING
     - Normal combustion active (see :ref:`burn-cycle-heating`)
   * - 6
     - COOLING
     - Cooldown sequence in progress (see :ref:`burn-cycle-cooldown`)
   * - 7
     - ERROR
     - Fault detected
   * - 8
     - E_STOP
     - Emergency stop active (see :ref:`impl-estop`)

**Error Codes**

See :doc:`/specifications/burn-cycle/index` for fault condition details.

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Code
     - Name
     - Description
   * - 0
     - NONE
     - No error
   * - 1
     - OVERHEAT
     - Temperature exceeded safety limit (triggers E_STOP)
   * - 2
     - SENSOR_FAULT
     - Temperature sensor failure
   * - 3
     - IGNITION_FAIL
     - Failed to ignite after preheat (see :ref:`burn-cycle-ignition`)
   * - 4
     - FLAME_OUT
     - Flame lost during heating (see :ref:`burn-cycle-heating`)
   * - 5
     - MOTOR_STALL
     - Motor RPM dropped below minimum
   * - 6
     - PUMP_FAULT
     - Pump operation failure
   * - 7
     - COMMANDED_ESTOP
     - Emergency stop commanded by controller


.. _msg-motor-data:

MOTOR_DATA
----------

Motor telemetry including RPM and PWM feedback.

| **Payload:** :ref:`MOTOR_DATA <payload-motor-data>`
| **Send Rate:** Per telemetry interval (100ms at default)

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - motor
     - int
     - Motor index
   * - timestamp
     - int
     - Milliseconds since boot
   * - rpm
     - int
     - Current measured RPM
   * - target
     - int
     - Target RPM setpoint
   * - max_rpm (optional)
     - int
     - Maximum achievable RPM. If absent, information not reported.
   * - min_rpm (optional)
     - int
     - Minimum stable RPM. If absent, information not reported.
   * - pwm (optional)
     - int
     - Current PWM pulse width in nanoseconds. If absent, information not reported.
   * - pwm_max (optional)
     - int
     - PWM period in nanoseconds. If absent, information not reported.


.. _msg-pump-data:

PUMP_DATA
---------

Fuel pump status and events.

| **Payload:** :ref:`PUMP_DATA <payload-pump-data>`
| **Send Rate:** On event (state changes, cycle events)

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - pump
     - int
     - Pump index
   * - timestamp
     - int
     - Milliseconds since boot
   * - type
     - enum
     - Event type (see values below)
   * - rate (optional)
     - int
     - Current pump rate in milliseconds. If absent, rate not applicable
       for this event type (e.g., INITIALIZING, READY, ERROR).

**Event Types**

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - Type
     - Description
   * - 0
     - INITIALIZING
     - Pump initializing
   * - 1
     - READY
     - Pump ready
   * - 2
     - ERROR
     - Pump error
   * - 3
     - CYCLE_START
     - Pump cycle started
   * - 4
     - PULSE_END
     - Solenoid pulse ended
   * - 5
     - CYCLE_END
     - Pump cycle completed


.. _msg-glow-data:

GLOW_DATA
---------

Glow plug status.

| **Payload:** :ref:`GLOW_DATA <payload-glow-data>`
| **Send Rate:** On event (on/off transitions)

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - glow
     - int
     - Glow plug index
   * - timestamp
     - int
     - Milliseconds since boot
   * - lit
     - bool
     - Lit status (true = on, false = off)

**Events that trigger GLOW_DATA**

- Glow plug turns on (GLOW_COMMAND with duration > 0): lit=true
- Glow plug turns off via command (duration=0): lit=false
- Glow plug turns off automatically (duration expired): lit=false


.. _msg-temperature-data:

TEMPERATURE_DATA
----------------

Temperature sensor readings and PID control status.

| **Payload:** :ref:`TEMPERATURE_DATA <payload-temperature-data>`
| **Send Rate:** Per telemetry interval (100ms at default, after warmup)

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Field
     - Type
     - Description
   * - thermometer
     - int
     - Temperature sensor index
   * - timestamp
     - int
     - Milliseconds since boot
   * - reading
     - decimal
     - Current temperature in Celsius
   * - temperature_rpm_control (optional)
     - bool
     - Temperature-based RPM control active. If absent, RPM control is not active.
   * - watched_motor (optional)
     - int
     - Motor being controlled. If absent, no motor is being controlled.
   * - target_temperature (optional)
     - decimal
     - Target temperature for PID control. If absent, no target is set.

**Invalid Reading Handling**

If the sensor returns an invalid value (NaN, Infinity, or fault), the appliance
MUST transition to ERROR state and send STATE_DATA with error code SENSOR_FAULT.


.. _msg-device-announce:

DEVICE_ANNOUNCE
---------------

Device capabilities announcement.

| **Payload:** :ref:`DEVICE_ANNOUNCE <payload-device-announce>`
| **Send Rate:** On :ref:`DISCOVERY_REQUEST <msg-discovery-request>` only

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Field
     - Type
     - Description
   * - motor_count
     - int
     - Number of motors (1-255)
   * - thermometer_count
     - int
     - Number of temperature sensors (1-255)
   * - pump_count
     - int
     - Number of pumps (1-255)
   * - glow_count
     - int
     - Number of glow plugs (1-255)

**Appliance Behavior**

- Sent in response to broadcast DISCOVERY_REQUEST
- ADDRESS field contains the appliance's unique 64-bit address
- Appliances MUST wait a random delay (0-50ms) before responding
- Controllers MUST wait at least 100ms to receive all responses

**Router Behavior**

Routers use DEVICE_ANNOUNCE to mark the end of discovery:

- After sending DEVICE_ANNOUNCE for all accessible appliances, routers send a
  final DEVICE_ANNOUNCE with:

  - ADDRESS: Stateless address (``0xFFFFFFFFFFFFFFFF``)
  - All capability fields set to 0

- This end-of-discovery marker:

  - Marks the end of the device list
  - Allows clients to know when discovery is complete
  - Is the ONLY packet sent when no appliances are available

For routing details, see :ref:`session-initiation`.

**Validation**

All counts MUST be in range [1, 255] for appliance announcements. A count of 0
indicates either a malformed announcement or the end-of-discovery marker (when
ADDRESS is the stateless address).

**Rationale**

- Enables dynamic discovery of network topology
- Allows controllers to build appropriate UI based on actual hardware
- Eliminates need for manual configuration of device capabilities


.. _msg-ping-response:

PING_RESPONSE
-------------

Heartbeat response with system uptime.

| **Payload:** :ref:`PING_RESPONSE <payload-ping-response>`
| **Send Rate:** On request (response to :ref:`PING_REQUEST <msg-ping-request>`)

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - uptime_ms
     - int
     - System uptime in milliseconds (wraps at 2^32)

**Address Field**

- **Appliances:** Use their own device address
- **Routers:** Use the stateless address (``0xFFFFFFFFFFFFFFFF``)

This allows clients to distinguish between appliance responses and router
responses.

**Routing Behavior**

Routers MUST NOT forward PING_RESPONSE messages from appliances. Ping is
handled at each hop independently—clients receive PING_RESPONSE only from
the router they are directly connected to.


Error Messages (0xE0–0xEF)
**************************

Error messages indicate command validation failures. Appliances send error
messages to controllers when commands cannot be processed.

**When Errors Are Sent**

Errors are only sent when a command is explicitly rejected. Successful commands
do NOT receive any response—success is inferred from subsequent telemetry or
the absence of an error.

**Routing Behavior**

Routers forward error messages (``0xE0``–``0xE1``) to all subscribed clients,
just like data messages. This allows clients to detect command failures even
when communicating through routers.


.. _msg-error-invalid-cmd:

ERROR_INVALID_CMD
-----------------

Command validation failed.

| **Payload:** :ref:`ERROR_INVALID_CMD <payload-error-invalid-cmd>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - error_code
     - int
     - Error reason (see values below)

**Error Codes**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Code
     - Description
   * - 1
     - Invalid parameter value (out of range, NaN, etc.)
   * - 2
     - Invalid device index (motor, pump, or sensor does not exist)

.. note::

   A mechanism to identify which specific field caused the validation error is
   planned for future expansion.


.. _msg-error-state-reject:

ERROR_STATE_REJECT
------------------

Command rejected by appliance state machine.

| **Payload:** :ref:`ERROR_STATE_REJECT <payload-error-state-reject>`

**Fields**

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Field
     - Type
     - Description
   * - error_code
     - int
     - Current state that rejected the command

The error_code contains the state value (see :ref:`STATE_DATA <msg-state-data>`)
that caused the rejection.

**Recovery**

Controllers should handle ERROR_STATE_REJECT by either:

- Waiting for the appliance to reach an appropriate state
- Retrying the command after addressing the state conflict

.. note::

   A mechanism to communicate the specific reason why the state rejected the
   command is planned for future expansion.
