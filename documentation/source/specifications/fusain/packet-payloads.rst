Packet Payloads
###############

This document defines the payload structure for each Fusain message type.
For packet framing and encoding, see :doc:`packet-format`.

All multi-byte integers use little-endian byte order. The CRC field is the
only exception (big-endian, documented in :ref:`Packet Format <packet-crc>`).


Message Types
*************

Configuration Commands
----------------------

.. list-table::
   :header-rows: 1
   :widths: 15 25 45 15

   * - Type
     - Name
     - Description
     - Size
   * - :ref:`0x10 <payload-motor-config>`
     - :ref:`MOTOR_CONFIG <payload-motor-config>`
     - Configure motor controller parameters
     - 48 bytes
   * - :ref:`0x11 <payload-pump-config>`
     - :ref:`PUMP_CONFIG <payload-pump-config>`
     - Configure pump controller parameters
     - 16 bytes
   * - :ref:`0x12 <payload-temperature-config>`
     - :ref:`TEMPERATURE_CONFIG <payload-temperature-config>`
     - Configure temperature controller parameters
     - 48 bytes
   * - :ref:`0x13 <payload-glow-config>`
     - :ref:`GLOW_CONFIG <payload-glow-config>`
     - Configure :term:`glow plug` parameters
     - 16 bytes
   * - :ref:`0x14 <payload-data-subscription>`
     - :ref:`DATA_SUBSCRIPTION <payload-data-subscription>`
     - Subscribe to data from appliance
     - 8 bytes
   * - :ref:`0x15 <payload-data-unsubscribe>`
     - :ref:`DATA_UNSUBSCRIBE <payload-data-unsubscribe>`
     - Unsubscribe from appliance data
     - 8 bytes
   * - :ref:`0x16 <payload-telemetry-config>`
     - :ref:`TELEMETRY_CONFIG <payload-telemetry-config>`
     - Enable/disable telemetry broadcasts
     - 8 bytes
   * - :ref:`0x17 <payload-timeout-config>`
     - :ref:`TIMEOUT_CONFIG <payload-timeout-config>`
     - Configure communication timeout
     - 8 bytes
   * - 0x18–0x1E
     - *Reserved*
     - Reserved for future configuration commands
     - —
   * - :ref:`0x1F <payload-discovery-request>`
     - :ref:`DISCOVERY_REQUEST <payload-discovery-request>`
     - Request device capabilities
     - 0 bytes

Control Commands
----------------

.. list-table::
   :header-rows: 1
   :widths: 15 25 45 15

   * - Type
     - Name
     - Description
     - Size
   * - :ref:`0x20 <payload-state-command>`
     - :ref:`STATE_COMMAND <payload-state-command>`
     - Set system mode/state
     - 8 bytes
   * - :ref:`0x21 <payload-motor-command>`
     - :ref:`MOTOR_COMMAND <payload-motor-command>`
     - Set motor RPM
     - 8 bytes
   * - :ref:`0x22 <payload-pump-command>`
     - :ref:`PUMP_COMMAND <payload-pump-command>`
     - Set pump rate
     - 8 bytes
   * - :ref:`0x23 <payload-glow-command>`
     - :ref:`GLOW_COMMAND <payload-glow-command>`
     - Control glow plug
     - 8 bytes
   * - :ref:`0x24 <payload-temperature-command>`
     - :ref:`TEMPERATURE_COMMAND <payload-temperature-command>`
     - Temperature controller control
     - 20 bytes
   * - :ref:`0x25 <payload-send-telemetry>`
     - :ref:`SEND_TELEMETRY <payload-send-telemetry>`
     - Request telemetry data (polling mode)
     - 8 bytes
   * - 0x26–0x2E
     - *Reserved*
     - Reserved for future control commands
     - —
   * - :ref:`0x2F <payload-ping-request>`
     - :ref:`PING_REQUEST <payload-ping-request>`
     - Heartbeat/connectivity check
     - 0 bytes

Telemetry Data
--------------

.. list-table::
   :header-rows: 1
   :widths: 15 25 45 15

   * - Type
     - Name
     - Description
     - Size
   * - :ref:`0x30 <payload-state-data>`
     - :ref:`STATE_DATA <payload-state-data>`
     - System state and status
     - 16 bytes
   * - :ref:`0x31 <payload-motor-data>`
     - :ref:`MOTOR_DATA <payload-motor-data>`
     - Motor telemetry
     - 32 bytes
   * - :ref:`0x32 <payload-pump-data>`
     - :ref:`PUMP_DATA <payload-pump-data>`
     - Pump status and events
     - 16 bytes
   * - :ref:`0x33 <payload-glow-data>`
     - :ref:`GLOW_DATA <payload-glow-data>`
     - Glow plug status
     - 12 bytes
   * - :ref:`0x34 <payload-temperature-data>`
     - :ref:`TEMPERATURE_DATA <payload-temperature-data>`
     - Temperature readings
     - 32 bytes
   * - :ref:`0x35 <payload-device-announce>`
     - :ref:`DEVICE_ANNOUNCE <payload-device-announce>`
     - Device capabilities announcement
     - 8 bytes
   * - 0x36–0x3E
     - *Reserved*
     - Reserved for future telemetry messages
     - —
   * - :ref:`0x3F <payload-ping-response>`
     - :ref:`PING_RESPONSE <payload-ping-response>`
     - Heartbeat response
     - 4 bytes

Error Messages
--------------

.. list-table::
   :header-rows: 1
   :widths: 15 25 45 15

   * - Type
     - Name
     - Description
     - Size
   * - :ref:`0xE0 <payload-error-invalid-cmd>`
     - :ref:`ERROR_INVALID_CMD <payload-error-invalid-cmd>`
     - Command validation failed
     - 4 bytes
   * - :ref:`0xE1 <payload-error-state-reject>`
     - :ref:`ERROR_STATE_REJECT <payload-error-state-reject>`
     - Command rejected by state machine
     - 4 bytes
   * - 0xE2–0xEF
     - *Reserved*
     - Reserved for future error messages
     - —


Configuration Commands
**********************

Configuration commands are sent from controllers to appliances to set
persistent parameters. Message types range from ``0x10`` to ``0x1F``.


.. _payload-motor-config:

MOTOR_CONFIG
------------

| **Message Type:** ``0x10``
| **Payload Size:** 48 bytes
| **Behavior:** :ref:`MOTOR_CONFIG <msg-motor-config>`

Configure motor controller parameters.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - motor
     - i32
     - —
     - Motor index
   * - pwm_period
     - u32
     - microseconds
     - PWM period
   * - pid_kp
     - f64
     - —
     - Proportional gain
   * - pid_ki
     - f64
     - —
     - Integral gain
   * - pid_kd
     - f64
     - —
     - Derivative gain
   * - max_rpm
     - i32
     - RPM
     - Maximum RPM
   * - min_rpm
     - i32
     - RPM
     - Minimum stable RPM
   * - min_pwm_duty
     - u32
     - microseconds
     - Minimum PWM pulse width
   * - padding
     - 4 bytes
     - —
     - Reserved


.. _payload-pump-config:

PUMP_CONFIG
-----------

| **Message Type:** ``0x11``
| **Payload Size:** 16 bytes
| **Behavior:** :ref:`PUMP_CONFIG <msg-pump-config>`

Configure fuel pump parameters.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - pump
     - i32
     - —
     - Pump index
   * - pulse_ms
     - u32
     - milliseconds
     - Solenoid pulse duration
   * - recovery_ms
     - u32
     - milliseconds
     - Recovery time after pulse
   * - padding
     - 4 bytes
     - —
     - Reserved


.. _payload-temperature-config:

TEMPERATURE_CONFIG
------------------

| **Message Type:** ``0x12``
| **Payload Size:** 48 bytes
| **Behavior:** :ref:`TEMPERATURE_CONFIG <msg-temperature-config>`

Configure temperature controller parameters.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - temperature
     - i32
     - —
     - Temperature sensor index
   * - pid_kp
     - f64
     - —
     - Proportional gain
   * - pid_ki
     - f64
     - —
     - Integral gain
   * - pid_kd
     - f64
     - —
     - Derivative gain
   * - sample_count
     - u32
     - —
     - Number of samples for moving average filter
   * - read_rate
     - u32
     - milliseconds
     - Temperature reading interval
   * - padding
     - 12 bytes
     - —
     - Reserved


.. _payload-glow-config:

GLOW_CONFIG
-----------

| **Message Type:** ``0x13``
| **Payload Size:** 16 bytes
| **Behavior:** :ref:`GLOW_CONFIG <msg-glow-config>`

Configure glow plug parameters.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - glow
     - i32
     - —
     - Glow plug index
   * - max_duration
     - u32
     - milliseconds
     - Maximum glow duration
   * - padding
     - 8 bytes
     - —
     - Reserved


.. _payload-data-subscription:

DATA_SUBSCRIPTION
-----------------

| **Message Type:** ``0x14``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`DATA_SUBSCRIPTION <msg-data-subscription>`

Subscribe to data from an appliance (controller-to-controller routing).

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Parameter
     - Type
     - Unit
     - Description
   * - appliance_address
     - u64
     - —
     - Address of the appliance to subscribe to


.. _payload-data-unsubscribe:

DATA_UNSUBSCRIBE
----------------

| **Message Type:** ``0x15``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`DATA_UNSUBSCRIBE <msg-data-unsubscribe>`

Remove a data subscription.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Parameter
     - Type
     - Unit
     - Description
   * - appliance_address
     - u64
     - —
     - Address of the appliance to unsubscribe from


.. _payload-telemetry-config:

TELEMETRY_CONFIG
----------------

| **Message Type:** ``0x16``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`

Enable or disable periodic telemetry broadcasts.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Parameter
     - Type
     - Unit
     - Description
   * - telemetry_enabled
     - u32
     - —
     - Telemetry state (see values below)
   * - interval_ms
     - u32
     - milliseconds
     - Broadcast interval (0 = polling mode)

**Telemetry State Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Description
   * - 0
     - Disable telemetry
   * - 1
     - Enable telemetry


.. _payload-timeout-config:

TIMEOUT_CONFIG
--------------

| **Message Type:** ``0x17``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`TIMEOUT_CONFIG <msg-timeout-config>`

Configure communication timeout behavior.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - enabled
     - u32
     - —
     - Timeout state (see values below)
   * - timeout_ms
     - u32
     - milliseconds
     - Timeout interval

**Timeout State Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Description
   * - 0
     - Disable timeout
   * - 1
     - Enable timeout


.. _payload-discovery-request:

DISCOVERY_REQUEST
-----------------

| **Message Type:** ``0x1F``
| **Payload Size:** 0 bytes
| **Behavior:** :ref:`DISCOVERY_REQUEST <msg-discovery-request>`

Request device capabilities from all appliances. Appliances respond with
:ref:`DEVICE_ANNOUNCE <payload-device-announce>`.


Control Commands
****************

Control commands are sent from controllers to appliances for real-time
operational control. Message types range from ``0x20`` to ``0x2F``.


.. _payload-state-command:

STATE_COMMAND
-------------

| **Message Type:** ``0x20``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`STATE_COMMAND <msg-state-command>`

Set system operating mode.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - mode
     - u32
     - —
     - Operating mode (see values below)
   * - argument
     - i32
     - varies
     - Mode-specific parameter

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
   * - 3
     - EMERGENCY
     - Ignored


.. _payload-motor-command:

MOTOR_COMMAND
-------------

| **Message Type:** ``0x21``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`MOTOR_COMMAND <msg-motor-command>`

Set motor speed.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - motor
     - i32
     - —
     - Motor index
   * - rpm
     - i32
     - RPM
     - Target speed (0 = stop)


.. _payload-pump-command:

PUMP_COMMAND
------------

| **Message Type:** ``0x22``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`PUMP_COMMAND <msg-pump-command>`

Set fuel pump rate.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - pump
     - i32
     - —
     - Pump index
   * - rate_ms
     - i32
     - milliseconds
     - Pulse interval (0 = stop)


.. _payload-glow-command:

GLOW_COMMAND
------------

| **Message Type:** ``0x23``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`GLOW_COMMAND <msg-glow-command>`

Control glow plug.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - glow
     - i32
     - —
     - Glow plug index
   * - duration
     - i32
     - milliseconds
     - Burn duration (0 = extinguish)


.. _payload-temperature-command:

TEMPERATURE_COMMAND
-------------------

| **Message Type:** ``0x24``
| **Payload Size:** 20 bytes
| **Behavior:** :ref:`TEMPERATURE_COMMAND <msg-temperature-command>`

Configure temperature controller operation.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - temperature
     - i32
     - —
     - Temperature sensor index
   * - type
     - u32
     - —
     - Command type (see values below)
   * - motor_index
     - i32
     - —
     - Motor to control (WATCH_MOTOR only)
   * - target_temperature
     - f64
     - °C
     - Target temperature (SET_TARGET_TEMPERATURE only)

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
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`SEND_TELEMETRY <msg-send-telemetry>`

Request specific telemetry data (polling mode).

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - telemetry_type
     - u32
     - —
     - Telemetry type (see values below)
   * - index
     - u32
     - —
     - Peripheral index (0xFFFFFFFF = all)

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
| **Payload Size:** 0 bytes
| **Behavior:** :ref:`PING_REQUEST <msg-ping-request>`

Connectivity check and timeout reset. Appliances respond with
:ref:`PING_RESPONSE <payload-ping-response>`.


Telemetry Data
**************

Telemetry messages are sent from appliances to controllers. Message types
range from ``0x30`` to ``0x3F``.


.. _payload-state-data:

STATE_DATA
----------

| **Message Type:** ``0x30``
| **Payload Size:** 16 bytes
| **Behavior:** :ref:`STATE_DATA <msg-state-data>`

System state and error status.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - error
     - u32
     - —
     - Error flag (see values below)
   * - code
     - i32
     - —
     - Error code (see values below)
   * - state
     - u32
     - —
     - Current state (see values below)
   * - timestamp
     - u32
     - milliseconds
     - Time since boot

**Error Flag Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Description
   * - 0
     - No error
   * - 1
     - Error occurred

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
| **Payload Size:** 32 bytes
| **Behavior:** :ref:`MOTOR_DATA <msg-motor-data>`

Motor telemetry.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - motor
     - i32
     - —
     - Motor index
   * - timestamp
     - u32
     - milliseconds
     - Time since boot
   * - rpm
     - i32
     - RPM
     - Current measured speed
   * - target
     - i32
     - RPM
     - Target speed setpoint
   * - max_rpm
     - i32
     - RPM
     - Maximum achievable speed
   * - min_rpm
     - i32
     - RPM
     - Minimum stable speed
   * - pwm
     - i32
     - microseconds
     - Current PWM pulse width
   * - pwm_max
     - i32
     - microseconds
     - PWM period


.. _payload-pump-data:

PUMP_DATA
---------

| **Message Type:** ``0x32``
| **Payload Size:** 16 bytes
| **Behavior:** :ref:`PUMP_DATA <msg-pump-data>`

Fuel pump status and events.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - pump
     - i32
     - —
     - Pump index
   * - timestamp
     - u32
     - milliseconds
     - Event timestamp since boot
   * - type
     - u32
     - —
     - Event type (see values below)
   * - rate
     - i32
     - milliseconds
     - Current pump rate

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
| **Payload Size:** 12 bytes
| **Behavior:** :ref:`GLOW_DATA <msg-glow-data>`

Glow plug status.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - glow
     - i32
     - —
     - Glow plug index
   * - timestamp
     - u32
     - milliseconds
     - Status timestamp since boot
   * - lit
     - u32
     - —
     - Glow state (see values below)

**Glow State Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Description
   * - 0
     - Off
   * - 1
     - Lit


.. _payload-temperature-data:

TEMPERATURE_DATA
----------------

| **Message Type:** ``0x34``
| **Payload Size:** 32 bytes
| **Behavior:** :ref:`TEMPERATURE_DATA <msg-temperature-data>`

Temperature readings and PID status.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Parameter
     - Type
     - Unit
     - Description
   * - temperature
     - i32
     - —
     - Sensor index
   * - timestamp
     - u32
     - milliseconds
     - Reading timestamp since boot
   * - reading
     - f64
     - °C
     - Current temperature
   * - temperature_rpm_control
     - u32
     - —
     - PID control state (see values below)
   * - watched_motor
     - i32
     - —
     - Motor being controlled (-1 = none)
   * - target_temperature
     - f64
     - °C
     - Target temperature

**PID Control State Values**

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - Value
     - Description
   * - 0
     - PID control inactive
   * - 1
     - PID control active


.. _payload-device-announce:

DEVICE_ANNOUNCE
---------------

| **Message Type:** ``0x35``
| **Payload Size:** 8 bytes
| **Behavior:** :ref:`DEVICE_ANNOUNCE <msg-device-announce>`

Device capabilities (response to :ref:`DISCOVERY_REQUEST <payload-discovery-request>`).

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Parameter
     - Type
     - Unit
     - Description
   * - motor_count
     - u8
     - —
     - Number of motors (1-255 for appliances, 0 for end marker)
   * - temperature_count
     - u8
     - —
     - Number of temperature sensors (1-255 for appliances, 0 for end marker)
   * - pump_count
     - u8
     - —
     - Number of pumps (1-255 for appliances, 0 for end marker)
   * - glow_count
     - u8
     - —
     - Number of glow plugs (1-255 for appliances, 0 for end marker)
   * - padding
     - 4 bytes
     - —
     - Reserved

**End-of-Discovery Marker**

Routers send a special DEVICE_ANNOUNCE to mark the end of discovery:

- ADDRESS field: Stateless address (``0xFFFFFFFFFFFFFFFF``)
- All capability fields (motor_count, temperature_count, pump_count,
  glow_count): 0

This packet indicates discovery is complete. It is the ONLY DEVICE_ANNOUNCE
sent when no appliances are available. See :ref:`session-initiation` for
details.


.. _payload-ping-response:

PING_RESPONSE
-------------

| **Message Type:** ``0x3F``
| **Payload Size:** 4 bytes
| **Behavior:** :ref:`PING_RESPONSE <msg-ping-response>`

Heartbeat response to :ref:`PING_REQUEST <payload-ping-request>`.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - uptime_ms
     - u32
     - milliseconds
     - System uptime


Error Messages
**************

Error messages are sent from appliances to controllers when commands fail.
Message types range from ``0xE0`` to ``0xEF``.


.. _payload-error-invalid-cmd:

ERROR_INVALID_CMD
-----------------

| **Message Type:** ``0xE0``
| **Payload Size:** 4 bytes
| **Behavior:** :ref:`ERROR_INVALID_CMD <msg-error-invalid-cmd>`

Command validation failed.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - error_code
     - i32
     - —
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
| **Payload Size:** 4 bytes
| **Behavior:** :ref:`ERROR_STATE_REJECT <msg-error-state-reject>`

Command rejected by state machine.

**Parameters**

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Parameter
     - Type
     - Unit
     - Description
   * - error_code
     - i32
     - —
     - Current state that rejected the command
