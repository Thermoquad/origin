# Fusain Protocol Specification

## Overview

The Fusain protocol is a binary packet-based protocol for communicating with Helios-compatible appliances (burners, ignition control units) over UART or other serial transports. The protocol provides comprehensive configuration, command/control capabilities, and real-time telemetry.

**Fusain** (fossilized charcoal) - A platform-independent communication protocol for thermal appliance control.

### RFC 2119 Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

**Transport Layer:**
- UART with optional physical layer translation (RS-485, LIN)
- Default Baud Rate: 115200 (adjustable for LIN compatibility)
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
- **Typical appliance:** 1 motor, 1 thermometer, 1 pump, 1 glow plug (e.g., Helios ICU)

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

**Broadcast Address Behavior (0x0000000000000000):**

When a device receives a packet with ADDRESS = 0x0000000000000000:

1. **Appliances:**
   - MUST process the command if it's a valid command message (0x10-0x2F)
   - MUST NOT send responses to broadcast commands (prevents bus collisions)
   - **Exception:** DISCOVERY_REQUEST (0x1F) triggers DEVICE_ANNOUNCE response
   - **Exception response behavior:**
     - RS-485 multi-drop: MUST add random delay (0-50ms) before responding to avoid collisions
     - Plain UART/LIN: MAY respond immediately (controller expects potential collisions)
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
| DISCOVERY_REQUEST (0x1F) | **YES** | Own address | Add random delay on RS-485 |
| MOTOR_CONFIG (0x10-0x13) | No response | - | Config silently applied |
| STATE_COMMAND (0x20) | No response | - | All appliances execute |
| PING_REQUEST (0x2F) | No response | - | Would cause bus collision |
| All other commands | No response | - | Processed but no response |

**Collision Avoidance for DISCOVERY_REQUEST:**

DISCOVERY_REQUEST is the ONLY broadcast command that expects responses. To prevent bus collisions:

- **RS-485/multi-drop networks:**
  - Each appliance MUST wait a random delay (0-50ms) before sending DEVICE_ANNOUNCE
  - Controller MUST wait at least 100ms (preferably 200ms) to receive all responses
  - Controller MAY receive partial/corrupted responses due to collisions
  - Controller SHOULD retry discovery if expected devices don't respond

- **Point-to-point networks (plain UART):**
  - No collision possible (only one appliance)
  - Appliance MAY respond immediately without delay

- **LIN networks:**
  - LIN master-slave architecture prevents collisions
  - Transceiver IC handles response scheduling
  - No random delay needed

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

To prevent confusion with START (0x7E) and END (0x7F) delimiters appearing in the payload, address, or CRC:

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
| 0x12 | TEMP_CONFIG | Configure temperature controller parameters | 48 bytes |
| 0x13 | GLOW_CONFIG | Configure glow plug parameters | 16 bytes |
| 0x14 | DATA_SUBSCRIPTION | Subscribe to data from appliance (for routing) | 16 bytes |
| 0x15 | DATA_UNSUBSCRIPTION | Unsubscribe from appliance data | 8 bytes |
| 0x16 | TELEMETRY_CONFIG | Enable/disable telemetry broadcasts | 12 bytes |
| 0x17-0x1E | *Reserved* | Reserved for future configuration commands | - |
| 0x1F | DISCOVERY_REQUEST | Request device capabilities | 0 bytes |

### Control Commands (Controller → Appliance)

Control commands provide real-time operational control without changing persistent configuration.

| MSG_TYPE | Name | Description | Payload Size |
|----------|------|-------------|--------------|
| 0x20 | STATE_COMMAND | Set system mode/state | 8 bytes |
| 0x21 | MOTOR_COMMAND | Set motor RPM | 8 bytes |
| 0x22 | PUMP_COMMAND | Set pump rate | 8 bytes |
| 0x23 | GLOW_COMMAND | Control glow plug | 8 bytes |
| 0x24 | TEMP_COMMAND | Temperature controller control | 20 bytes |
| 0x25-0x2E | *Reserved* | Reserved for future control commands | - |
| 0x2F | PING_REQUEST | Heartbeat/connectivity check | 0 bytes |

### Telemetry Data (Appliance → Controller)

Telemetry messages provide real-time status and sensor data from appliances.

| MSG_TYPE | Name | Description | Payload Size | Send Rate |
|----------|------|-------------|--------------|-----------|
| 0x30 | STATE_DATA | System state and status | 16 bytes | 2.5× telemetry interval |
| 0x31 | MOTOR_DATA | Motor telemetry | 32 bytes | Per telemetry interval |
| 0x32 | PUMP_DATA | Pump status and events | 16 bytes | On event |
| 0x33 | GLOW_DATA | Glow plug status | 12 bytes | On event |
| 0x34 | TEMP_DATA | Temperature readings | 32 bytes | Per telemetry interval |
| 0x35 | TELEMETRY_BUNDLE | Consolidated telemetry | Variable: 42-108 bytes | Per telemetry interval |
| 0x36 | DEVICE_ANNOUNCE | Device capabilities announcement | 8 bytes | On DISCOVERY_REQUEST |
| 0x37-0x3E | *Reserved* | Reserved for future telemetry messages | - | - |
| 0x3F | PING_RESPONSE | Heartbeat response | 4 bytes | On request |

### Error Messages (Bidirectional)

Error messages indicate protocol or command validation failures.

| MSG_TYPE | Name | Description | Payload Size |
|----------|------|-------------|--------------|
| 0xE0 | ERROR_INVALID_MSG | Invalid message received | 4 bytes |
| 0xE1 | ERROR_CRC_FAIL | CRC validation failed | 4 bytes |
| 0xE2 | ERROR_INVALID_CMD | Command validation failed | 4 bytes |
| 0xE3 | ERROR_STATE_REJECT | Command rejected by state machine | 4 bytes |

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
- **motor** (i32): Motor index (0-9, typically 0)
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
- pwm_period MUST be > 0
- max_rpm MUST be > min_rpm
- min_pwm_duty MUST be < pwm_period
- PID gains MAY be 0 (disables that term)

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
- **pump** (i32): Pump index (0-9, typically 0)
- **pulse_ms** (u32): Solenoid pulse duration in milliseconds (e.g., 50)
- **recovery_ms** (u32): Recovery time after pulse in milliseconds (e.g., 50)
- **padding** (4 bytes): Reserved for future use

**Default Values (Helios):**
- pulse_ms: 50
- recovery_ms: 50

**Validation:**
- pulse_ms MUST be > 0
- recovery_ms MUST be > 0
- Minimum pump rate = pulse_ms + recovery_ms (typically 100ms minimum)

**Rationale:**
- Pulse duration controls fuel delivery per cycle
- Recovery time prevents solenoid overheating and ensures complete valve closure

### 0x12 - TEMP_CONFIG

Configure temperature controller parameters including PID gains and sampling.

**Payload Structure (48 bytes):**
```
+-------------+--------+--------+--------+--------------+-----------+
| thermometer | pid_kp | pid_ki | pid_kd | sample_count | read_rate |
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
- **thermometer** (i32): Temperature controller index (0-9, typically 0)
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
- sample_count MUST be > 0
- read_rate MUST be > 0
- PID gains MAY be 0 (disables that term)

**Rationale:**
- Moving average filter reduces sensor noise
- Warmup time = sample_count × read_rate (60 × 50ms = 3 seconds)
- PID gains tuned for inverted control (higher temp → higher RPM for cooling)

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
- **glow** (i32): Glow plug index (0-9, typically 0)
- **max_duration** (u32): Maximum allowed glow duration in milliseconds (e.g., 300000 = 5 minutes)
- **padding** (8 bytes): Reserved for future use

**Default Values (Helios):**
- max_duration: 300000 ms (5 minutes)

**Validation:**
- max_duration MUST be > 0
- RECOMMENDED: max_duration ≤ 300000 ms (5 minutes) for safety

**Rationale:**
- Prevents indefinite glow plug operation
- Safety timeout for preheat phase

### 0x14 - DATA_SUBSCRIPTION

**Direction:** Controller → Controller (for routing scenarios)

**Note:** This is NOT a Controller → Appliance command. This command is sent from one controller (subscriber) to another controller (router) to establish data forwarding.

Subscribe to receive copies of data messages from a specific appliance. Used for controller routing scenarios where one controller (router) is physically connected to an appliance, and another controller (subscriber) wants to receive telemetry data through the router.

**Payload Structure (16 bytes):**
```
+-------------------+----------------+
| appliance_address | message_filter |
+-------------------+----------------+
| u64               | u32            |
+-------------------+----------------+

+----------+
| reserved |
+----------+
| u32      |
+----------+
```

**Fields:**
- **appliance_address** (u64): Address of the appliance to subscribe to
  - MUST be a valid appliance address (not a controller address)
  - MUST NOT be the broadcast address (0x0000000000000000)
- **message_filter** (u32): Bitmask specifying which data message types to forward
  - Each bit represents a data message type (0x30-0x3F)
  - Bit 0 = STATE_DATA (0x30)
  - Bit 1 = MOTOR_DATA (0x31)
  - Bit 2 = PUMP_DATA (0x32)
  - Bit 3 = GLOW_DATA (0x33)
  - Bit 4 = TEMP_DATA (0x34)
  - Bit 5 = TELEMETRY_BUNDLE (0x35)
  - Bit 6 = DEVICE_ANNOUNCE (0x36)
  - Bit 15 = PING_RESPONSE (0x3F)
  - 0xFFFFFFFF = Subscribe to all data messages (common case)
- **reserved** (u32): Reserved for future use, MUST be 0

**Behavior:**

When a controller (router) receives this command:
1. Extracts subscriber address from the packet's ADDRESS field (sender of this command)
2. Adds subscriber to routing table for the specified appliance_address
3. When data messages are received from appliance_address, checks message type against message_filter
4. If message type matches filter, forwards a copy to the subscriber over the physical layer where the subscription was received

**Multi-Hop Routing Example:**

Controller A (WiFi/BT) wants telemetry from Appliance C (LIN bus) through Controller B (WiFi+LIN bridge):

1. Controller A sends DATA_SUBSCRIPTION to Controller B:
   - ADDRESS: Controller B's address
   - appliance_address: Appliance C's address
   - message_filter: 0xFFFFFFFF (all data)

2. Controller B receives subscription, adds Controller A to routing table for Appliance C

3. When Appliance C sends telemetry (e.g., TELEMETRY_BUNDLE):
   - Packet ADDRESS field: Controller B's address (or broadcast)
   - Controller B receives packet, processes locally if needed
   - Controller B checks routing table, sees Controller A is subscribed
   - Controller B forwards packet to Controller A over WiFi/BT

4. Controller A receives telemetry from Appliance C through Controller B

**Validation:**

When a router controller receives this command, it MUST validate:

1. **appliance_address Validity:**
   - MUST NOT be the broadcast address (0x0000000000000000)
   - If appliance_address doesn't exist on any connected physical layer:
     - Router MAY ignore the subscription request silently
     - Router MAY return ERROR_INVALID_CMD (if error responses are implemented)
     - Router SHOULD NOT store subscriptions for non-existent appliances
   - Router MAY defer validation until first data message from that appliance

2. **Subscription Table Capacity:**
   - Routers MAY limit the number of active subscriptions (implementation-defined)
   - RECOMMENDED minimum: 10 concurrent subscriptions per router
   - When subscription table is full:
     - Router MUST NOT accept new subscriptions
     - Router SHOULD ignore the subscription request silently
     - Router MAY evict oldest/least-recently-active subscription (LRU eviction)
     - Router MAY return ERROR_INVALID_CMD (if error responses are implemented)

3. **Duplicate Subscriptions:**
   - If subscriber already has an active subscription to the same appliance:
     - Router SHOULD update the message_filter for existing subscription
     - Router SHOULD NOT create duplicate table entries

**Important Notes:**
- Subscriptions are NOT persistent - they are lost on power cycle or reset
- Router controllers SHOULD implement subscription timeout (recommend 60 seconds without PING_REQUEST)
- Subscriber controllers SHOULD periodically re-send DATA_SUBSCRIPTION to maintain routing
- Routers MAY limit the number of active subscriptions (implementation-defined, recommend ≥10)

### 0x15 - DATA_UNSUBSCRIPTION

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
  - MUST match an existing subscription
  - If no subscription exists, command is silently ignored

**Behavior:**

When a controller (router) receives this command:
1. Extracts subscriber address from the packet's ADDRESS field (sender of this command)
2. Removes subscriber from routing table for the specified appliance_address
3. No longer forwards data messages from that appliance to the subscriber

**Use Cases:**
- Clean shutdown when subscriber controller is disconnecting
- Reduce bandwidth usage when telemetry is no longer needed
- Change subscription filter (unsubscribe, then re-subscribe with new filter)

### 0x16 - TELEMETRY_CONFIG

Enable or disable periodic telemetry broadcasts, configure broadcast interval, and select telemetry format.

**Payload Structure (12 bytes):**
```
+-------------------+-------------+-----------------+
| telemetry_enabled | interval_ms | telemetry_mode  |
+-------------------+-------------+-----------------+
| u32               | u32         | u32             |
+-------------------+-------------+-----------------+
```

**Fields:**
- **telemetry_enabled** (u32): Telemetry broadcast control
  - 0 = Disable telemetry broadcasts (other parameters ignored)
  - 1 = Enable telemetry broadcasts at specified interval
- **interval_ms** (u32): Telemetry broadcast interval in milliseconds
  - MUST be within range: 100-5000 ms
  - RECOMMENDED: 100 ms (default)
  - Values outside range SHALL be clamped to nearest valid value
- **telemetry_mode** (u32): Telemetry message format
  - 0 = Bundled mode (default) - uses TELEMETRY_BUNDLE message
  - 1 = Individual mode - sends MOTOR_DATA, TEMP_DATA, STATE_DATA separately

**Default State:** Telemetry broadcasts are **disabled** on boot

**Data Message Restriction:**
- **IMPORTANT:** Appliances SHALL NOT send any data messages (0x30-0x3F) until a TELEMETRY_CONFIG command with telemetry_enabled=1 has been received
- The ONLY exception is PING_RESPONSE (0x3F), which may be sent at any time in response to PING_REQUEST
- This prevents unsolicited data messages before the controller is ready to receive them
- Violating this restriction will cause decoder synchronization issues on the controller

**Auto-Disable Behavior:**
- If telemetry is enabled but no PING_REQUEST is received for 30 seconds, telemetry broadcasts are automatically disabled
- This prevents the appliance from continuously transmitting when the controller is disconnected
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

---

## Control Command Formats

**IMPORTANT:** All multi-byte integers (u32, i32, u64, f64) use **little-endian** byte order unless otherwise specified. CRC-16 is the only exception (big-endian).

### 0x1F - DISCOVERY_REQUEST

Request device capabilities from all appliances on the bus.

**Payload:** None (0 bytes)

**Usage:**
- Controller sends this command with ADDRESS field set to broadcast address (0x0000000000000000)
- All appliances on the bus MUST respond with DEVICE_ANNOUNCE (0x36)
- Appliances SHOULD respond immediately (no delay)
- Response includes device address and capability counts

**Response:** DEVICE_ANNOUNCE (0x36) from each appliance

**Example Use Cases:**
- Network discovery on startup
- Detecting new devices added to the bus
- Enumerating available resources (motors, sensors, etc.)
- Building dynamic UI based on available devices

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
  - FAN_MODE: Target RPM (800-3400)
  - HEAT_MODE: Pump rate in milliseconds
  - IDLE_MODE/EMERGENCY: Ignored (set to 0)

**Example:** Enter fan mode at 2500 RPM
```
7E 08 [ADDRESS] 20 00 00 00 01 00 00 09 C4 [CRC-H] [CRC-L] 7F
                ^^mode=1    ^^argument=2500
```

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
- **motor** (i32): Motor index (0-9, typically 0)
- **rpm** (i32): Target RPM (0 = stop, 800-3400 = run)

**Validation:**
- RPM MUST be 0 OR within motor's min/max range
- Invalid RPM MUST return ERROR_INVALID_CMD

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
- **pump** (i32): Pump index (0-9, typically 0)
- **rate_ms** (i32): Pulse interval in milliseconds (0 = stop, ≥100 = run)

**Validation:**
- rate_ms MUST be 0 OR ≥ (pulse_ms + recovery_ms)

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
- **glow** (i32): Glow plug index (0-9, typically 0)
- **duration** (i32): Burn duration in milliseconds (0 = off, max = configured max_duration)

**Validation:**
- duration MUST be 0 to max_duration
- MUST NOT re-light already lit glow plug

### 0x24 - TEMP_COMMAND

Configure temperature controller operation.

**Payload Structure (20 bytes):**
```
+-------------+------+-------------+------------------+
| thermometer | type | motor_index | target_temp      |
+-------------+------+-------------+------------------+
| i32         | u32  | i32         | f64              |
+-------------+------+-------------+------------------+
```

**Fields:**
- **thermometer** (i32): Temperature controller index (0-9, typically 0)
- **type** (u32): Command type
  - 0 = WATCH_MOTOR (associate with motor)
  - 1 = UNWATCH_MOTOR (stop monitoring)
  - 2 = ENABLE_RPM_CONTROL (enable PID)
  - 3 = DISABLE_RPM_CONTROL (disable PID)
  - 4 = SET_TARGET_TEMP (set temperature target)
- **motor_index** (i32): Motor to control (used with WATCH_MOTOR)
- **target_temp** (f64): Target temperature in Celsius (used with SET_TARGET_TEMP)

**Note:** f64 is IEEE 754 double-precision, little-endian byte order

### 0x2F - PING_REQUEST

Connectivity check / heartbeat.

**Payload:** None (0 bytes)

**Response:** PING_RESPONSE (0x3F) with uptime

**Important:** PING_REQUEST resets the telemetry timeout timer. If telemetry is enabled and no PING_REQUEST is received for 30 seconds, telemetry broadcasts SHALL be automatically disabled.

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
- **code** (i32): Error code (application-specific)
- **state** (u32): Current system state
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

### 0x34 - TEMP_DATA

Temperature sensor readings and PID control status.

**Payload Structure (32 bytes):**
```
+-------------+-----------+------+-------------+-------------------+
| thermometer | timestamp | temp | pid_enabled | rpm_ctrl_enabled  |
+-------------+-----------+------+-------------+-------------------+
| i32         | u32       | f64  | u8          | u8                |
+-------------+-----------+------+-------------+-------------------+

+---------------+-----------------+---------+
| watched_motor | target_temp     | padding |
+---------------+-----------------+---------+
| i32           | f64             | 2 bytes |
+---------------+-----------------+---------+
```

**Fields:**
- **thermometer** (i32): Thermometer index
- **timestamp** (u32): Reading timestamp in milliseconds since boot
- **temp** (f64): Temperature in Celsius
- **pid_enabled** (u8): PID active flag (0 = off, 1 = on)
- **rpm_ctrl_enabled** (u8): Motor RPM control flag (0 = off, 1 = on)
- **watched_motor** (i32): Motor being controlled (-1 = none)
- **target_temp** (f64): Target temperature for PID control
- **padding** (2 bytes): Reserved for alignment

**Send Rate:** Per telemetry interval (100ms at default, after sample warmup period)

### 0x35 - TELEMETRY_BUNDLE

Consolidated telemetry packet for efficient polling with support for multiple motors and temperature sensors.

**Payload Structure (Variable: 42-108 bytes):**
```
+-------+-------+-------------+------------+
| state | error | motor_count | temp_count |
+-------+-------+-------------+------------+
| u32   | u8    | u8          | u8         |
+-------+-------+-------------+------------+

+----------+--------+------+------------+
| For each motor (motor_count × 16 bytes):
+----------+--------+------+------------+
| rpm      | target | pwm  | pwm_period |
+----------+--------+------+------------+
| i32      | i32    | i32  | i32        |
+----------+--------+------+------------+

+------+
| For each temperature sensor (temp_count × 8 bytes):
+------+
| temp |
+------+
| f64  |
+------+

+-----------+-----------+--------------+----------------+
| pump_rate | glow_st   | timestamp    | pid_enabled    |
+-----------+-----------+--------------+----------------+
| i32       | u8        | u32          | u8             |
+-----------+-----------+--------------+----------------+

+----------------+---------+
| rpm_ctrl_en    | padding |
+----------------+---------+
| u8             | 1 byte  |
+----------------+---------+
```

**Fields:**
- **state** (u32): Current system state (see STATE_DATA)
- **error** (u8): Error flag
- **motor_count** (u8): Number of motors in this bundle (1-3)
- **temp_count** (u8): Number of temperature sensors in this bundle (1-3)
- **motors[motor_count]**: Array of motor telemetry entries
  - **rpm** (i32): Current motor RPM
  - **target** (i32): Target motor RPM
  - **pwm** (i32): Motor PWM pulse width in microseconds
  - **pwm_period** (i32): PWM period in microseconds (for percentage calculation: pwm/pwm_period × 100)
- **temperatures[temp_count]**: Array of temperature readings
  - **temp** (f64): Temperature in Celsius
- **pump_rate** (i32): Current pump rate in milliseconds
- **glow_status** (u8): Glow plug lit status
- **timestamp** (u32): Bundle timestamp in milliseconds since boot
- **pid_enabled** (u8): Temperature PID active
- **rpm_ctrl_enabled** (u8): Motor RPM control active
- **padding** (1 byte): Reserved for alignment

**Payload Size Calculation:**
```
Size = 7 (header) + (motor_count × 16) + (temp_count × 8) + 11 (footer)
```

**Common Configurations:**

| Motors | Temps | Payload Size | Status | Notes |
|--------|-------|--------------|--------|-------|
| 1 | 1 | 7 + 16 + 8 + 11 = 42 bytes | ✓ Valid | **Typical** (most common) |
| 2 | 1 | 7 + 32 + 8 + 11 = 58 bytes | ✓ Valid | |
| 3 | 1 | 7 + 48 + 8 + 11 = 74 bytes | ✓ Valid | |
| 1 | 2 | 7 + 16 + 16 + 11 = 50 bytes | ✓ Valid | |
| 2 | 2 | 7 + 32 + 16 + 11 = 66 bytes | ✓ Valid | |
| 3 | 2 | 7 + 48 + 16 + 11 = 82 bytes | ✓ Valid | |
| 1 | 3 | 7 + 16 + 24 + 11 = 58 bytes | ✓ Valid | |
| 2 | 3 | 7 + 32 + 24 + 11 = 74 bytes | ✓ Valid | |
| 3 | 3 | 7 + 48 + 24 + 11 = 90 bytes | ✓ Valid | |
| 4 | 3 | 7 + 64 + 24 + 11 = 106 bytes | ✓ Valid | |
| 3 | 4 | 7 + 48 + 32 + 11 = 98 bytes | ✓ Valid | |
| 5 | 3 | 7 + 80 + 24 + 11 = 122 bytes | ✗ Exceeds 114-byte limit | |

**Maximum Payload Constraint:** 114 bytes (to fit within 128-byte packet with framing and address)

**Purpose:** Single packet containing all critical telemetry for efficient monitoring. Supports variable number of motors and temperature sensors for different appliance configurations.

**Send Rate:** Per telemetry interval (100ms at default)

**Validation:**

When encoding or decoding TELEMETRY_BUNDLE, implementations MUST validate:

1. **Count Field Ranges:**
   - motor_count MUST be in range [1, 5]
   - temp_count MUST be in range [1, 4]
   - motor_count = 0 OR temp_count = 0 is INVALID (use individual messages or omit telemetry)
   - Implementations SHOULD reject bundles with count values outside these ranges

2. **Payload Size Constraint:**
   - Total payload MUST NOT exceed 114 bytes (maximum allowed by protocol)
   - Calculated size = 7 + (motor_count × 16) + (temp_count × 8) + 11
   - If calculated size > 114 bytes:
     - Appliance MUST NOT send TELEMETRY_BUNDLE
     - Appliance MUST use individual messages (MOTOR_DATA, TEMP_DATA) instead
     - Controller MUST reject bundles exceeding size limit with ERROR_INVALID_MSG

3. **Device Capability Mismatch:**
   - If motor_count > actual motors available on appliance:
     - Appliance MAY pad with dummy motor data (rpm=0, target=0, pwm=0)
     - Appliance SHOULD only include actual motors in bundle
   - If temp_count > actual temperature sensors:
     - Appliance MAY pad with dummy temperature data (temp=0.0)
     - Appliance SHOULD only include actual sensors in bundle
   - Controllers SHOULD verify counts against DEVICE_ANNOUNCE capabilities
   - Controllers MAY ignore extra motors/temperatures beyond announced capabilities

4. **Array Parsing:**
   - Receivers MUST read motor_count and temp_count BEFORE parsing arrays
   - Receivers MUST validate that received payload size matches calculated size
   - Receivers MUST reject bundles where payload size ≠ expected size

**Notes:**
- Arrays are variable-length based on motor_count and temp_count fields
- Receivers MUST parse both count fields to determine array sizes
- Motor/temperature indices are implicit (array order: 0, 1, 2...)
- For configurations exceeding size limit, MUST use individual messages (MOTOR_DATA, TEMP_DATA) instead
- **Typical configuration:** 1 motor + 1 temperature (42 bytes) - most common appliance setup
- Multi-appliance configurations may use 2-3 motors/temps for expanded capabilities

### 0x36 - DEVICE_ANNOUNCE

Device capabilities announcement sent in response to DISCOVERY_REQUEST.

**Payload Structure (8 bytes):**
```
+-------------+------------------+------------+------------+
| motor_count | thermometer_count | pump_count | glow_count |
+-------------+------------------+------------+------------+
| u8          | u8               | u8         | u8         |
+-------------+------------------+------------+------------+

+---------+
| padding |
+---------+
| 4 bytes |
+---------+
```

**Fields:**
- **motor_count** (u8): Number of motors this device has (0-255)
- **thermometer_count** (u8): Number of temperature sensors (0-255)
- **pump_count** (u8): Number of pumps (0-255)
- **glow_count** (u8): Number of glow plugs (0-255)
- **padding** (4 bytes): Reserved for future expansion (firmware version, device type, etc.)

**Send Rate:** On DISCOVERY_REQUEST only

**Behavior:**
- Sent in response to DISCOVERY_REQUEST with broadcast address
- ADDRESS field contains the appliance's unique 64-bit address
- Appliances SHOULD respond immediately (no artificial delay)
- Multiple appliances may respond simultaneously (acceptable bus contention)
- Controllers SHOULD be prepared to receive multiple DEVICE_ANNOUNCE messages

**Example:**
```
Typical appliance (Helios ICU) responds with:
  motor_count: 1
  thermometer_count: 1
  pump_count: 1
  glow_count: 1

  (This is the most common configuration)

Multi-burner appliance responds with:
  motor_count: 3
  thermometer_count: 3
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
| u32        |
+------------+
```

### 0xE0 - ERROR_INVALID_MSG
Invalid message format or framing error.

**error_code:**
- 0 = Invalid START byte
- 1 = Invalid END byte
- 2 = Length exceeds maximum
- 3 = Timeout waiting for complete packet

### 0xE1 - ERROR_CRC_FAIL
CRC validation failed.

**error_code:**
- Expected CRC value (lower 16 bits)

### 0xE2 - ERROR_INVALID_CMD
Command validation failed.

**error_code:**
- 0 = Unknown message type
- 1 = Invalid parameter value
- 2 = Invalid device index

### 0xE3 - ERROR_STATE_REJECT
Command rejected by state machine.

**error_code:**
- Current state that rejected the command

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

### 2. Periodic Telemetry

Appliance broadcasts telemetry at fixed intervals **when enabled by controller:**

```
Controller → Appliance:  TELEMETRY_CONFIG (enable=1, interval_ms=100, mode=0)

[After enabling, appliance broadcasts at configured interval:]

Bundled Mode (mode=0, default):
  Every <interval_ms>:      TELEMETRY_BUNDLE (consolidated data)
  Every <interval_ms×2.5>:  STATE_DATA

Individual Mode (mode=1):
  Every <interval_ms>:      MOTOR_DATA + TEMP_DATA
  Every <interval_ms×2.5>:  STATE_DATA

Example at 100ms interval, bundled mode (recommended):
  TELEMETRY_BUNDLE every 100ms
  STATE_DATA every 250ms

Example at 500ms interval, individual mode (lower bandwidth):
  MOTOR_DATA + TEMP_DATA every 500ms
  STATE_DATA every 1250ms
```

**Important:**
- Telemetry is **disabled by default** on boot
- Controller MUST explicitly enable telemetry with TELEMETRY_CONFIG command
- **No data messages (except PING_RESPONSE) SHALL be sent until telemetry is enabled**
- Telemetry SHALL auto-disable after 30 seconds without PING_REQUEST
- This prevents boot synchronization issues and reduces unnecessary traffic
- SHOULD use bundled mode (default) for efficiency; MAY use individual mode for flexibility

### 3. Event-Driven Updates

Appliance sends data messages on state changes **when telemetry is enabled:**

```
PUMP_DATA:  Sent on pump cycle events
GLOW_DATA:  Sent when glow plug turns on/off
```

**Note:** Event-driven messages SHALL only be sent when telemetry is enabled (telemetry_enabled=1). They are independent of the telemetry_mode setting and SHALL be sent when their events occur.

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

**Default Timeout:** 30 seconds (configurable via firmware)

**Behavior:**
```
Normal operation:
  Controller → Appliance:  PING_REQUEST (every 10-15 seconds)
  Appliance → Controller:  PING_RESPONSE
  Appliance → Controller:  [Telemetry broadcasts continue if enabled]

Timeout condition:
  [30 seconds with no PING_REQUEST]
  Appliance: Automatically transitions to IDLE mode
  Appliance: Automatically disables telemetry broadcasts
  Appliance → Controller: STATE_DATA (state = IDLE, error = timeout) [final message]
  [No further telemetry until controller re-enables]
```

**Configuration:**
- Timeout mode MUST be **enabled by default**
- Timeout interval is configurable via firmware (default: 30000ms)
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
   message_filter: 0xFFFFFFFF (all data)

2. Router Controller processes subscription:
   - Adds Remote Controller to routing table for Appliance
   - Notes subscription came via WiFi physical layer

3. Appliance sends telemetry (via LIN):
   MSG_TYPE: TELEMETRY_BUNDLE (0x35)
   ADDRESS: Router controller address (or broadcast)

4. Router Controller receives telemetry:
   - Processes locally if needed
   - Checks routing table: Remote Controller subscribed to Appliance data
   - Forwards copy to Remote Controller via WiFi

5. Remote Controller receives telemetry from Appliance through Router
```

**Routing Table Management:**

Router controllers maintain subscription table:

| Subscriber Address | Appliance Address | Message Filter | Physical Layer | Last Activity |
|--------------------|-------------------|----------------|----------------|---------------|
| 0x1234567890ABCDEF | 0xFEDCBA0987654321 | 0xFFFFFFFF | WiFi | 2026-01-05 14:23:10 |
| 0xABCDEF1234567890 | 0xFEDCBA0987654321 | 0x00000020 | Bluetooth | 2026-01-05 14:22:45 |

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
   - For each subscriber, check message_filter bitmask
   - If message type matches filter, forward copy to subscriber's physical layer

4. **Subscription Timeout:**
   - Router SHOULD remove subscriptions after 60 seconds without PING_REQUEST from subscriber
   - This prevents stale routing table entries for disconnected controllers

5. **Loop Prevention:**
   - Routers SHOULD NOT forward packets back to the physical layer they were received from
   - Prevents routing loops in multi-router networks

**Example Scenario - Remote Burner Control:**

User wants to control their heater from a phone app:

1. **Setup:**
   - Appliance: Helios burner ICU (LIN bus, address 0xAABBCCDDEEFF0011)
   - Router: Slate controller (WiFi + LIN, address 0x1122334455667788)
   - Remote: Phone app with WiFi controller (address 0x99887766554433 22)

2. **Connection:**
   - Phone app connects to Router via WiFi
   - Phone app sends DATA_SUBSCRIPTION to Router for Appliance

3. **Control:**
   - User taps "Start Heating" in app
   - Phone app sends STATE_COMMAND (mode=HEAT) with ADDRESS=Appliance to Router via WiFi
   - Router forwards command to Appliance via LIN
   - Appliance starts heating

4. **Monitoring:**
   - Appliance sends TELEMETRY_BUNDLE every 500ms to Router via LIN
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

| Type | Size | Format | Range |
|------|------|--------|-------|
| u8   | 1 byte | Unsigned integer | 0 to 255 |
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
  - **Production (RS-485):** 115200 baud (less common, multi-wire installations)
  - **Development (UART):** 115200 baud (prototyping only)
- **Data Bits:** MUST be 8
- **Parity:** MUST be None
- **Stop Bits:** MUST be 1
- **Flow Control:** MUST be None

**Note:** The baud rate at the MCU UART depends on the physical layer. For LIN production deployments, the MCU communicates with the transceiver IC at 19.2 kbaud. The transceiver handles LIN bus timing. For RS-485 and plain UART, the MCU operates at 115.2 kbaud directly.

### Buffer Requirements

**Receive Buffer:** MUST be minimum 256 bytes (2x max packet size for byte stuffing)
**Transmit Buffer:** MUST be minimum 256 bytes (2x max packet size for byte stuffing)

### Protocol Behavior

These requirements apply to all implementations unless otherwise noted.

#### Appliance Transmission Requirements

**Appliances MUST NOT transmit data messages (0x30-0x3F) unless:**
- Responding to a PING_REQUEST with PING_RESPONSE, OR
- Telemetry broadcasting has been enabled via TELEMETRY_CONFIG command

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
- Continue retransmitting until TELEMETRY_BUNDLE received with state = E_STOP (0x08)
- Once emergency stop confirmed, stop retransmitting command

**Rationale:** This ensures emergency stop is reliably entered even if the initial command is lost or corrupted, providing safety-critical reliability for emergency stop activation.

**Example - Controller-Initiated Emergency Stop:**
```
Controller sends: STATE_COMMAND (mode=EMERGENCY)
Controller starts: Retransmitting EMERGENCY every 250ms
Appliance receives: STATE_COMMAND (mode=EMERGENCY)
Appliance enters: E_STOP state
Appliance begins: Broadcasting TELEMETRY_BUNDLE every 250ms
Controller receives: TELEMETRY_BUNDLE (state=E_STOP, ...)
Controller stops: Retransmitting EMERGENCY command
Controller continues: Receiving TELEMETRY_BUNDLE every 250ms
... continues until power cycle ...
```

#### Emergency Stop Behavior (Appliances Only)

**When an appliance enters an emergency stop state, it MUST:**
- Ignore ALL received commands, including PING_REQUEST
- Transmit TELEMETRY_BUNDLE indicating emergency stop every 250ms
- Continue emergency stop broadcasts until power cycle or hardware reset

**Rationale:** Emergency stop is a safety-critical state that requires immediate visibility to the controller and prevents any command processing that could interfere with safe shutdown.

**Transmission During Emergency Stop:**
- MUST transmit TELEMETRY_BUNDLE with state = E_STOP (0x08)
- Includes all sensor data (motor, temperature, etc.) for diagnostics
- Broadcast interval: 250ms (fixed, not configurable)
- Broadcasts occur regardless of TELEMETRY_CONFIG state

**Recovery:**
- Emergency stop state can ONLY be cleared by:
  - Power cycle (complete power loss and restoration)
  - Hardware reset (physical reset button or watchdog)
- Software commands MUST NOT clear emergency stop state

**Example - Appliance-Initiated Emergency Stop:**
```
Appliance detects fault temperature (>275°C)
Appliance enters: E_STOP state
Appliance begins: Broadcasting TELEMETRY_BUNDLE every 250ms
Controller sends: PING_REQUEST (ignored by appliance)
Controller sends: STATE_COMMAND(IDLE) (ignored by appliance)
Controller receives: TELEMETRY_BUNDLE (state=E_STOP, error=OVERHEAT, temp=280°C, ...) every 250ms
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
- Continue retrying until broadcast data is confirmed

**Example:**
```
Controller sends: TELEMETRY_CONFIG (enabled=true, interval=100ms)
Controller receives: PING_RESPONSE (uptime=5000ms)
Controller checks: Have we received TELEMETRY_BUNDLE? NO
Controller action: Retransmit TELEMETRY_CONFIG
Controller receives: PING_RESPONSE (uptime=15000ms)
Controller checks: Have we received TELEMETRY_BUNDLE? NO
Controller action: Retransmit TELEMETRY_CONFIG
Controller receives: TELEMETRY_BUNDLE (state=HEATING, ...)
Controller checks: Have we received TELEMETRY_BUNDLE? YES
Controller action: Stop retrying, broadcasts working
```

---

## Error Handling

### Transmit Errors
- **Buffer Full:** Drop oldest packet or block until space available
- **UART Error:** Log error, attempt retransmit once

### Receive Errors
- **CRC Failure:** Send ERROR_CRC_FAIL, discard packet
- **Framing Error:** Send ERROR_INVALID_MSG, resync to next START byte
- **Timeout:** Discard partial packet after 100ms silence
- **Invalid Command:** Send ERROR_INVALID_CMD with error code

### Recovery
- On 3 consecutive CRC failures, suggest baud rate mismatch
- On persistent framing errors, suggest physical connection check

---

## Performance Characteristics

### Throughput

**At Default 100ms Telemetry Period:**
- TELEMETRY_BUNDLE (3 motors + 3 temps): ~90 bytes raw, ~120 bytes after stuffing + address = 1200 bytes/sec
- TELEMETRY_BUNDLE (2 motors + 2 temps): ~66 bytes raw, ~90 bytes after stuffing + address = 900 bytes/sec
- Individual messages (MOTOR + TEMP + STATE): ~100 bytes = 1000 bytes/sec

**At 500ms Telemetry Period (Lower Bandwidth):**
- TELEMETRY_BUNDLE (3 motors + 3 temps): ~120 bytes after stuffing + address = 240 bytes/sec
- TELEMETRY_BUNDLE (2 motors + 2 temps): ~90 bytes after stuffing + address = 180 bytes/sec
- Individual messages: ~100 bytes = 200 bytes/sec

**At 115200 baud:**
- Effective throughput: ~11,520 bytes/sec
- Telemetry overhead: ~2-10% bandwidth utilization (depending on interval and configuration)

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

## Migration from Helios Serial Protocol v1.4

This section documents breaking changes for implementations migrating from Helios Serial Protocol v1.4.

### Packet Format Changes

1. **Address Field Added:**
   - New 8-byte address field inserted after LENGTH, before MSG_TYPE
   - Packet size increased: 6 bytes overhead → 14 bytes overhead
   - Maximum payload reduced: 122 bytes → 114 bytes
   - Buffer requirements unchanged (256 bytes still sufficient with stuffing)

2. **Message Type Reorganization:**
   - Configuration commands: 0x10-0x1F (new)
   - Control commands: 0x20-0x2F (renumbered from 0x10-0x1F)
   - Telemetry data: 0x30-0x3F (renumbered from 0x20-0x2F)
   - Error messages: 0xE0-0xEF (unchanged)

### Message Type Mapping

| Old Type (v1.4) | Old Name | New Type | New Name |
|-----------------|----------|----------|----------|
| 0x10 | STATE_COMMAND | 0x20 | STATE_COMMAND |
| 0x11 | MOTOR_COMMAND | 0x21 | MOTOR_COMMAND |
| 0x12 | PUMP_COMMAND | 0x22 | PUMP_COMMAND |
| 0x13 | GLOW_COMMAND | 0x23 | GLOW_COMMAND |
| 0x14 | TEMP_COMMAND | 0x24 | TEMP_COMMAND |
| 0x16 | TELEMETRY_CONFIG | 0x16 | TELEMETRY_CONFIG (unchanged) |
| (new) | - | 0x1F | DISCOVERY_REQUEST (new) |
| 0x1F | PING_REQUEST | 0x2F | PING_REQUEST |
| 0x20 | STATE_DATA | 0x30 | STATE_DATA |
| 0x21 | MOTOR_DATA | 0x31 | MOTOR_DATA |
| 0x22 | PUMP_DATA | 0x32 | PUMP_DATA |
| 0x23 | GLOW_DATA | 0x33 | GLOW_DATA |
| 0x24 | TEMP_DATA | 0x34 | TEMP_DATA |
| 0x25 | TELEMETRY_BUNDLE | 0x35 | TELEMETRY_BUNDLE |
| (new) | - | 0x36 | DEVICE_ANNOUNCE (new) |
| 0x2F | PING_RESPONSE | 0x3F | PING_RESPONSE |

### New Features

1. **Configuration Commands (0x10-0x13, 0x16):**
   - MOTOR_CONFIG (0x10): Configure PWM, PID, RPM limits
   - PUMP_CONFIG (0x11): Configure pulse timing
   - TEMP_CONFIG (0x12): Configure PID and sampling
   - GLOW_CONFIG (0x13): Configure max duration
   - TELEMETRY_CONFIG (0x16): Configure telemetry broadcasts

2. **Device Addressing:**
   - All packets now include 64-bit source/destination address
   - Enables multi-device networks
   - Broadcast address (0x0000000000000000) for simultaneous control

3. **Device Discovery:**
   - DISCOVERY_REQUEST (0x1F): Broadcast command to discover devices
   - DEVICE_ANNOUNCE (0x36): Response with device capabilities (motor/temp/pump/glow counts)

4. **Terminology Changes:**
   - Master → Controller
   - Slave → Appliance
   - (New) Monitor device type

### Migration Checklist

**For Protocol Implementations:**
- [ ] Add 8-byte address field to packet encoder/decoder
- [ ] Update message type constants (see mapping table)
- [ ] Reduce maximum payload from 122 to 114 bytes
- [ ] Implement configuration command handlers (0x10-0x15)
- [ ] Update CRC calculation to include address field

**For Applications:**
- [ ] Assign unique 64-bit addresses to all devices
- [ ] Update command transmission to include destination address
- [ ] Update data reception to filter/match source addresses
- [ ] Implement configuration persistence for new config commands
- [ ] Update terminology in UI/documentation (master→controller, slave→appliance)

**Backward Compatibility:**
- **NOT SUPPORTED** - This is a breaking protocol change
- Old v1.4 devices cannot communicate with new Fusain devices
- Network upgrades must be performed atomically (all devices at once)

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
   - Telemetry broadcasts occur at staggered intervals (configured per-device)

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

**Use Case:** Production deployments for retrofitting heaters with single-wire communication to controller.

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
   - No random delay needed (LIN master-slave avoids collisions)

**MCU Integration:**

The MCU firmware is identical to UART/RS-485 configurations. The transceiver handles all LIN-specific details. No firmware changes required - the Fusain library works unchanged regardless of physical layer.

**Benefits of LIN for Production:**
- **Single-wire topology:** Ideal for retrofits with limited wiring
- **Automotive-grade reliability:** Proven in harsh environments
- **Built-in collision avoidance:** LIN master-slave scheduling prevents bus contention
- **Lower cost:** Cheaper than CAN bus, simpler than RS-485
- **Integrated features:** Many transceivers include voltage regulation, diagnostics
- **Standard compliance:** LIN 2.0+ specification ensures interoperability

**Performance Characteristics:**

| Metric | Value | Notes |
|--------|-------|-------|
| **Baud Rate** | 19.2 kbaud | LIN standard, sufficient for Fusain |
| **Max Distance** | ~40m | Shorter than RS-485, acceptable for heater installations |
| **Max Nodes** | 16 | 1 controller + up to 15 appliances |
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
| **Multi-drop** | No | Yes (32-256 nodes) | Yes (16 nodes) |
| **Wiring** | 2-wire | 2-wire (differential) | 1-wire + ground |
| **Noise Immunity** | Low | High | Medium-High |
| **Collision Handling** | None | None (SW required) | Master-controlled |
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

2. **Multi-wire installations, long distances (>40m), or high node count (>16):** RS-485
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
| 2.0 | 2026-01-05 | Thermoquad Team | **BREAKING CHANGE:** Complete protocol redesign. Renamed from "Helios Serial Protocol" to "Fusain Protocol". Added 64-bit device addressing for multi-device networks. Reorganized message types: configuration (0x10-0x1F), control (0x20-0x2F), telemetry (0x30-0x3F), errors (0xE0-0xEF). Added comprehensive configuration commands for motor (PWM, PID, RPM limits), pump (pulse timing), temperature (PID, sampling), and glow plug (max duration). Removed application-specific state machine configuration (STATE_CONFIG). Added device discovery mechanism: DISCOVERY_REQUEST (0x1F) broadcast command and DEVICE_ANNOUNCE (0x36) response with device capabilities. Updated terminology: master→controller, slave→appliance, added monitor device type. Maximum payload reduced from 122 to 114 bytes due to address field. Standardized units: timestamps in milliseconds since boot, PWM values in microseconds, pump/glow timing in milliseconds. Defined physical layer architecture: LIN (default production, via UART-to-LIN transceiver IC for single-wire retrofits), RS-485 (production for multi-wire installations), plain UART (development only). Added comprehensive multi-drop RS-485 topology documentation. Not backward compatible with v1.x. |
| 1.4 | 2026-01-03 | Helios Team | Added pwm_period field to TELEMETRY_BUNDLE motor entries. Increased maximum packet size from 64 to 128 bytes. |
| 1.3 | 2026-01-02 | Helios Team | Added Protocol Behavior subsection documenting slave transmission restrictions, byte synchronization, emergency stop behavior, and broadcast retry requirements. |
| 1.2 | 2026-01-02 | Helios Team | Updated specification to use RFC 2119 requirement language. |
| 1.1 | 2026-01-02 | Helios Team | Added TELEMETRY_CONFIG command for broadcast control with configurable interval and mode selection. Telemetry now disabled by default. |
| 1.0 | 2025-12-31 | Helios Team | Initial Helios Serial Protocol specification |

---

## References

- **RFC 2119:** Key words for use in RFCs to Indicate Requirement Levels - https://www.rfc-editor.org/rfc/rfc2119.txt
- **CRC-16-CCITT:** ITU-T Recommendation V.41
- **IEEE 754:** IEEE Standard for Floating-Point Arithmetic
- **LIN Specification:** LIN Consortium, LIN 2.0 Protocol Specification
