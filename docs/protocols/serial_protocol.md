# Fusain Protocol Specification

## Overview

The Fusain protocol is a binary packet-based protocol for communicating with Helios-compatible appliances (burners, ignition control units) over UART or other serial transports. The protocol provides comprehensive configuration, command/control capabilities, and real-time telemetry.

**Fusain** (fossilized charcoal) - A platform-independent communication protocol for thermal appliance control.

### RFC 2119 Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

**Terminology Note:** Throughout this document, "ignore" means "silently ignore" - the receiver discards the packet without sending any response or error message.

**Transport Layer:**
- UART with optional physical layer translation (RS-485, LIN)
- Baud Rate: 115200 (UART, RS-485) or 19200 (LIN) - determined by physical layer
- Data Format: 8N1 (8 data bits, no parity, 1 stop bit)
- Flow Control: None
- See "Physical Layer Compatibility" section for RS-485 and LIN requirements

**Protocol Characteristics:**
- Binary packet format with framing
- CRC-16-CCITT for error detection
- Fixed maximum packet size: 128 bytes
- Bidirectional communication
- Device addressing for multi-device networks
- Optional periodic telemetry broadcast (configurable 100-5000ms, disabled by default)

**Network Architecture:**
- **Controller Device:** Sends command packets, receives data packets (e.g., UI controller, automation system)
- **Appliance Device:** Receives command packets, sends data packets (e.g., Helios ICU, burner control unit)
- **Monitor Device:** Receives packets but does not send commands or data (e.g., display-only device, data logger)
- Controllers initiate configuration and control commands
- Appliances respond with telemetry and status
- Appliances broadcast periodic telemetry when enabled by controller (configurable interval)
- Telemetry broadcasting is disabled by default for boot synchronization
- Default/recommended interval: 100ms (range: 100-5000ms)
- **Typical appliance:** 1 motor, 1 temperature sensor, 1 pump, 1 glow plug (e.g., Helios ICU)

---

## Packet Format

All packets follow this structure:

```
+----------+--------+---------+----------+---------+-----------+--------+
| START    | LENGTH | ADDRESS | MSG_TYPE | PAYLOAD | CRC-16    | END    |
| 1 byte   | 1 byte | 8 bytes | 1 byte   | N bytes | 2 bytes   | 1 byte |
+----------+--------+---------+----------+---------+-----------+--------+
| 0x7E     | 0-114  | u64     | 0x00-FF  | ...     | MSB, LSB  | 0x7F   |
+----------+--------+---------+----------+---------+-----------+--------+
```

### Field Descriptions

| Field | Size | Description |
|-------|------|-------------|
| **START** | 1 byte | Start delimiter: `0x7E` |
| **LENGTH** | 1 byte | Payload length (0-114 bytes, excludes framing/CRC/address) |
| **ADDRESS** | 8 bytes | 64-bit device address (little-endian u64) |
| **MSG_TYPE** | 1 byte | Message type identifier (see Message Types) |
| **PAYLOAD** | 0-114 bytes | Message-specific payload data |
| **CRC-16** | 2 bytes | CRC-16-CCITT (poly 0x1021, init 0xFFFF) over LENGTH + ADDRESS + MSG_TYPE + PAYLOAD |
| **END** | 1 byte | End delimiter: `0x7F` |

**Total Packet Size:** 14 bytes (framing/CRC/address) + payload length = 14-128 bytes

### Device Addressing

**Address Field (u64, little-endian):**
- **Command Packets:** Address represents the DESTINATION device
- **Data Packets:** Address represents the SOURCE device (sender)
- **Broadcast Address:** 0x0000000000000000 (commands sent to all devices)
- **Reserved Addresses:** 0xFFFFFFFFFFFFFFFF (reserved for future use)

**Address Assignment:**
- Addresses SHOULD be globally unique (e.g., MAC address, serial number)
- Controllers MAY use broadcast address for discovery or simultaneous control
- Appliances MUST respond with their own address in data packets
- Monitors MUST use their own address for filtering (if applicable)
- Appliances MUST silently ignore packets addressed to other devices (non-matching, non-broadcast address)

**Broadcast Address Behavior (0x0000000000000000):**

When a device receives a packet with ADDRESS = 0x0000000000000000:

1. **Appliances:**
   - MUST process the command if it's a valid command message (0x10-0x2F), except for Controller → Controller commands (0x14-0x15) which MUST be silently ignored
   - MUST NOT respond to broadcast-addressed commands (prevents bus collisions)
   - **Exception:** DISCOVERY_REQUEST (0x1F) triggers DEVICE_ANNOUNCE response
   - **Exception response behavior:**
     - Appliances MUST add random delay (0-50ms) before responding to avoid collisions
     - Response ADDRESS field: MUST use appliance's own address (source)

2. **Controllers:**
   - MAY use broadcast for:
     - Discovery (DISCOVERY_REQUEST to enumerate all appliances)
     - Simultaneous control (e.g., emergency stop all appliances)
     - Configuration distribution (set same config on all appliances)
   - MUST be prepared to receive multiple responses when using DISCOVERY_REQUEST
   - SHOULD implement timeout (100-200ms) to collect all discovery responses
   - MUST NOT expect responses for other broadcast commands (unless specifically designed to handle collisions)

3. **Monitors:**
   - Receive and log all broadcast packets
   - MUST NOT respond to broadcast messages

**Broadcast Command Response Rules:**

| Command Type | Appliance Responds? | Response ADDRESS | Notes |
|--------------|---------------------|------------------|-------|
| DISCOVERY_REQUEST (0x1F) | **YES** | Own address | Add random delay (0-50ms) |
| Config commands (0x10-0x13, 0x16-0x17) | No response | - | Config silently applied |
| STATE_COMMAND (0x20) | No response | - | All appliances execute |
| PING_REQUEST (0x2F) | No response | - | Would cause bus collision |
| All other commands | No response | - | Processed but no response |

**Collision Avoidance for DISCOVERY_REQUEST:**

DISCOVERY_REQUEST is the ONLY broadcast command that expects responses. To prevent bus collisions on multi-drop networks (RS-485):

- Each appliance MUST wait a random delay (0-50ms) before sending DEVICE_ANNOUNCE
- Controller MUST wait at least 100ms (preferably 200ms) to receive all responses
- Controller MAY receive partial/corrupted responses due to collisions on RS-485
- Controller SHOULD retry discovery if expected devices don't respond

**Note:** The random delay is required for all physical layers to ensure consistent behavior. While point-to-point UART and LIN networks don't strictly require collision avoidance, the universal delay simplifies implementation and ensures protocol compatibility across physical layers.

**Example - Broadcast Emergency Stop:**
```
Controller sends: STATE_COMMAND (mode=EMERGENCY, ADDRESS=broadcast)
Appliance 1: Receives, enters E_STOP, does NOT respond
Appliance 2: Receives, enters E_STOP, does NOT respond
Appliance 3: Receives, enters E_STOP, does NOT respond
Controller: Does not expect responses (broadcast command)
```

**Example - Broadcast Discovery:**
```
Controller sends: DISCOVERY_REQUEST (ADDRESS=broadcast)
Appliance 1: Waits 12ms (random), sends DEVICE_ANNOUNCE (ADDRESS=0xAABB...)
Appliance 2: Waits 37ms (random), sends DEVICE_ANNOUNCE (ADDRESS=0xCCDD...)
Appliance 3: Waits 5ms (random), sends DEVICE_ANNOUNCE (ADDRESS=0xEEFF...)
Controller: Receives 3 DEVICE_ANNOUNCE responses over 200ms window
```

**Rationale:**
- Enables multi-device networks (multiple appliances on one bus)
- Allows selective communication without physical addressing
- Supports device discovery and enumeration
- Prevents bus collisions for most broadcast commands
- Provides flexibility for point-to-point or broadcast communication

### CRC Calculation

**Algorithm:** CRC-16-CCITT
- **Polynomial:** 0x1021
- **Initial Value:** 0xFFFF
- **XOR Out:** 0x0000
- **Reflect In:** False
- **Reflect Out:** False

**CRC Coverage:** LENGTH + ADDRESS + MSG_TYPE + PAYLOAD fields
- CRC MUST NOT include START or END delimiters
- **CRC MUST be transmitted as 2 bytes: MSB first, then LSB (big-endian)**
- **IMPORTANT:** CRC is the ONLY field transmitted in big-endian byte order
- **All other multi-byte fields use little-endian** (see Data Type Encodings section)

### Byte Stuffing

To prevent confusion with START (0x7E) and END (0x7F) delimiters appearing in the data between START and END (LENGTH, ADDRESS, MSG_TYPE, PAYLOAD, and CRC):

**Escape Sequence:** Use `0x7D` as escape byte

| Original Byte | Escaped Sequence |
|---------------|------------------|
| 0x7E (START)  | 0x7D 0x5E       |
| 0x7F (END)    | 0x7D 0x5F       |
| 0x7D (ESC)    | 0x7D 0x5D       |

**Note:** Byte stuffing is applied AFTER CRC calculation and BEFORE framing.

---

## Message Types

### Configuration Commands (Controller → Appliance)

Configuration commands set persistent parameters on appliances. Changes SHOULD persist across power cycles.

| MSG_TYPE | Name | Description | Payload Size |
|----------|------|-------------|--------------|
| 0x10 | MOTOR_CONFIG | Configure motor controller parameters | 48 bytes |
| 0x11 | PUMP_CONFIG | Configure pump controller parameters | 16 bytes |
| 0x12 | TEMPERATURE_CONFIG | Configure temperature controller parameters | 48 bytes |
| 0x13 | GLOW_CONFIG | Configure glow plug parameters | 16 bytes |
| 0x14 | DATA_SUBSCRIPTION | Subscribe to data from appliance (for routing) | 8 bytes |
| 0x15 | DATA_UNSUBSCRIBE | Unsubscribe from appliance data | 8 bytes |
| 0x16 | TELEMETRY_CONFIG | Enable/disable telemetry broadcasts | 8 bytes |
| 0x17 | TIMEOUT_CONFIG | Configure communication timeout | 8 bytes |
| 0x18-0x1E | *Reserved* | Reserved for future configuration commands | - |
| 0x1F | DISCOVERY_REQUEST | Request device capabilities | 0 bytes |

### Control Commands (Controller → Appliance)

Control commands provide real-time operational control without changing persistent configuration.

| MSG_TYPE | Name | Description | Payload Size |
|----------|------|-------------|--------------|
| 0x20 | STATE_COMMAND | Set system mode/state | 8 bytes |
| 0x21 | MOTOR_COMMAND | Set motor RPM | 8 bytes |
| 0x22 | PUMP_COMMAND | Set pump rate | 8 bytes |
| 0x23 | GLOW_COMMAND | Control glow plug | 8 bytes |
| 0x24 | TEMPERATURE_COMMAND | Temperature controller control | 20 bytes |
| 0x25 | SEND_TELEMETRY | Request telemetry data (polling mode) | 8 bytes |
| 0x26-0x2E | *Reserved* | Reserved for future control commands | - |
| 0x2F | PING_REQUEST | Heartbeat/connectivity check | 0 bytes |

### Telemetry Data (Appliance → Controller)

Telemetry messages provide real-time status and sensor data from appliances.

| MSG_TYPE | Name | Description | Payload Size | Send Rate |
|----------|------|-------------|--------------|-----------|
| 0x30 | STATE_DATA | System state and status | 16 bytes | 2.5× telemetry interval |
| 0x31 | MOTOR_DATA | Motor telemetry | 32 bytes | Per telemetry interval |
| 0x32 | PUMP_DATA | Pump status and events | 16 bytes | On event |
| 0x33 | GLOW_DATA | Glow plug status | 12 bytes | On event |
| 0x34 | TEMPERATURE_DATA | Temperature readings | 32 bytes | Per telemetry interval |
| 0x35 | DEVICE_ANNOUNCE | Device capabilities announcement | 8 bytes | On DISCOVERY_REQUEST |
| 0x36-0x3E | *Reserved* | Reserved for future telemetry messages | - | - |
| 0x3F | PING_RESPONSE | Heartbeat response | 4 bytes | On request |

### Error Messages (Appliance → Controller)

Error messages indicate command validation failures. Appliances send error messages to controllers when commands cannot be processed.

| MSG_TYPE | Name | Description | Payload Size |
|----------|------|-------------|--------------|
| 0xE0 | ERROR_INVALID_CMD | Command validation failed | 4 bytes |
| 0xE1 | ERROR_STATE_REJECT | Command rejected by state machine | 4 bytes |
| 0xE2-0xEF | *Reserved* | Reserved for future error messages | - |

**Note:** CRC failures are not reported via error messages. Controllers should retransmit important commands if no response or acknowledgment is received. Critical commands like emergency stop MUST be sent repeatedly until confirmed.

### Reserved Message Types

Reserved message types (0x18-0x1E, 0x26-0x2E, 0x36-0x3E, 0xE2-0xEF) are placeholders for future protocol extensions.

**Handling:**
- Receivers MUST silently ignore packets with reserved message types
- Receivers MUST NOT send error responses for reserved message types
- This allows future protocol versions to add new message types without breaking existing implementations

### Undefined Message Types

Undefined message types (0x00-0x0F, 0x40-0xDF, 0xF0-0xFF) are not part of this protocol specification.

**Handling:**
- Receivers MUST silently ignore packets with undefined message types (same as reserved types)
- Receivers MUST NOT send error responses for undefined message types

---

## Configuration Command Formats

**IMPORTANT:** All multi-byte integers (u32, i32, u64, f64) use **little-endian** byte order unless otherwise specified. CRC-16 is the only exception (big-endian).

### 0x10 - MOTOR_CONFIG

Configure motor controller parameters including PWM, PID gains, and RPM limits.

**Payload Structure (48 bytes):**
```
+-------+------------+--------+--------+--------+---------+---------+
| motor | pwm_period | pid_kp | pid_ki | pid_kd | max_rpm | min_rpm |
+-------+------------+--------+--------+--------+---------+---------+
| i32   | u32        | f64    | f64    | f64    | i32     | i32     |
+-------+------------+--------+--------+--------+---------+---------+

+--------------+---------+
| min_pwm_duty | padding |
+--------------+---------+
| u32          | 4 bytes |
+--------------+---------+
```

**Fields:**
- **motor** (i32): Motor index (0 to motor_count-1, typically 0)
- **pwm_period** (u32): PWM period in microseconds (e.g., 50 μs = 20kHz)
- **pid_kp** (f64): Proportional gain for motor PID controller
- **pid_ki** (f64): Integral gain for motor PID controller
- **pid_kd** (f64): Derivative gain for motor PID controller
- **max_rpm** (i32): Maximum achievable RPM (e.g., 3400)
- **min_rpm** (i32): Minimum stable RPM (e.g., 800)
- **min_pwm_duty** (u32): Minimum PWM pulse width in microseconds (e.g., 10 μs)
- **padding** (4 bytes): Reserved for future use

**Default Values (Helios):**
- pwm_period: 50 μs (20 kHz)
- pid_kp: 4.0
- pid_ki: 12.0
- pid_kd: 0.1
- max_rpm: 3400
- min_rpm: 800
- min_pwm_duty: 10 μs

**Validation:**
- motor MUST be within device capability (0 to motor_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- pwm_period MUST be > 0; invalid value returns ERROR_INVALID_CMD with code 1
- max_rpm MUST be > min_rpm; invalid value returns ERROR_INVALID_CMD with code 1
- min_pwm_duty MUST be < pwm_period; invalid value returns ERROR_INVALID_CMD with code 1
- PID gains MAY be 0 (disables that term)
- PID gains MUST NOT be NaN or Infinity; invalid f64 values return ERROR_INVALID_CMD with code 1

### 0x11 - PUMP_CONFIG

Configure fuel pump parameters including pulse duration and recovery time.

**Payload Structure (16 bytes):**
```
+------+-----------+--------------+
| pump | pulse_ms  | recovery_ms  |
+------+-----------+--------------+
| i32  | u32       | u32          |
+------+-----------+--------------+

+---------+
| padding |
+---------+
| 4 bytes |
+---------+
```

**Fields:**
- **pump** (i32): Pump index (0 to pump_count-1, typically 0)
- **pulse_ms** (u32): Solenoid pulse duration in milliseconds (e.g., 50)
- **recovery_ms** (u32): Recovery time after pulse in milliseconds (e.g., 50)
- **padding** (4 bytes): Reserved for future use

**Default Values (Helios):**
- pulse_ms: 50
- recovery_ms: 50

**Validation:**
- pump MUST be within device capability (0 to pump_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- pulse_ms MUST be > 0; invalid value returns ERROR_INVALID_CMD with code 1
- recovery_ms MUST be > 0; invalid value returns ERROR_INVALID_CMD with code 1
- Minimum pump rate = pulse_ms + recovery_ms (typically 100ms minimum)

**Rationale:**
- Pulse duration controls fuel delivery per cycle
- Recovery time prevents solenoid overheating and ensures complete valve closure

### 0x12 - TEMPERATURE_CONFIG

Configure temperature controller parameters including PID gains and sampling.

**Payload Structure (48 bytes):**
```
+-------------+--------+--------+--------+--------------+-----------+
| temperature | pid_kp | pid_ki | pid_kd | sample_count | read_rate |
+-------------+--------+--------+--------+--------------+-----------+
| i32         | f64    | f64    | f64    | u32          | u32       |
+-------------+--------+--------+--------+--------------+-----------+

+----------+
| padding  |
+----------+
| 12 bytes |
+----------+
```

**Fields:**
- **temperature** (i32): Temperature sensor index (0 to temperature_count-1, typically 0)
- **pid_kp** (f64): Proportional gain for temperature PID controller
- **pid_ki** (f64): Integral gain for temperature PID controller
- **pid_kd** (f64): Derivative gain for temperature PID controller
- **sample_count** (u32): Number of samples for moving average filter (e.g., 60)
- **read_rate** (u32): Temperature reading interval in milliseconds (e.g., 50)
- **padding** (12 bytes): Reserved for future use

**Default Values (Helios):**
- pid_kp: 100.0
- pid_ki: 10.0
- pid_kd: 5.0
- sample_count: 60 (provides ~3 second warmup at 50ms read rate)
- read_rate: 50 ms

**Validation:**
- temperature MUST be within device capability (0 to temperature_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- sample_count MUST be > 0; invalid value returns ERROR_INVALID_CMD with code 1
- read_rate MUST be > 0; invalid value returns ERROR_INVALID_CMD with code 1
- PID gains MAY be 0 (disables that term)
- PID gains MUST NOT be NaN or Infinity; invalid f64 values return ERROR_INVALID_CMD with code 1

**Rationale:**
- Moving average filter reduces sensor noise
- Warmup time = sample_count × read_rate (60 × 50ms = 3 seconds)
- PID gains tuned for inverted control (higher temperature → higher RPM for cooling)

### 0x13 - GLOW_CONFIG

Configure glow plug parameters.

**Payload Structure (16 bytes):**
```
+------+--------------+---------+
| glow | max_duration | padding |
+------+--------------+---------+
| i32  | u32          | 8 bytes |
+------+--------------+---------+
```

**Fields:**
- **glow** (i32): Glow plug index (0 to glow_count-1, typically 0)
- **max_duration** (u32): Maximum allowed glow duration in milliseconds (e.g., 300000 = 5 minutes)
- **padding** (8 bytes): Reserved for future use

**Default Values (Helios):**
- max_duration: 300000 ms (5 minutes)

**Validation:**
- glow MUST be within device capability (0 to glow_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- max_duration MUST be > 0; invalid value returns ERROR_INVALID_CMD with code 1
- RECOMMENDED: max_duration ≤ 300000 ms (5 minutes) for safety

**Rationale:**
- Prevents indefinite glow plug operation
- Safety timeout for preheat phase

### 0x14 - DATA_SUBSCRIPTION

**Direction:** Controller → Controller (for routing scenarios)

**Note:** This is NOT a Controller → Appliance command. This command is sent from one controller (subscriber) to another controller (router) to establish data forwarding over a point-to-point connection.

Subscribe to receive copies of ALL data messages from a specific appliance. Used for controller routing scenarios where one controller (router) is physically connected to an appliance, and another controller (subscriber) wants to receive telemetry data through the router.

**Payload Structure (8 bytes):**
```
+-------------------+
| appliance_address |
+-------------------+
| u64               |
+-------------------+
```

**Fields:**
- **appliance_address** (u64): Address of the appliance to subscribe to
  - MUST be a valid appliance address (not a controller address)
  - MUST NOT be the broadcast address (0x0000000000000000)

**Behavior:**

When a controller (router) receives this command over a point-to-point connection:
1. Associates the subscription with the connection on which it was received
2. Adds the connection to the routing table for the specified appliance_address
3. When recognized data messages (0x30-0x35, 0x3F) are received from appliance_address, forwards those messages to the subscribed connection
   - Routers MUST NOT forward packets with reserved message types (0x18-0x1E, 0x26-0x2E, 0x36-0x3E, 0xE2-0xEF)

**Connection-Based Routing:**

In a multi-drop topology, there is one controller and multiple appliances on the same bus. When a remote controller connects to another controller acting as a router, the connection is point-to-point (WiFi, Bluetooth, TCP, etc.). Subscriptions are associated with the connection, not a packet-level source address.

**Multi-Hop Routing Example:**

Controller A (WiFi) wants telemetry from Appliance C (LIN bus) through Controller B (WiFi+LIN bridge):

1. Controller A establishes point-to-point connection to Controller B (via WiFi)

2. Controller A sends DATA_SUBSCRIPTION to Controller B:
   - ADDRESS: Controller B's address
   - appliance_address: Appliance C's address

3. Controller B receives subscription, associates it with the WiFi connection to Controller A

4. When Appliance C sends telemetry (e.g., MOTOR_DATA, TEMPERATURE_DATA, STATE_DATA):
   - Controller B receives packet from LIN bus
   - Controller B checks routing table, sees Controller A's connection is subscribed
   - Controller B forwards packet to Controller A over the WiFi connection

5. Controller A receives telemetry from Appliance C through Controller B

**Validation:**

When a router controller receives this command, it MUST validate:

1. **appliance_address Validity:**
   - MUST NOT be the broadcast address (0x0000000000000000)
   - Router MAY defer validation until first data message from that appliance

2. **Subscription Table Capacity:**
   - Routers MAY limit the number of active subscriptions (implementation-defined)
   - RECOMMENDED minimum: 10 concurrent subscriptions per router

3. **Duplicate Subscriptions:**
   - If the same connection already has a subscription to the same appliance, the subscription is refreshed (timeout reset)

**Important Notes:**
- Subscriptions are NOT persistent - they are lost on power cycle, reset, or connection close
- Router controllers SHOULD implement subscription timeout (recommend 60 seconds without PING_RESPONSE from subscriber)
- Subscriber controllers SHOULD periodically re-send DATA_SUBSCRIPTION to maintain routing
- Controller-to-controller links are expected to be high-bandwidth (WiFi, Bluetooth, TCP); lower-bandwidth links (e.g., LoRa) may require future protocol extensions for filtering

### 0x15 - DATA_UNSUBSCRIBE

**Direction:** Controller → Controller (for routing scenarios)

**Note:** This is NOT a Controller → Appliance command. This command is sent from one controller (subscriber) to another controller (router) to remove data forwarding.

Remove a previously established data subscription. The subscribing controller no longer wants to receive data messages from the specified appliance.

**Payload Structure (8 bytes):**
```
+-------------------+
| appliance_address |
+-------------------+
| u64               |
+-------------------+
```

**Fields:**
- **appliance_address** (u64): Address of the appliance to unsubscribe from
  - If no subscription exists for this appliance on the connection, command is silently ignored

**Behavior:**

When a controller (router) receives this command over a point-to-point connection:
1. Identifies the connection on which the command was received
2. Removes the subscription for the specified appliance_address from that connection
3. No longer forwards data messages from that appliance to the connection

**Use Cases:**
- Clean shutdown when subscriber controller is disconnecting
- Reduce bandwidth usage when telemetry is no longer needed

### 0x16 - TELEMETRY_CONFIG

Enable or disable periodic telemetry broadcasts and configure broadcast interval.

**Payload Structure (8 bytes):**
```
+-------------------+-------------+
| telemetry_enabled | interval_ms |
+-------------------+-------------+
| u32               | u32         |
+-------------------+-------------+
```

**Fields:**
- **telemetry_enabled** (u32): Telemetry broadcast control
  - 0 = Disable telemetry broadcasts (interval_ms ignored)
  - 1 = Enable telemetry broadcasts at specified interval
- **interval_ms** (u32): Telemetry broadcast interval in milliseconds
  - 0 = Polling mode (telemetry is enabled but not automatically broadcast; use SEND_TELEMETRY command (0x25) to request data)
  - MUST be within range: 0 or 100-5000 ms
  - RECOMMENDED: 100 ms for automatic broadcasts
  - Values 1-99 SHALL be clamped to 100 ms
  - Values above 5000 SHALL be clamped to 5000 ms

**Polling Mode:**

When interval_ms=0, the appliance enables telemetry but does NOT automatically broadcast data messages. Instead, the controller must use the SEND_TELEMETRY command (0x25) to explicitly request telemetry data. This is useful for:
- **Multi-appliance topologies:** Prevents bus collisions when multiple appliances would otherwise broadcast simultaneously
- **Bandwidth-constrained links:** Controller can request only the specific data it needs
- **Power-sensitive applications:** Reduces unnecessary transmissions

In polling mode:
- Appliance MUST send data messages only in response to SEND_TELEMETRY commands
- PING_RESPONSE (0x3F) is always allowed in response to PING_REQUEST
- E_STOP broadcasts are still permitted (safety takes precedence)
- **IMPORTANT:** Controllers MUST still send periodic PING_REQUEST to prevent communication timeout. SEND_TELEMETRY does NOT reset the timeout timer.

**Telemetry Messages (Broadcast Mode, interval_ms > 0):**

When telemetry is enabled in broadcast mode, the appliance sends individual data messages automatically:
- **MOTOR_DATA** (0x31): Sent per telemetry interval for each motor
- **TEMPERATURE_DATA** (0x34): Sent per telemetry interval for each temperature sensor
- **STATE_DATA** (0x30): Sent at 2.5× the telemetry interval (e.g., every 250ms at 100ms interval)
- **PUMP_DATA** (0x32): Sent on pump events (cycle start, pulse end, cycle end)
- **GLOW_DATA** (0x33): Sent on glow plug state changes

For appliances with multiple motors or temperature sensors, a separate message is sent for each device at the configured interval.

**Telemetry Messages (Polling Mode, interval_ms = 0):**

In polling mode, ALL data messages (including PUMP_DATA and GLOW_DATA) are ONLY sent in response to SEND_TELEMETRY commands. Event-driven messages are NOT sent automatically; controllers must explicitly request pump and glow plug status when needed.

**Default State:** Telemetry broadcasts are **disabled** on boot

**Data Message Restriction:**
- **IMPORTANT:** Appliances SHALL NOT send any data messages (0x30-0x3F) until a TELEMETRY_CONFIG command with telemetry_enabled=1 has been received
- **Exceptions:**
  - PING_RESPONSE (0x3F), which may be sent at any time in response to PING_REQUEST
  - E_STOP state broadcasts, which are sent regardless of TELEMETRY_CONFIG state (see "Emergency Stop Behavior")
- This prevents unsolicited data messages before the controller is ready to receive them
- Violating this restriction will cause decoder synchronization issues on the controller

**Auto-Disable Behavior:**
- Telemetry broadcasts are automatically disabled when the communication timeout elapses (see TIMEOUT_CONFIG)
- The same timer controls both IDLE mode transition and telemetry auto-disable
- Controller must re-enable telemetry after reconnecting

**Rationale:**
- Disabling telemetry on boot prevents synchronization issues during initial connection
- Controller can establish communication, send initial commands, then enable telemetry when ready
- Auto-disable on timeout prevents unnecessary transmissions when controller is absent
- Reduces power consumption and bus traffic when controller is disconnected

**Recovery Use Case:**
- If the controller's receive buffer becomes out of sync (repeated decode errors), it can send TELEMETRY_CONFIG (enable=0) to stop the flood of incoming telemetry data
- This allows the controller to clear its receive buffer, reset the decoder state, and resynchronize
- Once synchronized, the controller can re-enable telemetry with TELEMETRY_CONFIG (enable=1)
- This is particularly useful during boot or after communication errors when packet boundaries are lost

### 0x17 - TIMEOUT_CONFIG

Configure communication timeout behavior. Timeout mode provides a safety mechanism that transitions the appliance to IDLE mode and disables telemetry if communication with the controller is lost.

**Payload Structure (8 bytes):**
```
+---------+------------+
| enabled | timeout_ms |
+---------+------------+
| u32     | u32        |
+---------+------------+
```

**Fields:**
- **enabled** (u32): Timeout mode control
  - 0 = Disable timeout mode (appliance runs indefinitely without controller)
  - 1 = Enable timeout mode (default)
- **timeout_ms** (u32): Timeout interval in milliseconds
  - MUST be within range: 5000-60000 ms (5-60 seconds)
  - RECOMMENDED: 30000 ms (30 seconds, default)
  - Values outside range SHALL be clamped to nearest valid value

**Default State:** Timeout mode is **enabled** on boot with 30-second timeout

**Behavior When Enabled:**
- Appliance tracks time since last PING_REQUEST received
- **IMPORTANT:** Only PING_REQUEST resets the timeout timer. Other commands (STATE_COMMAND, MOTOR_COMMAND, SEND_TELEMETRY, etc.) do NOT reset the timer.
- If timeout_ms elapses without receiving PING_REQUEST:
  - Appliance automatically transitions to IDLE mode
  - Telemetry broadcasts are automatically disabled
- Prevents continued operation without controller supervision

**Disabling Timeout Mode:**
- Setting enabled=0 allows "headless" operation where the appliance runs without an attached controller
- Use case: Appliance configured via initial commands, then controller disconnects
- **WARNING:** Disabling timeout mode removes a critical safety feature - use with caution

**Safety Rationale:**
- Ensures appliance doesn't operate indefinitely without controller supervision
- Critical for burner systems where loss of communication requires safe shutdown
- IDLE mode performs proper cooldown if temperature is elevated
- 30-second default timeout allows for brief disconnections without unnecessary cooldown cycles

### 0x1F - DISCOVERY_REQUEST

Request device capabilities from all appliances on the bus.

**Payload:** None (0 bytes)

**Usage:**
- Controller MUST send this command with ADDRESS field set to broadcast address (0x0000000000000000)
- Addressed (non-broadcast) DISCOVERY_REQUEST is not supported and SHALL be ignored by appliances
- All appliances on the bus MUST respond with DEVICE_ANNOUNCE (0x35)
- Appliances MUST wait a random delay (0-50ms) before responding to avoid bus collisions
- Response includes device address and capability counts

**Response:** DEVICE_ANNOUNCE (0x35) from each appliance (after 0-50ms random delay)

**Example Use Cases:**
- Network discovery on startup
- Detecting new devices added to the bus
- Enumerating available resources (motors, sensors, etc.)
- Building dynamic UI based on available devices

---

## Control Command Formats

**IMPORTANT:** All multi-byte integers (u32, i32, u64, f64) use **little-endian** byte order unless otherwise specified. CRC-16 is the only exception (big-endian).

### 0x20 - STATE_COMMAND

Set system operating mode.

**Payload Structure (8 bytes):**
```
+------+----------+
| mode | argument |
+------+----------+
| u32  | i32      |
+------+----------+
```

**Fields:**
- **mode** (u32): Operating mode
  - 0 = IDLE_MODE
  - 1 = FAN_MODE
  - 2 = HEAT_MODE
  - 3 = EMERGENCY
- **argument** (i32): Mode-specific parameter
  - FAN_MODE: Target RPM (0 or 800-3400). If 0, treated as IDLE_MODE.
  - HEAT_MODE: Pump rate in milliseconds. If 0, treated as IDLE_MODE. Valid range: (pulse_ms + recovery_ms) to 5000ms (see PUMP_CONFIG).
  - IDLE_MODE/EMERGENCY: Ignored (set to 0)

**Example:** Enter fan mode at 2500 RPM
```
7E 08 [ADDRESS] 20 01 00 00 00 C4 09 00 00 [CRC-H] [CRC-L] 7F
                   ^^mode=1    ^^argument=2500 (little-endian)
```

**Mode vs State Distinction:**

Command modes (sent via STATE_COMMAND) trigger state machine transitions but do NOT directly set the state. The appliance state machine determines the actual state based on the requested mode and current conditions.

**Mode → State Mapping:**

| Command Mode | Possible Resulting States | Notes |
|--------------|---------------------------|-------|
| IDLE_MODE (0) | IDLE, COOLING | If temperature is elevated, enters COOLING first |
| FAN_MODE (1) | BLOWING | Direct transition |
| HEAT_MODE (2) | PREHEAT → PREHEAT_STAGE_2 → HEATING | Progresses through heating sequence |
| EMERGENCY (3) | E_STOP | Immediate transition, requires power cycle to exit |

**States Not Settable via Command:**
- INITIALIZING (0): Only occurs during boot
- ERROR (7): Set by internal fault detection

See STATE_DATA (0x30) for complete list of state values reported by the appliance.

**Validation:**
- Invalid mode values (>3) MUST return ERROR_INVALID_CMD with code 1
- For FAN_MODE: argument values between 1 and min_rpm-1 are invalid and MUST return ERROR_INVALID_CMD with code 1 (e.g., 1-799 if min_rpm=800, see MOTOR_CONFIG)
- For HEAT_MODE: argument values between 1 and (pulse_ms + recovery_ms - 1) are invalid and MUST return ERROR_INVALID_CMD with code 1 (see PUMP_CONFIG)

### 0x21 - MOTOR_COMMAND

Control motor (fan) speed.

**Payload Structure (8 bytes):**
```
+-------+------+
| motor | rpm  |
+-------+------+
| i32   | i32  |
+-------+------+
```

**Fields:**
- **motor** (i32): Motor index (0 to motor_count-1, typically 0)
- **rpm** (i32): Target RPM (0 = stop, min_rpm to max_rpm = run, typically 800-3400)

**Validation:**
- motor MUST be within device capability (0 to motor_count-1)
- rpm MUST be 0 OR within motor's configured min/max range (see MOTOR_CONFIG)
- rpm values between 1 and min_rpm-1 (e.g., 1-799) are INVALID
- Invalid motor index or rpm MUST return ERROR_INVALID_CMD with code 2 (invalid device index) or code 1 (invalid parameter value)

### 0x22 - PUMP_COMMAND

Control fuel pump rate.

**Payload Structure (8 bytes):**
```
+------+---------+
| pump | rate_ms |
+------+---------+
| i32  | i32     |
+------+---------+
```

**Fields:**
- **pump** (i32): Pump index (0 to pump_count-1, typically 0)
- **rate_ms** (i32): Pulse interval in milliseconds (0 = stop, minimum = pulse_ms + recovery_ms, maximum = 5000)

**Validation:**
- pump MUST be within device capability (0 to pump_count-1)
- rate_ms MUST be 0 OR within valid range: (pulse_ms + recovery_ms) to 5000ms
- rate_ms values between 1 and (pulse_ms + recovery_ms - 1) are INVALID
- Invalid pump index or rate_ms MUST return ERROR_INVALID_CMD with code 2 (invalid device index) or code 1 (invalid parameter value)

### 0x23 - GLOW_COMMAND

Control glow plug heating.

**Payload Structure (8 bytes):**
```
+------+----------+
| glow | duration |
+------+----------+
| i32  | i32      |
+------+----------+
```

**Fields:**
- **glow** (i32): Glow plug index (0 to glow_count-1, typically 0)
- **duration** (i32): Burn duration in milliseconds (0 = immediately extinguish, >0 = light for specified duration, max = configured max_duration)

**Behavior:**
- The glow plug automatically turns off when the specified duration expires
- Duration expiry triggers a GLOW_DATA message with lit=0 (see GLOW_DATA)
- Duration MUST NOT be extended: sending a non-zero duration to an already lit glow plug returns an error
- Sending duration=0 to a lit glow plug immediately extinguishes it and triggers GLOW_DATA with lit=0

**Validation:**
- glow MUST be within device capability (0 to glow_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- duration MUST be 0 to max_duration; invalid value returns ERROR_INVALID_CMD with code 1
- Attempting to light an already lit glow plug (duration > 0 when lit=1) MUST return:
  - ERROR_INVALID_CMD with code 1 if sent as a manual command (direct GLOW_COMMAND)
  - ERROR_STATE_REJECT if the appliance is in HEAT_MODE (state machine controls glow plug)

### 0x24 - TEMPERATURE_COMMAND

Configure temperature controller operation.

**Payload Structure (20 bytes):**
```
+-------------+------+-------------+------------------+
| temperature | type | motor_index | target_temperature      |
+-------------+------+-------------+------------------+
| i32         | u32  | i32         | f64              |
+-------------+------+-------------+------------------+
```

**Fields:**
- **temperature** (i32): Temperature sensor index (0 to temperature_count-1, typically 0)
- **type** (u32): Command type
  - 0 = WATCH_MOTOR (associate with motor)
  - 1 = UNWATCH_MOTOR (stop monitoring)
  - 2 = ENABLE_RPM_CONTROL (enable PID)
  - 3 = DISABLE_RPM_CONTROL (disable PID)
  - 4 = SET_TARGET_TEMPERATURE (set temperature target)
- **motor_index** (i32): Motor to control (used with WATCH_MOTOR only)
- **target_temperature** (f64): Target temperature in Celsius (used with SET_TARGET_TEMPERATURE only)

**Unused Field Handling:**
- Controllers SHOULD set unused fields to zero
- Appliances MUST ignore unused fields for their respective command types
- Example: For ENABLE_RPM_CONTROL (type=2), both motor_index and target_temperature are unused and ignored

**Note:** f64 is IEEE 754 double-precision, little-endian byte order

**Validation:**
- Invalid type values (>4) MUST return ERROR_INVALID_CMD with code 1
- temperature index MUST be within device capability (0 to temperature_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- motor_index (for WATCH_MOTOR) MUST be within device capability (0 to motor_count-1); invalid index returns ERROR_INVALID_CMD with code 2
- target_temperature (for SET_TARGET_TEMPERATURE) MUST NOT be NaN or Infinity; invalid f64 values return ERROR_INVALID_CMD with code 1

**SET_TARGET_TEMPERATURE Restrictions:**
- SET_TARGET_TEMPERATURE (type=4) MAY only be sent when the appliance is in HEATING state (state=5)
- Sending SET_TARGET_TEMPERATURE in any other state MUST return ERROR_STATE_REJECT
- Appliance firmware MUST provide a default target_temperature value (typically 210°C for heat exchanger temperature sensor)
- The default value is used until the controller sends SET_TARGET_TEMPERATURE

### 0x25 - SEND_TELEMETRY

Request specific telemetry data from an appliance. This command is used in polling mode (when TELEMETRY_CONFIG has interval_ms=0) to explicitly request telemetry data instead of relying on automatic broadcasts.

**Payload Structure (8 bytes):**
```
+----------------+-------+----------+
| telemetry_type | index | reserved |
+----------------+-------+----------+
| u8             | u8    | 6 bytes  |
+----------------+-------+----------+
```

**Fields:**
- **telemetry_type** (u8): Type of telemetry data to request
  - 0 = STATE (request STATE_DATA)
  - 1 = MOTOR (request MOTOR_DATA)
  - 2 = TEMPERATURE (request TEMPERATURE_DATA)
  - 3 = PUMP (request PUMP_DATA)
  - 4 = GLOW (request GLOW_DATA)
- **index** (u8): Peripheral index to request
  - 0 to (peripheral_count-1) = Request data for specific peripheral at this index
  - 0xFF (255) = Request data for ALL peripherals of the specified type
  - For STATE (type=0): index is ignored; controllers SHOULD send 0
- **reserved** (6 bytes): Reserved for future use; controllers SHOULD send zeros, appliances MUST ignore this field

**Behavior:**

When an appliance receives SEND_TELEMETRY:
1. If telemetry is disabled or in broadcast mode, the command MUST be ignored (takes precedence over all other checks)
2. If telemetry_type is invalid (>4), appliance SHALL respond with ERROR_INVALID_CMD
3. If telemetry_type is STATE (0), index is ignored and appliance SHALL send exactly one STATE_DATA message
4. For other telemetry types: if index is out of range for the device (e.g., index=2 but device has only 1 motor) and index is not 0xFF, appliance SHALL respond with ERROR_INVALID_CMD
5. If index=0xFF, appliance SHALL send one data message per peripheral of the specified type
6. If index is valid (0 to peripheral_count-1), appliance SHALL send exactly one data message for that peripheral

**Response Messages:**
- STATE (0): STATE_DATA (0x30) - always exactly one message
- MOTOR (1): MOTOR_DATA (0x31) - one per motor if index=0xFF
- TEMPERATURE (2): TEMPERATURE_DATA (0x34) - one per temperature sensor if index=0xFF
- PUMP (3): PUMP_DATA (0x32) - one per pump if index=0xFF
- GLOW (4): GLOW_DATA (0x33) - one per glow plug if index=0xFF

**Example Usage:**
```
Controller → Appliance:  SEND_TELEMETRY (type=1, index=0xFF)  # Request all motors
Appliance → Controller:  MOTOR_DATA (index=0, rpm=2500, ...)
Appliance → Controller:  MOTOR_DATA (index=1, rpm=2480, ...)

Controller → Appliance:  SEND_TELEMETRY (type=2, index=0)   # Request temperature sensor 0
Appliance → Controller:  TEMPERATURE_DATA (index=0, reading=225.5, ...)
```

**Notes:**
- SEND_TELEMETRY does NOT reset the communication timeout timer (only PING_REQUEST does)
- This command is only meaningful when telemetry is in polling mode (interval_ms=0)
- If telemetry is disabled (telemetry_enabled=0), this command MUST be ignored
- If telemetry is in automatic broadcast mode (interval_ms>0), this command MUST be ignored
- **Rationale:** Silent ignore (no error response) prevents excess bus chatter and avoids collisions in multi-device networks

### 0x2F - PING_REQUEST

Connectivity check / heartbeat.

**Payload:** None (0 bytes)

**Response:** PING_RESPONSE (0x3F) with uptime

**Important:** PING_REQUEST resets the communication timeout timer. If timeout mode is enabled (see TIMEOUT_CONFIG) and no PING_REQUEST is received within the configured timeout interval (default 30 seconds), the appliance SHALL automatically transition to IDLE mode and disable telemetry broadcasts.

---

## Telemetry Data Formats

**IMPORTANT:** All multi-byte integers (u32, i32, u64, f64) use **little-endian** byte order unless otherwise specified. CRC-16 is the only exception (big-endian).

### 0x30 - STATE_DATA

System state and error status.

**Payload Structure (16 bytes):**
```
+-------+------+-------+-----------+----------+
| error | code | state | timestamp | padding  |
+-------+------+-------+-----------+----------+
| u8    | i32  | u32   | u32       | 3 bytes  |
+-------+------+-------+-----------+----------+
```

**Fields:**
- **error** (u8): Error flag (0 = no error, 1 = error)
- **code** (i32): Error code (see table below)
- **state** (u32): Current system state (state machine state, not command mode)
  - 0 = INITIALIZING
  - 1 = IDLE
  - 2 = BLOWING
  - 3 = PREHEAT
  - 4 = PREHEAT_STAGE_2
  - 5 = HEATING
  - 6 = COOLING
  - 7 = ERROR
  - 8 = E_STOP
- **timestamp** (u32): Timestamp in milliseconds since boot (wraps at 2^32)
- **padding** (3 bytes): Reserved for alignment

**Note:** State values represent internal state machine states, which differ from the command modes sent via STATE_COMMAND (0x20). A single command mode may result in multiple state transitions (e.g., HEAT_MODE → PREHEAT → PREHEAT_STAGE_2 → HEATING). See STATE_COMMAND for mode-to-state mapping.

**Standard Error Codes:**

| Code | Name | Description |
|------|------|-------------|
| 0 | NONE | No error |
| 1 | OVERHEAT | Temperature exceeded safety limit |
| 2 | SENSOR_FAULT | Temperature sensor failure or invalid reading |
| 3 | IGNITION_FAIL | Failed to ignite after preheat phase |
| 4 | FLAME_OUT | Flame detected during heating but subsequently lost |
| 5 | MOTOR_STALL | Motor RPM dropped below minimum threshold |
| 6 | PUMP_FAULT | Pump operation failure |
| 7 | COMMANDED_ESTOP | Emergency stop commanded by controller (via STATE_COMMAND mode=EMERGENCY) |

**Note:** Communication timeout is not an error code. When communication is lost, the appliance automatically transitions to IDLE mode (see TIMEOUT_CONFIG). Codes 8-255 are reserved for future use. Negative codes (i32 < 0) may be used for application-specific errors.

**Error Flag and Code Relationship:**
- Appliances MUST NOT send a code other than 0 when the error flag is 0
- Controllers SHOULD ignore the error code unless the error flag is set (error=1)

**Send Rate:** 2.5× telemetry interval (250ms at default 100ms interval)

### 0x31 - MOTOR_DATA

Motor telemetry including RPM and PWM feedback.

**Payload Structure (32 bytes):**
```
+-------+-----------+------+--------+---------+---------+------+---------+
| motor | timestamp | rpm  | target | max_rpm | min_rpm | pwm  | pwm_max |
+-------+-----------+------+--------+---------+---------+------+---------+
| i32   | u32       | i32  | i32    | i32     | i32     | i32  | i32     |
+-------+-----------+------+--------+---------+---------+------+---------+
```

**Fields:**
- **motor** (i32): Motor index
- **timestamp** (u32): Reading timestamp in milliseconds since boot
- **rpm** (i32): Current measured RPM
- **target** (i32): Target RPM setpoint
- **max_rpm** (i32): Maximum achievable RPM (typically 3400)
- **min_rpm** (i32): Minimum stable RPM (typically 800)
- **pwm** (i32): Current PWM pulse width in microseconds
- **pwm_max** (i32): PWM period in microseconds

**Send Rate:** Per telemetry interval (100ms at default)

### 0x32 - PUMP_DATA

Fuel pump status and events.

**Payload Structure (16 bytes):**
```
+------+-----------+------+------+
| pump | timestamp | type | rate |
+------+-----------+------+------+
| i32  | u32       | u32  | i32  |
+------+-----------+------+------+
```

**Fields:**
- **pump** (i32): Pump index
- **timestamp** (u32): Event timestamp in milliseconds since boot
- **type** (u32): Event type
  - 0 = INITIALIZING
  - 1 = READY
  - 2 = ERROR
  - 3 = CYCLE_START
  - 4 = PULSE_END
  - 5 = CYCLE_END
- **rate** (i32): Current pump rate in milliseconds

**Send Rate:** On event (state changes, cycle events)

**Note:** This message provides both pump events (type field) and current pump rate, enabling controllers to monitor pump operation in real-time.

### 0x33 - GLOW_DATA

Glow plug status.

**Payload Structure (12 bytes):**
```
+------+-----------+-----+---------+
| glow | timestamp | lit | padding |
+------+-----------+-----+---------+
| i32  | u32       | u8  | 3 bytes |
+------+-----------+-----+---------+
```

**Fields:**
- **glow** (i32): Glow plug index
- **timestamp** (u32): Status timestamp in milliseconds since boot
- **lit** (u8): Lit status (0 = off, 1 = lit)
- **padding** (3 bytes): Reserved for alignment

**Send Rate:** On event (on/off transitions)

**Events that trigger GLOW_DATA:**
- Glow plug turns on (GLOW_COMMAND with duration > 0): sends lit=1
- Glow plug turns off via command (GLOW_COMMAND with duration=0): sends lit=0
- Glow plug turns off automatically (duration expired): sends lit=0

### 0x34 - TEMPERATURE_DATA

Temperature sensor readings and PID control status.

**Payload Structure (32 bytes):**
```
+-------------+-----------+---------+------------------+
| temperature | timestamp | reading | ctrl_rpm_by_temperature |
+-------------+-----------+---------+------------------+
| i32         | u32       | f64     | u8               |
+-------------+-----------+---------+------------------+

+---------------+--------------------+---------+
| watched_motor | target_temperature | padding |
+---------------+--------------------+---------+
| i32           | f64                | 3 bytes |
+---------------+--------------------+---------+
```

**Fields:**
- **temperature** (i32): Temperature sensor index
- **timestamp** (u32): Reading timestamp in milliseconds since boot
- **reading** (f64): Current temperature reading in Celsius
- **ctrl_rpm_by_temperature** (u8): Temperature-based RPM control active (0 = off, 1 = on)
- **watched_motor** (i32): Motor being controlled (-1 = none, or last configured motor index when ctrl_rpm_by_temperature=0)
- **target_temperature** (f64): Target temperature for PID control (firmware default value until SET_TARGET_TEMPERATURE received)
- **padding** (3 bytes): Reserved for alignment

**Send Rate:** Per telemetry interval (100ms at default, after sample warmup period)

**Invalid Reading Handling:**
- If the temperature sensor returns an invalid value (NaN, Infinity, or sensor fault), the appliance MUST NOT transmit invalid f64 values
- Instead, the appliance MUST transition to ERROR state and send STATE_DATA with error code SENSOR_FAULT (2)
- This applies to all f64 temperature fields (reading, target_temperature)

### 0x35 - DEVICE_ANNOUNCE

Device capabilities announcement sent in response to DISCOVERY_REQUEST.

**Payload Structure (8 bytes):**
```
+-------------+-------------------+------------+------------+
| motor_count | temperature_count | pump_count | glow_count |
+-------------+-------------------+------------+------------+
| u8          | u8                | u8         | u8         |
+-------------+-------------------+------------+------------+

+---------+
| padding |
+---------+
| 4 bytes |
+---------+
```

**Fields:**
- **motor_count** (u8): Number of motors this device has (1-255)
- **temperature_count** (u8): Number of temperature sensors (1-255)
- **pump_count** (u8): Number of pumps (1-255)
- **glow_count** (u8): Number of glow plugs (1-255)
- **padding** (4 bytes): Reserved for future expansion (firmware version, device type, etc.)

**Validation:**
- All counts MUST be in range [1, 255]
- Implementations MUST reject DEVICE_ANNOUNCE with any count equal to 0:
  - Controllers with a UI SHOULD display a diagnostic message to the user
  - Routers MUST forward the packet to subscribed controllers (allowing them to display diagnostics)
  - The packet MUST NOT be used for device enumeration

**Send Rate:** On DISCOVERY_REQUEST only

**Behavior:**
- Sent in response to DISCOVERY_REQUEST with broadcast address
- ADDRESS field contains the appliance's unique 64-bit address
- Appliances MUST wait a random delay (0-50ms) before responding to avoid bus collisions
- Controllers MUST wait at least 100ms (preferably 200ms) to receive all responses
- Controllers SHOULD be prepared to receive multiple DEVICE_ANNOUNCE messages

**Example:**
```
Typical appliance (Helios ICU) responds with:
  motor_count: 1
  temperature_count: 1
  pump_count: 1
  glow_count: 1

  (This is the most common configuration)

Multi-burner appliance responds with:
  motor_count: 3
  temperature_count: 3
  pump_count: 2
  glow_count: 2

  (Less common - industrial/commercial applications)
```

**Rationale:**
- Enables dynamic discovery of network topology
- Allows controllers to build appropriate UI based on actual hardware
- Supports heterogeneous networks (different appliance types on same bus)
- Eliminates need for manual configuration of device capabilities

### 0x3F - PING_RESPONSE

Heartbeat response with system uptime.

**Payload Structure (4 bytes):**
```
+-----------+
| uptime_ms |
+-----------+
| u32       |
+-----------+
```

**Fields:**
- **uptime_ms** (u32): System uptime in milliseconds (wraps at 2^32)

**Send Rate:** On request (response to PING_REQUEST)

---

## Error Message Formats

**IMPORTANT:** All multi-byte integers (u32, i32, u64, f64) use **little-endian** byte order unless otherwise specified. CRC-16 is the only exception (big-endian).

All error messages share the same structure:

**Payload Structure (4 bytes):**
```
+------------+
| error_code |
+------------+
| i32        |
+------------+
```

### 0xE0 - ERROR_INVALID_CMD
Command validation failed.

**Direction:** Appliance → Controller

**error_code:**
- 0 = Reserved for future use
- 1 = Invalid parameter value
- 2 = Invalid device index

**ADDRESS field:** Appliance's own address (source)

### 0xE1 - ERROR_STATE_REJECT
Command rejected by appliance state machine.

**Direction:** Appliance → Controller

**error_code:**
- Current state that rejected the command (see STATE_DATA state values)

**ADDRESS field:** Appliance's own address (source)

**Recovery:** Controllers should handle ERROR_STATE_REJECT by either:
- Waiting for the appliance to reach an appropriate state, or
- Retrying the command after addressing the state conflict

---

## Communication Patterns

### 1. Command-Response

Controller sends command, appliance may respond with error:

```
Controller → Appliance:  [STATE_COMMAND: Set HEAT mode]
Appliance → Controller:  (Success: no response)
                         OR
                         [ERROR_INVALID_CMD: Invalid parameter]
```

**Command Acknowledgment Philosophy:**

The protocol does NOT use explicit ACK/NACK messages for successful commands. Instead:

- **Configuration commands** (MOTOR_CONFIG, PUMP_CONFIG, etc.): Controllers should maintain the desired configuration state and periodically send commands to reconcile any differences. This provides built-in retry capability without explicit acknowledgments.

- **Control commands** (STATE_COMMAND, MOTOR_COMMAND, etc.): Success is inferred from subsequent telemetry (e.g., STATE_DATA shows expected state, MOTOR_DATA shows expected RPM).

- **Error responses**: Only sent when a command is explicitly rejected. Absence of an error response does NOT guarantee the command was received or applied.

**Rationale:** This approach simplifies implementation, reduces protocol overhead, and provides natural resilience to packet loss through periodic state reconciliation.

### 2. Periodic Telemetry

Appliance broadcasts telemetry at fixed intervals **when enabled by controller:**

```
Controller → Appliance:  TELEMETRY_CONFIG (enable=1, interval_ms=100)

[After enabling, appliance broadcasts at configured interval:]

  Every <interval_ms>:      MOTOR_DATA (per motor) + TEMPERATURE_DATA (per sensor)
  Every <interval_ms×2.5>:  STATE_DATA

Example at 100ms interval:
  MOTOR_DATA + TEMPERATURE_DATA every 100ms
  STATE_DATA every 250ms

Example at 500ms interval (lower bandwidth):
  MOTOR_DATA + TEMPERATURE_DATA every 500ms
  STATE_DATA every 1250ms

For appliances with multiple motors/sensors, separate messages are sent for each.
```

**Important:**
- Telemetry is **disabled by default** on boot
- Controller MUST explicitly enable telemetry with TELEMETRY_CONFIG command
- **No data messages (except PING_RESPONSE and E_STOP broadcasts) SHALL be sent until telemetry is enabled** (see "Emergency Stop Behavior" for E_STOP transmission rules)
- Telemetry auto-disables when communication timeout elapses (see TIMEOUT_CONFIG)
- This prevents boot synchronization issues and reduces unnecessary traffic

### 3. Event-Driven Updates

Appliance sends data messages on state changes **when telemetry is enabled:**

```
PUMP_DATA:  Sent on pump cycle events
GLOW_DATA:  Sent when glow plug turns on/off
```

**Note:** Event-driven messages SHALL only be sent when telemetry is enabled (telemetry_enabled=1). They SHALL be sent when their events occur. In polling mode (interval_ms=0), event-driven messages are NOT sent automatically; controllers must use SEND_TELEMETRY to explicitly request PUMP_DATA and GLOW_DATA.

### 4. Heartbeat

Controller can check connectivity:

```
Controller → Appliance:  PING_REQUEST
Appliance → Controller:  PING_RESPONSE (with uptime)
```

### 5. Timeout Mode (Safety Feature)

Appliances automatically transition to IDLE mode and disable telemetry if communication with controller is lost.

**Operation:**
- Appliance tracks time since last PING_REQUEST received
- If timeout interval exceeded (30 seconds):
  - Appliance automatically enters IDLE mode (state machine safety)
  - Telemetry broadcasts are automatically disabled (communication safety)
- Prevents continued operation without controller supervision
- **Enabled by default** for safety

**Default Timeout:** 30 seconds (configurable via TIMEOUT_CONFIG command, see Configuration Commands)

**Behavior:**
```
Normal operation:
  Controller → Appliance:  PING_REQUEST (every 10-15 seconds)
  Appliance → Controller:  PING_RESPONSE
  Appliance → Controller:  [Telemetry broadcasts continue if enabled]

Timeout condition:
  [Configured timeout elapses with no PING_REQUEST]
  Appliance: Automatically transitions to IDLE mode
  Appliance: Automatically disables telemetry broadcasts
  [No further telemetry until controller re-enables]
```

**Configuration:**
- Timeout mode MUST be **enabled by default**
- Timeout interval default: 30000ms (configurable via TIMEOUT_CONFIG command)
- RECOMMENDED ping interval: 10000-15000ms (well below timeout)

**Safety Rationale:**
- Ensures appliance doesn't operate indefinitely without controller supervision
- Critical for burner systems where loss of communication requires safe shutdown
- IDLE mode performs proper cooldown if temperature is elevated
- Telemetry auto-disable prevents:
  - Continuous transmission when controller is absent (reduces power, prevents bus congestion)
  - Boot synchronization issues when controller reconnects
  - Unnecessary telemetry traffic during disconnected periods
- 30-second timeout allows for:
  - Controller reconnection (unplugging/replugging for relocation)
  - Temporary network disruptions
  - Prevents unnecessary cooldown cycles during brief disconnections
- Controller MUST explicitly re-enable telemetry after reconnection to resume broadcasts

### 6. Controller Routing (Multi-Hop Communication)

Controllers can act as routers to forward commands and data between other controllers and appliances across different physical layers (WiFi/BT/LoRa ↔ LIN/RS-485).

**Use Case:** Remote controller wants to control/monitor an appliance through an intermediate controller that has physical connectivity.

**Architecture:**
```
Remote Controller (WiFi/BT)
    |
    | Fusain over WiFi/BT/TCP
    |
Router Controller (WiFi + LIN)
    |
    | Fusain over LIN
    |
Appliance (LIN)
```

**Command Forwarding:**

Remote controller sends commands to appliance through router:

```
1. Remote Controller → Router Controller (via WiFi):
   ADDRESS: Appliance address (final destination)
   MSG_TYPE: STATE_COMMAND (or any control command)
   PAYLOAD: [command parameters]

2. Router Controller receives packet:
   - Sees ADDRESS ≠ own address
   - Recognizes appliance is on LIN physical layer
   - Forwards packet unchanged to LIN bus

3. Appliance receives command:
   - Sees ADDRESS = own address
   - Processes command normally
```

**Data Subscription and Forwarding:**

Remote controller subscribes to telemetry from appliance through router:

```
1. Remote Controller → Router Controller (via WiFi):
   MSG_TYPE: DATA_SUBSCRIPTION (0x14)
   ADDRESS: Router controller address
   appliance_address: Appliance address

2. Router Controller processes subscription:
   - Adds Remote Controller to routing table for Appliance
   - Notes subscription came via WiFi physical layer

3. Appliance sends telemetry (via LIN):
   MSG_TYPE: MOTOR_DATA, TEMPERATURE_DATA, STATE_DATA (0x30-0x34)
   ADDRESS: Appliance's own address (source)

4. Router Controller receives telemetry:
   - Processes locally if needed
   - Checks routing table: Remote Controller subscribed to Appliance data
   - Forwards copy to Remote Controller via WiFi

5. Remote Controller receives telemetry from Appliance through Router
```

**Routing Table Management:**

Router controllers maintain subscription table:

| Subscriber Connection | Appliance Address | Physical Layer | Last Activity |
|-----------------------|-------------------|----------------|---------------|
| WiFi connection #1 | 0xFEDCBA0987654321 | WiFi | 2026-01-05 14:23:10 |
| BT connection #2 | 0xFEDCBA0987654321 | Bluetooth | 2026-01-05 14:22:45 |

**ADDRESS Field Usage in Routed Packets:**

When a router controller forwards packets between physical layers, the ADDRESS field is NEVER modified:

- **Command Packets (Controller → Appliance):**
  - ADDRESS field contains the final destination appliance address
  - Router inspects ADDRESS field to determine if forwarding is needed
  - If ADDRESS ≠ router's address AND appliance is on a different physical layer:
    - Router forwards the packet to that physical layer WITHOUT modifying ADDRESS field
  - Destination appliance receives packet with its own address (as expected)

- **Data Packets (Appliance → Controller):**
  - ADDRESS field contains the original source appliance address (sender)
  - Router receives packet from appliance (via physical layer or broadcast)
  - Router checks subscription table for subscribers to this appliance
  - Router forwards copy to each subscriber WITHOUT modifying ADDRESS field
  - Subscriber receives packet with original appliance address (source identification)

**Rationale:** Preserving the ADDRESS field allows end devices to identify the true source/destination regardless of routing hops. Routers are transparent forwarders, not packet originators.

**Routing Implementation Requirements:**

1. **Packet Inspection:**
   - Router MUST inspect ADDRESS field of all received packets
   - If ADDRESS matches router's own address → process locally
   - If ADDRESS doesn't match → check if forwarding is needed

2. **Command Forwarding:**
   - When router receives command (0x10-0x2F) with ADDRESS ≠ own address:
   - Forward packet to physical layer where destination device is located
   - Router MAY maintain device location cache (which physical layer each device is on)
   - If device location unknown, MAY broadcast to all physical layers or drop packet

3. **Data Forwarding:**
   - When router receives data message (0x30-0x3F):
   - Check subscription table for matching appliance_address
   - Forward copy of ALL data messages to each subscribed connection

4. **Subscription Timeout:**
   - Router SHOULD remove subscriptions after 60 seconds without PING_RESPONSE from subscriber
   - This prevents stale routing table entries for disconnected controllers

5. **Loop Prevention:**
   - Routers SHOULD NOT forward packets back to the physical layer they were received from
   - Prevents routing loops in multi-router networks

**Example Scenario - Remote Burner Control:**

User wants to control their heater from a phone app:

1. **Setup:**
   - Appliance: Helios burner ICU (LIN bus, address 0xAABBCCDDEEFF0011)
   - Router: Slate controller (WiFi + LIN, address 0x1122334455667788)
   - Remote: Phone app with WiFi controller (address 0x9988776655443322)

2. **Connection:**
   - Phone app connects to Router via WiFi
   - Phone app sends DATA_SUBSCRIPTION to Router for Appliance

3. **Control:**
   - User taps "Start Heating" in app
   - Phone app sends STATE_COMMAND (mode=HEAT) with ADDRESS=Appliance to Router via WiFi
   - Router forwards command to Appliance via LIN
   - Appliance starts heating

4. **Monitoring:**
   - Appliance sends telemetry (MOTOR_DATA, TEMPERATURE_DATA, STATE_DATA) every 500ms to Router via LIN
   - Router checks subscription table, sees Phone app is subscribed
   - Router forwards telemetry to Phone app via WiFi
   - Phone app displays temperature, RPM, state in real-time

5. **Disconnection:**
   - User closes app or moves out of WiFi range
   - Router detects no PING from Phone app for 60 seconds
   - Router removes Phone app subscription
   - Router stops forwarding telemetry (saves WiFi bandwidth)

**Routing Considerations:**

- **Bandwidth:** Router must handle combined traffic from all subscribers and appliances
- **Latency:** Multi-hop adds latency (typically 50-200ms per hop)
- **Security:** Routers SHOULD validate subscriber permissions (implementation-defined)
- **Scalability:** Routers MAY limit max subscriptions (recommend ≥10 concurrent)
- **Physical Layer Bridging:** Router must handle different baud rates, frame sizes between layers

---

## Data Type Encodings

All multi-byte integers MUST use **little-endian** byte order.

All payload structures MUST be explicitly **packed** (no padding between fields). Implementations MUST use compiler-specific attributes (e.g., `__attribute__((packed))` in GCC/Clang, `#pragma pack(1)` in MSVC) to ensure correct wire format.

| Type | Size | Format | Range |
|------|------|--------|-------|
| u8   | 1 byte | Unsigned integer | 0 to 255 |
| i8   | 1 byte | Signed integer | -128 to 127 |
| i32  | 4 bytes | Signed integer (LE) | -2^31 to 2^31-1 |
| u32  | 4 bytes | Unsigned integer (LE) | 0 to 2^32-1 |
| u64  | 8 bytes | Unsigned integer (LE) | 0 to 2^64-1 |
| f64  | 8 bytes | IEEE 754 double (LE) | ±1.7E±308 |

**Float Encoding Example (225.5°C):**
```
IEEE 754 double: 0x406C280000000000
Little-endian:   00 00 00 00 00 28 6C 40
```

**Address Encoding Example (0x123456789ABCDEF0):**
```
Big-endian (standard): 12 34 56 78 9A BC DE F0
Little-endian (wire):  F0 DE BC 9A 78 56 34 12
```

---

## Implementation Requirements

### UART Configuration

- **Baud Rate:**
  - **Production (LIN):** 19200 baud (default, via UART-to-LIN transceiver)
  - **Production (RS-485):** 115200 baud (upgraded heaters, multi-wire/multi-heater installations)
  - **Development (UART):** 115200 baud (prototyping only)
- **Data Bits:** MUST be 8
- **Parity:** MUST be None
- **Stop Bits:** MUST be 1
- **Flow Control:** MUST be None

**Note:** The baud rate at the MCU UART depends on the physical layer. For LIN production deployments, the MCU communicates with the transceiver IC at 19.2 kbaud. The transceiver handles LIN bus timing. For RS-485 and plain UART, the MCU operates at 115.2 kbaud directly.

### Buffer Requirements

**Receive Buffer:** MUST be minimum 256 bytes (2x max packet size for byte stuffing)
**Transmit Buffer:** MUST be minimum 256 bytes (2x max packet size for byte stuffing)

### Inter-Byte Timeout

**All implementations MUST discard partial packets after 100ms of silence (no bytes received).**

This timeout ensures that incomplete packets due to transmission errors, disconnections, or noise do not permanently block the decoder. When the timeout elapses mid-packet, the receiver resets to searching for a new START byte.

### Protocol Behavior

These requirements apply to all implementations unless otherwise noted.

#### Appliance Transmission Requirements

**Appliances MUST NOT transmit data messages (0x30-0x3F) unless:**
- Responding to a PING_REQUEST with PING_RESPONSE, OR
- Telemetry broadcasting has been enabled via TELEMETRY_CONFIG command, OR
- Appliance is in E_STOP state (see "Emergency Stop Behavior (Appliances Only)" below)

**Rationale:** This prevents boot synchronization errors by ensuring the controller is ready to receive data before the appliance begins broadcasting.

**Exception:** PING_RESPONSE (0x3F) MAY be transmitted at any time in response to PING_REQUEST.

#### Byte Synchronization

**All implementations MUST ignore bytes received on the serial line until a valid START byte (0x7E) is observed.**

**Rationale:** This ensures proper frame synchronization and prevents misinterpretation of noise, garbage bytes, or mid-packet data as valid packets.

**Behavior:**
- Discard all bytes until START byte detected
- After START byte, begin packet decoding
- On decode error, reset to searching for START byte
- Continue until valid packet received or error occurs

**Error Recovery:**
- **START byte (0x7E) received mid-packet:**
  - Abandon current packet immediately
  - Treat the new START byte as beginning of a new packet
  - Log error if applicable (indicates previous packet was corrupted or incomplete)
- **LENGTH field exceeds maximum payload (>114):**
  - Immediately reject the packet when invalid LENGTH byte is received
  - Reset receive buffer
  - Discard all bytes until next START byte detected
  - Log error if applicable
- **Packet exceeds maximum length (128 bytes):**
  - Reset receive buffer immediately
  - Discard all bytes until next START byte detected
  - Log error if applicable
- **END byte (0x7F) received before expected:**
  - Packet incomplete or corrupted
  - Reset receive buffer immediately
  - Discard all bytes until next START byte detected
  - Log error if applicable

#### Emergency Stop Behavior (Controllers Only)

**When a controller transmits STATE_COMMAND with mode=EMERGENCY to an appliance, it MUST:**
- Retransmit the EMERGENCY command every 250ms
- Continue retransmitting until STATE_DATA received with state = E_STOP (0x08)
- Once emergency stop confirmed, stop retransmitting command

**Rationale:** This ensures emergency stop is reliably entered even if the initial command is lost or corrupted, providing safety-critical reliability for emergency stop activation.

**Example - Controller-Initiated Emergency Stop:**
```
Controller sends: STATE_COMMAND (mode=EMERGENCY)
Controller starts: Retransmitting EMERGENCY every 250ms
Appliance receives: STATE_COMMAND (mode=EMERGENCY)
Appliance enters: E_STOP state
Appliance begins: Broadcasting all telemetry (STATE_DATA, MOTOR_DATA, TEMPERATURE_DATA, PUMP_DATA, GLOW_DATA) every 250ms
Controller receives: STATE_DATA (state=E_STOP, error=1, code=...)
Controller stops: Retransmitting EMERGENCY command
Controller continues: Receiving telemetry every 250ms
... continues until power cycle ...
```

#### Emergency Stop Behavior (Appliances Only)

**When an appliance enters an emergency stop state, it MUST:**
- Ignore ALL received commands, including PING_REQUEST and SEND_TELEMETRY
- Transmit STATE_DATA, MOTOR_DATA, TEMPERATURE_DATA, PUMP_DATA, and GLOW_DATA every 250ms
- Continue emergency stop broadcasts until power cycle or hardware reset

**Rationale:** Emergency stop is a safety-critical state that requires immediate visibility to the controller and prevents any command processing that could interfere with safe shutdown.

**Transmission During Emergency Stop:**
- MUST transmit STATE_DATA with state = E_STOP (0x08) and appropriate error code
- MUST transmit MOTOR_DATA, TEMPERATURE_DATA, PUMP_DATA, and GLOW_DATA for diagnostics
- Broadcast interval: 250ms (fixed, not configurable)
- Broadcasts occur regardless of TELEMETRY_CONFIG state
- For multi-device appliances: all messages (one per motor, one per sensor, etc.) MUST be sent as a burst at the start of each 250ms interval

**Recovery:**
- Emergency stop state can ONLY be cleared by:
  - Power cycle (complete power loss and restoration)
  - Hardware reset (physical reset button or watchdog)
- Software commands MUST NOT clear emergency stop state

**Example - Appliance-Initiated Emergency Stop:**
```
Appliance detects fault temperature (>275°C)
Appliance enters: E_STOP state
Appliance begins: Broadcasting all telemetry (STATE_DATA, MOTOR_DATA, TEMPERATURE_DATA, PUMP_DATA, GLOW_DATA) every 250ms
Controller sends: PING_REQUEST (ignored by appliance)
Controller sends: STATE_COMMAND(IDLE) (ignored by appliance)
Controller receives: STATE_DATA (state=E_STOP, error=1, code=OVERHEAT) every 250ms
Controller receives: TEMPERATURE_DATA (temperature=280°C, ...) every 250ms
Controller receives: MOTOR_DATA, PUMP_DATA, GLOW_DATA every 250ms
... continues until power cycle ...
```

#### Broadcast Retry Requirements (Controllers Only)

**If a controller enables broadcast mode on an appliance via TELEMETRY_CONFIG, the controller MUST retransmit the broadcast enable command every time a PING_RESPONSE is received from that appliance if the controller has NOT received a corresponding broadcast data message.**

**Rationale:** This provides automatic recovery if:
- The initial enable command is lost or corrupted
- The appliance resets and disables broadcasting
- Communication is interrupted and broadcasting stops

**Behavior:**
- Track which broadcasts have been enabled
- Track which broadcast data has been received
- On each PING_RESPONSE:
  - If broadcast enabled but no data received → retransmit enable command
  - If broadcast data received → no action needed (working correctly)
- Continue retrying indefinitely until broadcast data is confirmed
- There is NO retry limit - if an appliance only responds to pings but never sends telemetry, it should be considered inoperative

**Example:**
```
Controller sends: TELEMETRY_CONFIG (enabled=true, interval=100ms)
Controller receives: PING_RESPONSE (uptime=5000ms)
Controller checks: Have we received telemetry data? NO
Controller action: Retransmit TELEMETRY_CONFIG
Controller receives: PING_RESPONSE (uptime=15000ms)
Controller checks: Have we received telemetry data? NO
Controller action: Retransmit TELEMETRY_CONFIG
Controller receives: MOTOR_DATA (rpm=2500, ...)
Controller checks: Have we received telemetry data? YES
Controller action: Stop retrying, broadcasts working
```

---

## Error Handling

### Transmit Errors
- **Buffer Full:** Drop oldest packet or block until space available
- **UART Error:** Log error, attempt retransmit once

### Receive Errors
- **CRC Failure:** Discard packet silently, resync to next START byte
- **Framing Error:** Discard packet silently, resync to next START byte
- **Timeout:** Discard partial packet after 100ms silence
- **Invalid Command:** Send ERROR_INVALID_CMD with error code

### Recovery
- On 3 consecutive CRC failures, suggest baud rate mismatch
- On persistent framing errors, suggest physical connection check
- Important commands should be retransmitted if no acknowledgment is received

---

## Performance Characteristics

### Throughput

**At Default 100ms Telemetry Period (1 motor, 1 temperature):**
- MOTOR_DATA (32 bytes) + TEMPERATURE_DATA (32 bytes) + STATE_DATA (16 bytes at 2.5× rate)
- ~70 bytes average per interval = 700 bytes/sec

**At Default 100ms Telemetry Period (3 motors, 3 temperatures):**
- 3× MOTOR_DATA + 3× TEMPERATURE_DATA + STATE_DATA
- ~200 bytes average per interval = 2000 bytes/sec

**At 500ms Telemetry Period (Lower Bandwidth):**
- 1 motor + 1 temperature: ~140 bytes/sec
- 3 motors + 3 temperatures: ~400 bytes/sec

**At 115200 baud:**
- Effective throughput: ~11,520 bytes/sec
- Telemetry overhead: ~1-17% bandwidth utilization (depending on interval and device count)

### Latency

- **Command Processing:** < 5ms (zbus publish + state machine cycle)
- **Telemetry Delay:** 0 to configured interval (depends on timing within broadcast cycle)
  - At 100ms interval: 0-100ms latency
  - At 500ms interval: 0-500ms latency

### Reliability

- **CRC-16:** Detects all single-bit and double-bit errors
- **Byte Stuffing:** Prevents false START/END detection
- **Framing:** Robust resynchronization on errors
- **Timeout Mode:** Automatic IDLE transition on communication loss (enabled by default)
  - Default timeout: 30 seconds
  - Recommended controller ping interval: 10-15 seconds
  - Ensures safe shutdown if controller connection is lost

---

## Physical Layer Compatibility

The Fusain protocol uses UART-based packet framing and can work with various physical layer transceivers. The protocol itself remains unchanged across all physical layers - transceivers handle the adaptation between Fusain packets (on UART/SPI/I2C) and the physical bus.

**Default Configuration:**
- **Production:** LIN physical layer via UART-to-LIN transceiver IC
- **Development:** Plain UART or RS-485 for prototyping and testing

**Transceiver Architecture:**
```
MCU (Fusain firmware)
    |
    | UART/SPI/I2C (Fusain packets)
    |
[Transceiver IC]
    |
    | LIN/RS-485/UART (physical layer)
    |
Physical Bus
```

The transceiver IC handles:
- Physical layer protocol (LIN break/sync, RS-485 DE/RE control, etc.)
- Frame fragmentation/reassembly if needed (for LIN's 8-byte limit)
- Fusain packet buffering and delivery
- The MCU firmware sees only Fusain packets regardless of physical layer

### RS-485 Compatibility

**Status:** ✅ **COMPATIBLE** - Production (Less Common) and Development

**Use Case:** Production deployments where multi-wire connections are available, and development/testing environments.

RS-485 provides robust, multi-drop communication suitable for installations with existing multi-wire infrastructure or where LIN's distance/node limitations are insufficient (up to 1200m, 32-256 nodes).

**Requirements:**

1. **Transmit Enable Control (DE/RE pins):**
   - Firmware MUST control DE (Driver Enable) and RE (Receiver Enable) pins
   - Assert DE before transmission begins
   - De-assert DE after transmission completes (wait for UART TX shift register empty)
   - RE typically tied inverse to DE (DE high = transmit, DE low = receive)

2. **Inter-Packet Gap:**
   - RECOMMENDED: 1-2ms gap between packets to allow DE/RE switching
   - Gives all devices time to switch from transmit to receive mode
   - Controllers SHOULD wait this gap before expecting responses

3. **Discovery Response Collision Avoidance:**
   - DISCOVERY_REQUEST causes all appliances to respond simultaneously
   - On RS-485, this causes bus contention and garbled responses
   - **Solution:** Appliances MUST add random delay (0-50ms) before responding to DISCOVERY_REQUEST
   - Controllers MUST wait sufficient time (100-200ms) to receive all responses
   - Alternative: Use addressed ping instead of broadcast discovery

4. **Turn-Around Time:**
   - RS-485 transceivers have turn-around delay (driver disable to receiver enable)
   - Typically 1-10μs depending on IC
   - Firmware timing must account for this in DE/RE control

**Benefits of RS-485:**
- Longer distances (up to 1200m vs ~15m for plain UART)
- Better noise immunity (differential signaling)
- Multi-drop support (up to 32-256 nodes depending on IC)
- Higher reliability in industrial environments

**Considerations:**
- Requires additional GPIO pin for DE/RE control
- Firmware complexity increased for transmit enable timing
- Discovery requires collision avoidance strategy
- Bus termination resistors (120Ω) required at both ends

#### Multi-Drop RS-485 Topology

RS-485's multi-drop capability allows multiple devices to share a single bus, making it ideal for networks with one controller and multiple appliances.

**Bus Topology Options:**

1. **Daisy-Chain (Recommended):**
   ```
   [Controller] --[120Ω]-- [Appliance 1] -- [Appliance 2] -- ... -- [Appliance N] --[120Ω]-- Ground
        A/B                      A/B              A/B                    A/B
   ```
   - Devices connected in series along the main bus
   - Short stubs (< 30cm) from bus to each device
   - Termination resistors (120Ω) at both physical ends only
   - **Best:** Minimizes reflections, maximum distance
   - **Limitation:** Physical cable routing must follow chain

2. **Star Topology (NOT Recommended):**
   ```
   [Controller]
        |
        +-- [Hub] -- [Appliance 1]
              |
              +-- [Appliance 2]
              |
              +-- [Appliance 3]
   ```
   - Multiple branches from central point
   - **Problem:** Creates impedance mismatches and reflections
   - **Problem:** Difficult to terminate properly (multiple endpoints)
   - **Result:** Unreliable communication, signal integrity issues
   - **Avoid:** Use daisy-chain instead

3. **Tree/Mixed Topology (Problematic):**
   - Combination of daisy-chain with branches
   - **Problem:** Each branch creates reflections at branch point
   - **Acceptable:** Very short stubs (< 30cm) from main bus to devices
   - **Not Acceptable:** Long branches from main bus
   - **Rule:** Keep branch stubs < 1/10 of signal wavelength (~30cm at 115200 baud)

**Node Capacity:**

RS-485 node limits depend on transceiver "unit load" (UL):

| Transceiver Type | Unit Load | Max Nodes on Bus | Example ICs |
|------------------|-----------|------------------|-------------|
| Standard | 1 UL | 32 nodes | MAX485, SN75176 |
| 1/2 Unit Load | 0.5 UL | 64 nodes | MAX3485 |
| 1/4 Unit Load | 0.25 UL | 128 nodes | MAX13487E |
| 1/8 Unit Load | 0.125 UL | 256 nodes | MAX14850 |

- **Unit Load:** Input capacitance presented to the bus
- **Total Load:** Sum of all devices must not exceed 32 UL
- **Typical Fusain Network:** 1 controller + 1-10 appliances = standard transceivers sufficient
- **Large Networks:** Use 1/8 UL transceivers for >32 devices

**Bus Termination:**

Termination resistors prevent signal reflections that cause data corruption.

**Requirements:**
- **Value:** 120Ω, 1/4W or greater
- **Location:** Both physical ends of the bus (not at every device)
- **Topology:** Must match characteristic impedance of twisted pair cable
- **Failure Mode:** Missing termination → reflections → CRC errors, communication failures

**Proper Termination Example (3 appliances):**
```
[120Ω]                                                              [120Ω]
   |                                                                   |
[Controller] -------- [Appliance 1] -------- [Appliance 2] -------- [Appliance 3]
 (End #1)                                                              (End #2)
```

**Incorrect Termination Examples:**
```
❌ Termination at every device:
[120Ω]           [120Ω]           [120Ω]           [120Ω]
   |                |                |                |
[Controller] -- [Appliance 1] -- [Appliance 2] -- [Appliance 3]
Problem: Bus impedance = 120Ω || 120Ω || 120Ω = 30Ω (severe mismatch)

❌ No termination:
[Controller] -------- [Appliance 1] -------- [Appliance 2] -------- [Appliance 3]
Problem: Signal reflections cause data corruption

❌ Only one end terminated:
[120Ω]
   |
[Controller] -------- [Appliance 1] -------- [Appliance 2] -------- [Appliance 3]
Problem: Reflections from unterminated far end
```

**Cable Specifications:**

1. **Cable Type:**
   - **Required:** Twisted pair (differential signaling requires matching pair)
   - **Characteristic Impedance:** 120Ω (matches termination resistors)
   - **Recommended:** Industrial RS-485 cable, CAT5/CAT6 Ethernet cable (one pair)
   - **Avoid:** Parallel untwisted wires (poor noise immunity, wrong impedance)

2. **Maximum Cable Length:**
   - 1200m at 115200 baud (ideal conditions, low capacitance cable)
   - 100m guaranteed for most CAT5 installations
   - Longer distances: Reduce baud rate or use repeaters

3. **Shielding:**
   - **Shielded (STP):** Recommended for industrial/noisy environments
   - **Unshielded (UTP):** Acceptable for benign environments (office, residential)
   - **Shield Grounding:** Ground shield at one end only to prevent ground loops

**Grounding and Biasing:**

1. **Common-Mode Ground:**
   - RS-485 is differential (voltage between A and B matters)
   - Common-mode voltage (average of A and B) must stay within ±7V of device ground
   - **Recommendation:** Connect all device grounds together
   - **Alternative:** Use isolated RS-485 transceivers (ground isolation up to 2.5kV)

2. **Fail-Safe Biasing:**
   - Idle bus (no drivers active) is undefined state
   - Can cause receivers to see random noise as data
   - **Solution:** Add bias resistors to pull bus to known idle state:
     - Pull-up resistor (560Ω) from A to +5V
     - Pull-down resistor (560Ω) from B to ground
   - **Modern ICs:** Many have built-in fail-safe biasing (check datasheet)

**Collision Avoidance for Fusain Protocol:**

Since Fusain is not inherently collision-safe, RS-485 multi-drop requires software strategies:

1. **Discovery Phase (DISCOVERY_REQUEST):**
   - Problem: All appliances respond simultaneously → bus collision
   - Solution: Each appliance adds random delay (0-50ms) before responding
   - Controller must wait 100-200ms to receive all DEVICE_ANNOUNCE responses

2. **Normal Operation:**
   - Controller uses addressing to send commands to specific appliances
   - Each appliance only responds when addressed (no collisions)
   - Polling mode (TELEMETRY_CONFIG with interval_ms=0) is REQUIRED for multi-appliance networks to avoid broadcast collisions

3. **Alternative: Addressed Polling:**
   - Avoid broadcast discovery entirely
   - Controller pings each known address sequentially
   - No collisions, but slower discovery
   - Better for large networks (>10 devices)

**Practical Implementation Notes:**

1. **Stub Lengths:**
   - Keep device connection stubs < 30cm from main bus
   - Longer stubs create reflections (signal integrity degradation)
   - At 115200 baud: wavelength ≈ 2.6m, so stubs must be << 260cm

2. **Cable Routing:**
   - Run cable in straight line from controller to farthest appliance
   - Avoid loops, branches, or stars
   - Mark physical cable ends for termination placement

3. **Common Issues:**
   - **CRC errors:** Check termination, cable type, stub lengths
   - **Intermittent communication:** Check grounding, cable shield
   - **No communication:** Check A/B polarity (swap if needed), verify baud rate
   - **Collisions during discovery:** Increase random delay range, use addressed polling

4. **Testing:**
   - Start with 2 devices (controller + 1 appliance), verify communication
   - Add devices one at a time, testing after each addition
   - Use oscilloscope to verify differential signal quality
   - Check for reflections, ringing, or excessive overshoot

**Recommended Hardware:**

- **Transceivers:** MAX485 (basic), MAX3485 (improved), ADM2682E (isolated)
- **Cable:** Belden 3105A (industrial RS-485), CAT5e (budget option, use one pair)
- **Connectors:** RJ45 (Ethernet), Phoenix Contact terminal blocks (industrial)
- **Termination:** 120Ω 1/4W carbon film resistors (not ceramic, not wirewound)

**Example Network Configurations:**

**Small System (1 controller + 3 appliances, <100m):**
- Cable: CAT5e UTP, use one twisted pair for A/B
- Transceivers: MAX485 (standard, cheap)
- Topology: Daisy-chain
- Termination: 120Ω at controller and last appliance
- Grounding: Connect all grounds together (no isolation needed)

**Industrial System (1 controller + 20 appliances, 500m):**
- Cable: Belden 3105A shielded twisted pair
- Transceivers: ADM2682E (isolated, 1/4 unit load)
- Topology: Daisy-chain along production line
- Termination: 120Ω at both physical ends
- Grounding: Isolated transceivers, shield grounded at controller only
- Fail-safe: 560Ω bias resistors at controller (A to +5V, B to GND)

### LIN Compatibility

**Status:** ✅ **COMPATIBLE** - Default Production Physical Layer

**Restriction:** LIN MUST NOT be used for multi-appliance networks. LIN's single-master architecture and limited collision handling make it unsuitable for multi-drop configurations. Use RS-485 for multi-appliance networks.

**Use Case:** Production deployments for retrofitting heaters with single-wire communication to controller (single appliance per bus).

LIN (Local Interconnect Network) is the default physical layer for Fusain in production environments. Most heaters being retrofitted only have one wire available for appliance-to-controller communication, making LIN's single-wire topology ideal.

**Architecture:**

The Fusain protocol runs at the MCU level unchanged. A UART-to-LIN transceiver IC handles all LIN physical layer requirements:

```
Controller MCU                          Appliance MCU
    |                                        |
    | UART (Fusain packets)                  | UART (Fusain packets)
    |                                        |
[UART-to-LIN IC]                        [UART-to-LIN IC]
    |                                        |
    +--------------- LIN Bus ----------------+
           (single wire + ground)
```

**Transceiver IC Responsibilities:**
- **Fragmentation/Reassembly:** Splits Fusain packets (up to 128 bytes) into LIN frames (8 bytes max)
- **LIN Physical Layer:** Handles break field, sync byte, frame IDs, checksums
- **Buffering:** Queues Fusain packets for transmission, reassembles received frames
- **Transparent Operation:** MCU firmware sends/receives complete Fusain packets via UART/SPI/I2C

**Recommended Transceiver ICs:**

| IC | Interface | Features | Use Case |
|----|-----------|----------|----------|
| **TJA1027** | UART | Transparent UART-to-LIN, low standby current, automotive-grade | **Recommended for production** |
| **TLIN1029** | UART | UART-to-LIN with diagnostic features, integrated regulator | Industrial applications |
| **MCP2003B** | UART | Low-cost UART-to-LIN, basic functionality | Budget builds |
| **NCV7321** | SPI | SPI-controlled LIN transceiver, flexible | Complex systems |

**TJA1027 Example (Recommended):**
- UART interface to MCU (19.2 kbaud typical)
- Automatically handles LIN break/sync generation and detection
- Transparent packet passthrough (IC handles fragmentation internally)
- Automotive-grade reliability (AEC-Q100 qualified)
- Low standby current (ideal for battery-powered applications)
- Sleep mode support with wake-up capability
- No external components required (internal termination, biasing)

**Configuration:**

1. **Baud Rate:**
   - **MCU ↔ Transceiver:** 19.2 kbaud (standard LIN rate)
   - **LIN Bus:** 19.2 kbaud (transceiver handles timing)
   - Fusain protocol timing adjusted for lower throughput

2. **Telemetry Intervals:**
   - Recommended: 500ms minimum (allows for fragmentation overhead)
   - Maximum packet: ~128 bytes → ~16 LIN frames → ~80ms transmission time
   - 500ms interval provides 5× margin for fragmentation and bus scheduling

3. **Event-Driven Messages:**
   - PUMP_DATA and GLOW_DATA still supported
   - Transceiver queues events for transmission
   - May experience higher latency due to fragmentation

4. **Discovery:**
   - DISCOVERY_REQUEST supported (uses LIN broadcast frame ID)
   - DEVICE_ANNOUNCE responses queued by transceivers
   - Random delay (0-50ms) MUST still be applied for protocol consistency across physical layers

**MCU Integration:**

The MCU firmware is identical to UART/RS-485 configurations. The transceiver handles all LIN-specific details. No firmware changes required - the Fusain library works unchanged regardless of physical layer.

**Benefits of LIN for Production:**
- **Single-wire topology:** Ideal for retrofits with limited wiring
- **Automotive-grade reliability:** Proven in harsh environments
- **Built-in collision avoidance:** LIN centralized scheduling prevents bus contention
- **Lower cost:** Cheaper than CAN bus, simpler than RS-485
- **Integrated features:** Many transceivers include voltage regulation, diagnostics
- **Standard compliance:** LIN 2.0+ specification ensures interoperability

**Performance Characteristics:**

| Metric | Value | Notes |
|--------|-------|-------|
| **Baud Rate** | 19.2 kbaud | LIN standard, sufficient for Fusain |
| **Max Distance** | ~40m | Shorter than RS-485, acceptable for heater installations |
| **Max Nodes** | 2 (Fusain) | LIN supports 16 nodes, but Fusain restricts to 1 controller + 1 appliance |
| **Packet Latency** | 80-150ms | Depends on fragmentation and bus scheduling |
| **Telemetry Interval** | 500ms recommended | Allows for fragmentation overhead |
| **Bandwidth Utilization** | ~15-20% | Telemetry + commands at 500ms interval |

**Wiring:**

- **LIN Bus:** Single wire (12V nominal, automotive-grade wire)
- **Ground:** Separate ground return (vehicle chassis or dedicated wire)
- **Termination:** Built into most transceiver ICs (no external resistors needed)
- **Connectors:** Automotive-grade connectors (Deutsch, AMP, or equivalent)

**Limitations (Handled by Transceiver):**
- Lower bandwidth than UART/RS-485 (handled by increasing telemetry intervals)
- Fragmentation latency (acceptable for heater control - not time-critical)
- Distance limitation (40m sufficient for most heater installations)

### Comparison Summary

| Feature | Plain UART | RS-485 | LIN |
|---------|-----------|--------|-----|
| **Distance** | ~15m | ~1200m | ~40m |
| **Baud Rate** | 115200 | 115200 | 19200 |
| **Multi-drop** | No | Yes (32-256 nodes) | No (Fusain restriction) |
| **Wiring** | 2-wire | 2-wire (differential) | 1-wire + ground |
| **Noise Immunity** | Low | High | Medium-High |
| **Collision Handling** | None | None (SW required) | Centralized |
| **Implementation** | Simple | Moderate (DE/RE + SW) | Simple (transceiver IC) |
| **Fusain Compatibility** | ✅ Native | ✅ With firmware mods | ✅ Via transceiver IC |
| **Recommended Use** | Dev/testing only | Production (less common) | **Production (default)** |
| **Typical Use Case** | Lab bench testing | Multi-wire installations | **Single-wire retrofits** |

### Recommendations

**For Production Deployments:**

1. **Single-wire retrofits (MOST COMMON):** LIN with UART-to-LIN transceiver IC
   - Use TJA1027 or equivalent transceiver
   - Configure 19.2 kbaud at MCU UART
   - Set telemetry interval to 500ms minimum
   - Ideal for heater retrofits with existing single-wire infrastructure

2. **Multi-appliance networks, multi-wire installations, or long distances (>40m):** RS-485
   - Implement DE/RE control in firmware or use auto-direction transceivers
   - Add random delays (0-50ms) to DISCOVERY_REQUEST responses
   - Use daisy-chain topology with proper termination (120Ω at both ends)
   - Consider for industrial environments with harsh EMI

3. **Development and prototyping:** Plain UART
   - Simplest setup for initial development
   - Direct UART connection between devices
   - Use for bench testing, firmware development, protocol validation

**Protocol Configuration:**
- **Production (LIN):** 19.2 kbaud, 500ms telemetry interval
- **Production (RS-485):** 115200 baud, 100-500ms telemetry interval
- **Development (UART):** 115200 baud, 100ms telemetry interval
- All configurations use identical packet format (framing, CRC, addressing)

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-06 | Thermoquad Team | Initial Fusain Protocol specification |

---

## References

- **RFC 2119:** Key words for use in RFCs to Indicate Requirement Levels - https://www.rfc-editor.org/rfc/rfc2119.txt
- **CRC-16-CCITT:** ITU-T Recommendation V.41
- **IEEE 754:** IEEE Standard for Floating-Point Arithmetic
- **LIN Specification:** LIN Consortium, LIN 2.0 Protocol Specification
