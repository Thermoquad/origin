Packet Payloads
###############

This document defines the CBOR payload structure for each Fusain message type.
For packet framing, see :doc:`packet-format`. For the canonical schema, see
``fusain.cddl``.

All payloads are CBOR maps with integer keys. Optional fields (marked with
``?``) may be omitted.


Message Types
*************

Configuration Commands
----------------------

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Type
     - Name
     - Description
   * - :ref:`0x10 <payload-motor-config>`
     - :ref:`MOTOR_CONFIG <payload-motor-config>`
     - Configure motor controller parameters
   * - :ref:`0x11 <payload-pump-config>`
     - :ref:`PUMP_CONFIG <payload-pump-config>`
     - Configure pump controller parameters
   * - :ref:`0x12 <payload-temperature-config>`
     - :ref:`TEMPERATURE_CONFIG <payload-temperature-config>`
     - Configure temperature controller parameters
   * - :ref:`0x13 <payload-glow-config>`
     - :ref:`GLOW_CONFIG <payload-glow-config>`
     - Configure :term:`glow plug` parameters
   * - :ref:`0x14 <payload-data-subscription>`
     - :ref:`DATA_SUBSCRIPTION <payload-data-subscription>`
     - Subscribe to data from appliance
   * - :ref:`0x15 <payload-data-unsubscribe>`
     - :ref:`DATA_UNSUBSCRIBE <payload-data-unsubscribe>`
     - Unsubscribe from appliance data
   * - :ref:`0x16 <payload-telemetry-config>`
     - :ref:`TELEMETRY_CONFIG <payload-telemetry-config>`
     - Enable/disable telemetry broadcasts
   * - :ref:`0x17 <payload-timeout-config>`
     - :ref:`TIMEOUT_CONFIG <payload-timeout-config>`
     - Configure communication timeout
   * - 0x18–0x1E
     - *Reserved*
     - Reserved for future configuration commands
   * - :ref:`0x1F <payload-discovery-request>`
     - :ref:`DISCOVERY_REQUEST <payload-discovery-request>`
     - Request device capabilities

Control Commands
----------------

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Type
     - Name
     - Description
   * - :ref:`0x20 <payload-state-command>`
     - :ref:`STATE_COMMAND <payload-state-command>`
     - Set system mode/state
   * - :ref:`0x21 <payload-motor-command>`
     - :ref:`MOTOR_COMMAND <payload-motor-command>`
     - Set motor RPM
   * - :ref:`0x22 <payload-pump-command>`
     - :ref:`PUMP_COMMAND <payload-pump-command>`
     - Set pump rate
   * - :ref:`0x23 <payload-glow-command>`
     - :ref:`GLOW_COMMAND <payload-glow-command>`
     - Control glow plug
   * - :ref:`0x24 <payload-temperature-command>`
     - :ref:`TEMPERATURE_COMMAND <payload-temperature-command>`
     - Temperature controller control
   * - :ref:`0x25 <payload-send-telemetry>`
     - :ref:`SEND_TELEMETRY <payload-send-telemetry>`
     - Request telemetry data (polling mode)
   * - 0x26–0x2E
     - *Reserved*
     - Reserved for future control commands
   * - :ref:`0x2F <payload-ping-request>`
     - :ref:`PING_REQUEST <payload-ping-request>`
     - Heartbeat/connectivity check

Telemetry Data
--------------

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Type
     - Name
     - Description
   * - :ref:`0x30 <payload-state-data>`
     - :ref:`STATE_DATA <payload-state-data>`
     - System state and status
   * - :ref:`0x31 <payload-motor-data>`
     - :ref:`MOTOR_DATA <payload-motor-data>`
     - Motor telemetry
   * - :ref:`0x32 <payload-pump-data>`
     - :ref:`PUMP_DATA <payload-pump-data>`
     - Pump status and events
   * - :ref:`0x33 <payload-glow-data>`
     - :ref:`GLOW_DATA <payload-glow-data>`
     - Glow plug status
   * - :ref:`0x34 <payload-temperature-data>`
     - :ref:`TEMPERATURE_DATA <payload-temperature-data>`
     - Temperature readings
   * - :ref:`0x35 <payload-device-announce>`
     - :ref:`DEVICE_ANNOUNCE <payload-device-announce>`
     - Device capabilities announcement
   * - 0x36–0x3E
     - *Reserved*
     - Reserved for future telemetry messages
   * - :ref:`0x3F <payload-ping-response>`
     - :ref:`PING_RESPONSE <payload-ping-response>`
     - Heartbeat response

Error Messages
--------------

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Type
     - Name
     - Description
   * - :ref:`0xE0 <payload-error-invalid-cmd>`
     - :ref:`ERROR_INVALID_CMD <payload-error-invalid-cmd>`
     - Command validation failed
   * - :ref:`0xE1 <payload-error-state-reject>`
     - :ref:`ERROR_STATE_REJECT <payload-error-state-reject>`
     - Command rejected by state machine
   * - 0xE2–0xEF
     - *Reserved*
     - Reserved for future error messages


Configuration Commands
**********************

Configuration commands are sent from controllers to appliances to set
persistent parameters. Message types range from ``0x10`` to ``0x1F``.


.. _payload-motor-config:

MOTOR_CONFIG
------------

| **Message Type:** ``0x10``
| **Behavior:** :ref:`MOTOR_CONFIG <msg-motor-config>`

Configure motor controller parameters.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - motor
     - uint
     - Motor index
   * - 1 (?)
     - pwm_period
     - uint
     - PWM period in nanoseconds (optional)
   * - 2 (?)
     - pid_kp
     - float
     - Proportional gain (optional)
   * - 3 (?)
     - pid_ki
     - float
     - Integral gain (optional)
   * - 4 (?)
     - pid_kd
     - float
     - Derivative gain (optional)
   * - 5 (?)
     - max_rpm
     - int
     - Maximum RPM (optional)
   * - 6 (?)
     - min_rpm
     - int
     - Minimum stable RPM (optional)
   * - 7 (?)
     - min_pwm_duty
     - uint
     - Minimum PWM pulse width in nanoseconds (optional)


.. _payload-pump-config:

PUMP_CONFIG
-----------

| **Message Type:** ``0x11``
| **Behavior:** :ref:`PUMP_CONFIG <msg-pump-config>`

Configure fuel pump parameters.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - pump
     - uint
     - Pump index
   * - 1 (?)
     - pulse_ms
     - uint
     - Solenoid pulse duration in milliseconds (optional)
   * - 2 (?)
     - recovery_ms
     - uint
     - Recovery time after pulse in milliseconds (optional)


.. _payload-temperature-config:

TEMPERATURE_CONFIG
------------------

| **Message Type:** ``0x12``
| **Behavior:** :ref:`TEMPERATURE_CONFIG <msg-temperature-config>`

Configure temperature controller parameters.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - thermometer
     - uint
     - Temperature sensor index
   * - 1 (?)
     - pid_kp
     - float
     - Proportional gain (optional)
   * - 2 (?)
     - pid_ki
     - float
     - Integral gain (optional)
   * - 3 (?)
     - pid_kd
     - float
     - Derivative gain (optional)


.. _payload-glow-config:

GLOW_CONFIG
-----------

| **Message Type:** ``0x13``
| **Behavior:** :ref:`GLOW_CONFIG <msg-glow-config>`

Configure glow plug parameters.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - glow
     - uint
     - Glow plug index
   * - 1 (?)
     - max_duration
     - uint
     - Maximum glow duration in milliseconds (optional)


.. _payload-data-subscription:

DATA_SUBSCRIPTION
-----------------

| **Message Type:** ``0x14``
| **Behavior:** :ref:`DATA_SUBSCRIPTION <msg-data-subscription>`

Subscribe to data from an appliance (controller-to-controller routing).

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - appliance_address
     - uint
     - 64-bit address of the appliance to subscribe to


.. _payload-data-unsubscribe:

DATA_UNSUBSCRIBE
----------------

| **Message Type:** ``0x15``
| **Behavior:** :ref:`DATA_UNSUBSCRIBE <msg-data-unsubscribe>`

Remove a data subscription.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - appliance_address
     - uint
     - 64-bit address of the appliance to unsubscribe from


.. _payload-telemetry-config:

TELEMETRY_CONFIG
----------------

| **Message Type:** ``0x16``
| **Behavior:** :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`

Enable or disable periodic telemetry broadcasts.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - enabled
     - bool
     - Telemetry broadcast enabled
   * - 1
     - interval_ms
     - uint
     - Broadcast interval in milliseconds (0 = polling mode)


.. _payload-timeout-config:

TIMEOUT_CONFIG
--------------

| **Message Type:** ``0x17``
| **Behavior:** :ref:`TIMEOUT_CONFIG <msg-timeout-config>`

Configure communication timeout behavior.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - enabled
     - bool
     - Timeout enabled
   * - 1
     - timeout_ms
     - uint
     - Timeout interval in milliseconds (5000–60000)


.. _payload-discovery-request:

DISCOVERY_REQUEST
-----------------

| **Message Type:** ``0x1F``
| **Behavior:** :ref:`DISCOVERY_REQUEST <msg-discovery-request>`

Request device capabilities from all appliances. Appliances respond with
:ref:`DEVICE_ANNOUNCE <payload-device-announce>`.

**Payload:** ``nil`` (empty)


Control Commands
****************

Control commands are sent from controllers to appliances for real-time
operational control. Message types range from ``0x20`` to ``0x2F``.


.. _payload-state-command:

STATE_COMMAND
-------------

| **Message Type:** ``0x20``
| **Behavior:** :ref:`STATE_COMMAND <msg-state-command>`

Set system operating mode.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - mode
     - uint
     - Operating mode (see values below)
   * - 1 (?)
     - argument
     - int
     - Mode-specific parameter (optional)

**Mode Values**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - Value
     - Mode
     - Argument
   * - 0
     - IDLE
     - Ignored
   * - 1
     - FAN
     - Target RPM
   * - 2
     - HEAT
     - Pump rate (milliseconds)
   * - 255
     - EMERGENCY
     - Ignored


.. _payload-motor-command:

MOTOR_COMMAND
-------------

| **Message Type:** ``0x21``
| **Behavior:** :ref:`MOTOR_COMMAND <msg-motor-command>`

Set motor speed.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - motor
     - uint
     - Motor index
   * - 1
     - rpm
     - int
     - Target speed (0 = stop)


.. _payload-pump-command:

PUMP_COMMAND
------------

| **Message Type:** ``0x22``
| **Behavior:** :ref:`PUMP_COMMAND <msg-pump-command>`

Set fuel pump rate.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - pump
     - uint
     - Pump index
   * - 1
     - rate_ms
     - int
     - Pulse interval in milliseconds (0 = stop)


.. _payload-glow-command:

GLOW_COMMAND
------------

| **Message Type:** ``0x23``
| **Behavior:** :ref:`GLOW_COMMAND <msg-glow-command>`

Control glow plug.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - glow
     - uint
     - Glow plug index
   * - 1
     - duration
     - int
     - Burn duration in milliseconds (0 = extinguish)


.. _payload-temperature-command:

TEMPERATURE_COMMAND
-------------------

| **Message Type:** ``0x24``
| **Behavior:** :ref:`TEMPERATURE_COMMAND <msg-temperature-command>`

Configure temperature controller operation.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - thermometer
     - uint
     - Temperature sensor index
   * - 1
     - type
     - uint
     - Command type (see values below)
   * - 2 (?)
     - motor_index
     - uint
     - Motor to control (WATCH_MOTOR only, optional)
   * - 3 (?)
     - target_temperature
     - float
     - Target temperature in °C (SET_TARGET_TEMPERATURE only, optional)

**Command Type Values**

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Value
     - Type
     - Description
   * - 0
     - WATCH_MOTOR
     - Associate temperature sensor with a motor
   * - 1
     - UNWATCH_MOTOR
     - Remove motor association
   * - 2
     - ENABLE_RPM_CONTROL
     - Enable automatic RPM adjustment
   * - 3
     - DISABLE_RPM_CONTROL
     - Disable automatic RPM adjustment
   * - 4
     - SET_TARGET_TEMPERATURE
     - Set target temperature for PID control


.. _payload-send-telemetry:

SEND_TELEMETRY
--------------

| **Message Type:** ``0x25``
| **Behavior:** :ref:`SEND_TELEMETRY <msg-send-telemetry>`

Request specific telemetry data (polling mode).

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - telemetry_type
     - uint
     - Telemetry type (see values below)
   * - 1 (?)
     - index
     - uint
     - Peripheral index (0xFFFFFFFF = all, optional)

**Telemetry Type Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Type
   * - 0
     - STATE
   * - 1
     - MOTOR
   * - 2
     - TEMPERATURE
   * - 3
     - PUMP
   * - 4
     - GLOW


.. _payload-ping-request:

PING_REQUEST
------------

| **Message Type:** ``0x2F``
| **Behavior:** :ref:`PING_REQUEST <msg-ping-request>`

Connectivity check and timeout reset. Appliances respond with
:ref:`PING_RESPONSE <payload-ping-response>`.

**Payload:** ``nil`` (empty)


Telemetry Data
**************

Telemetry messages are sent from appliances to controllers. Message types
range from ``0x30`` to ``0x3F``.


.. _payload-state-data:

STATE_DATA
----------

| **Message Type:** ``0x30``
| **Behavior:** :ref:`STATE_DATA <msg-state-data>`

System state and error status.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - error
     - bool
     - Error flag (true if error occurred)
   * - 1
     - code
     - uint
     - Error code (see values below)
   * - 2
     - state
     - uint
     - Current state (see values below)
   * - 3
     - timestamp
     - uint
     - Time since boot in milliseconds

**State Values**

See :doc:`/specifications/burn-cycle/index` for operational details.

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
     - Fan running without combustion
   * - 3
     - PREHEAT
     - Glow plug warming
   * - 4
     - PREHEAT_STAGE_2
     - Combustion starting, not yet stable
   * - 5
     - HEATING
     - Normal combustion active
   * - 6
     - COOLING
     - Cooldown sequence in progress
   * - 7
     - ERROR
     - Fault detected
   * - 8
     - E_STOP
     - Emergency stop active

**Error Code Values**

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - Code
     - Description
   * - 0
     - NONE
     - No error
   * - 1
     - OVERHEAT
     - Temperature exceeded safety limit
   * - 2
     - SENSOR_FAULT
     - Temperature sensor failure
   * - 3
     - IGNITION_FAIL
     - Failed to reach ignition temperature
   * - 4
     - FLAME_OUT
     - Temperature dropped during heating
   * - 5
     - MOTOR_STALL
     - Motor RPM below threshold
   * - 6
     - PUMP_FAULT
     - Pump operation failure
   * - 7
     - COMMANDED_ESTOP
     - Emergency stop commanded


.. _payload-motor-data:

MOTOR_DATA
----------

| **Message Type:** ``0x31``
| **Behavior:** :ref:`MOTOR_DATA <msg-motor-data>`

Motor telemetry.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - motor
     - uint
     - Motor index
   * - 1
     - timestamp
     - uint
     - Time since boot in milliseconds
   * - 2
     - rpm
     - int
     - Current measured speed
   * - 3
     - target
     - int
     - Target speed setpoint
   * - 4 (?)
     - max_rpm
     - int
     - Maximum achievable speed (optional)
   * - 5 (?)
     - min_rpm
     - int
     - Minimum stable speed (optional)
   * - 6 (?)
     - pwm
     - uint
     - Current PWM pulse width in nanoseconds (optional)
   * - 7 (?)
     - pwm_max
     - uint
     - PWM period in nanoseconds (optional)


.. _payload-pump-data:

PUMP_DATA
---------

| **Message Type:** ``0x32``
| **Behavior:** :ref:`PUMP_DATA <msg-pump-data>`

Fuel pump status and events.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - pump
     - uint
     - Pump index
   * - 1
     - timestamp
     - uint
     - Event timestamp since boot in milliseconds
   * - 2
     - type
     - uint
     - Event type (see values below)
   * - 3 (?)
     - rate
     - int
     - Current pump rate in milliseconds (optional)

**Event Type Values**

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - Type
     - Description
   * - 0
     - INITIALIZING
     - Pump controller starting up
   * - 1
     - READY
     - Pump ready for operation
   * - 2
     - ERROR
     - Pump fault detected
   * - 3
     - CYCLE_START
     - Beginning of a pump cycle
   * - 4
     - PULSE_END
     - Solenoid pulse completed
   * - 5
     - CYCLE_END
     - Recovery period completed


.. _payload-glow-data:

GLOW_DATA
---------

| **Message Type:** ``0x33``
| **Behavior:** :ref:`GLOW_DATA <msg-glow-data>`

Glow plug status.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - glow
     - uint
     - Glow plug index
   * - 1
     - timestamp
     - uint
     - Status timestamp since boot in milliseconds
   * - 2
     - lit
     - bool
     - Glow state (true = lit, false = off)


.. _payload-temperature-data:

TEMPERATURE_DATA
----------------

| **Message Type:** ``0x34``
| **Behavior:** :ref:`TEMPERATURE_DATA <msg-temperature-data>`

Temperature readings and PID status.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - thermometer
     - uint
     - Sensor index
   * - 1
     - timestamp
     - uint
     - Reading timestamp since boot in milliseconds
   * - 2
     - reading
     - float
     - Current temperature in °C
   * - 3 (?)
     - temperature_rpm_control
     - bool
     - PID control active (optional)
   * - 4 (?)
     - watched_motor
     - int
     - Motor being controlled (optional)
   * - 5 (?)
     - target_temperature
     - float
     - Target temperature in °C (optional)


.. _payload-device-announce:

DEVICE_ANNOUNCE
---------------

| **Message Type:** ``0x35``
| **Behavior:** :ref:`DEVICE_ANNOUNCE <msg-device-announce>`

Device capabilities (response to :ref:`DISCOVERY_REQUEST <payload-discovery-request>`).

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - motor_count
     - uint
     - Number of motors
   * - 1
     - thermometer_count
     - uint
     - Number of temperature sensors
   * - 2
     - pump_count
     - uint
     - Number of pumps
   * - 3
     - glow_count
     - uint
     - Number of glow plugs

**End-of-Discovery Marker**

Routers send a special DEVICE_ANNOUNCE to mark the end of discovery:

- ADDRESS field: Stateless address (``0xFFFFFFFFFFFFFFFF``)
- All capability fields (motor_count, thermometer_count, pump_count,
  glow_count): 0

This packet indicates discovery is complete. It is the ONLY DEVICE_ANNOUNCE
sent when no appliances are available. See :ref:`session-initiation` for
details.


.. _payload-ping-response:

PING_RESPONSE
-------------

| **Message Type:** ``0x3F``
| **Behavior:** :ref:`PING_RESPONSE <msg-ping-response>`

Heartbeat response to :ref:`PING_REQUEST <payload-ping-request>`.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - uptime_ms
     - uint
     - System uptime in milliseconds


Error Messages
**************

Error messages are sent from appliances to controllers when commands fail.
Message types range from ``0xE0`` to ``0xEF``.


.. _payload-error-invalid-cmd:

ERROR_INVALID_CMD
-----------------

| **Message Type:** ``0xE0``
| **Behavior:** :ref:`ERROR_INVALID_CMD <msg-error-invalid-cmd>`

Command validation failed.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - error_code
     - int
     - Error reason (see values below)

**Error Code Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Description
   * - 1
     - Invalid parameter value (out of range, NaN, etc.)
   * - 2
     - Invalid device index (motor, pump, or sensor does not exist)


.. _payload-error-state-reject:

ERROR_STATE_REJECT
------------------

| **Message Type:** ``0xE1``
| **Behavior:** :ref:`ERROR_STATE_REJECT <msg-error-state-reject>`

Command rejected by state machine.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - error_code
     - uint
     - State that rejected the command
