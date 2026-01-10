Implementation Guide
####################

This guide covers implementing the :term:`Fusain` protocol in firmware and software
applications. For protocol specification, see :doc:`packet-format`. For message
definitions, see :doc:`messages`.


.. _impl-buffers:

Buffers
*******

Implementations MUST allocate buffers large enough to handle worst-case byte
stuffing.


Receive Buffer
--------------

The receive buffer holds stuffed bytes between START and END delimiters.

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - Calculation
     - Size
   * - Unstuffed content (128-byte packet minus START and END delimiters)
     - 128 − 2 = 126 bytes
   * - Worst-case stuffing (every byte escaped)
     - 126 × 2 = 252 bytes
   * - **Minimum buffer size**
     - **256 bytes**

If 256 bytes are received without an END delimiter, the buffer MUST be
discarded and the receiver MUST resynchronize by waiting for a new START
delimiter.


Transmit Buffer
---------------

The transmit buffer holds the encoded packet before transmission.

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - Calculation
     - Size
   * - Maximum packet size
     - 128 bytes
   * - Worst-case stuffing
     - 252 bytes
   * - START and END delimiters
     - 2 bytes
   * - **Minimum buffer size**
     - **256 bytes**


Timeouts
********

Inter-Byte Timeout
------------------

Implementations MUST discard partial packets after 100ms of silence (no bytes
received).

This timeout ensures that incomplete packets due to transmission errors,
disconnections, or noise do not permanently block the decoder. When the timeout
elapses mid-packet, the receiver resets to searching for a new START byte.

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Parameter
     - Value
   * - Inter-byte timeout
     - 100ms
   * - Action on timeout
     - Discard partial packet, reset to START search


Communication Timeout
---------------------

:ref:`Appliances <fusain-device-roles>` track time since last :ref:`PING_REQUEST <msg-ping-request>` received.
This provides a safety mechanism for unattended operation.

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Parameter
     - Value
   * - Default timeout
     - 30 seconds
   * - Configurable range
     - 5–60 seconds
   * - Action on timeout
     - Transition to IDLE, disable telemetry

**Important:** Only :ref:`PING_REQUEST <msg-ping-request>` resets the communication
timeout timer. Other commands (:ref:`STATE_COMMAND <msg-state-command>`,
:ref:`MOTOR_COMMAND <msg-motor-command>`, :ref:`SEND_TELEMETRY <msg-send-telemetry>`,
etc.) do NOT reset the timer.

Controllers SHOULD send :ref:`PING_REQUEST <msg-ping-request>` every 10–15 seconds
to maintain communication.


Byte Synchronization
********************

All implementations MUST ignore bytes received on the serial line until a valid
START byte (``0x7E``) is observed.

**Rationale:** This ensures proper frame synchronization and prevents
misinterpretation of noise, garbage bytes, or mid-packet data as valid packets.

**Behavior:**

1. Discard all bytes until START byte detected
2. After START byte, begin packet decoding
3. On decode error, reset to searching for START byte
4. Continue until valid packet received or error occurs


Packet Decoder
**************

The packet decoder is a state machine that processes incoming bytes.

States
------

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - State
     - Description
   * - WAIT_START
     - Waiting for START delimiter (``0x7E``)
   * - READ_LENGTH
     - Reading LENGTH byte
   * - READ_ADDRESS
     - Reading 8-byte ADDRESS field
   * - READ_TYPE
     - Reading MSG_TYPE byte
   * - READ_PAYLOAD
     - Reading PAYLOAD bytes (length from LENGTH field)
   * - READ_CRC
     - Reading 2-byte CRC field
   * - WAIT_END
     - Waiting for END delimiter (``0x7F``)


State Transitions
-----------------

.. code-block:: text

   WAIT_START
       |
       | [0x7E received]
       v
   READ_LENGTH
       |
       | [LENGTH byte received, LENGTH <= 114]
       v
   READ_ADDRESS
       |
       | [8 bytes received]
       v
   READ_TYPE
       |
       | [MSG_TYPE byte received]
       v
   READ_PAYLOAD
       |
       | [LENGTH bytes received]
       v
   READ_CRC
       |
       | [2 bytes received]
       v
   WAIT_END
       |
       | [0x7F received] → Validate CRC → Process packet
       v
   WAIT_START


Byte Unstuffing
---------------

Apply byte unstuffing during reception for all bytes between START and END.
For escape sequences, see :ref:`byte-stuffing` in Packet Format.

1. If ``0x7D`` received, read next byte and XOR with ``0x20``
2. Otherwise, use byte as-is


Error Conditions
----------------

**START byte (0x7E) received mid-packet:**

- Abandon current packet immediately
- Treat the new START byte as beginning of a new packet
- Log error if applicable (previous packet was corrupted or incomplete)

**LENGTH field exceeds maximum (>114):**

- Immediately reject the packet
- Reset receive buffer
- Discard all bytes until next START byte detected

**Buffer overflow (>256 bytes without END):**

- Reset receive buffer immediately
- Discard all bytes until next START byte detected

**END byte (0x7F) received before expected:**

- Packet incomplete or corrupted
- Reset receive buffer immediately
- Discard all bytes until next START byte detected

**CRC validation failure:**

- Discard packet silently
- Reset to WAIT_START state
- Do NOT send error response


Packet Encoder
**************

The packet encoder builds a complete packet for transmission.

Encoding Steps
--------------

1. Build the unstuffed packet:

   - LENGTH (1 byte): payload length
   - ADDRESS (8 bytes): destination or source address
   - MSG_TYPE (1 byte): message type identifier
   - PAYLOAD (0–114 bytes): message data

2. Calculate CRC-16-CCITT over the unstuffed packet

3. Append CRC (2 bytes, big-endian: MSB first, then LSB)

4. Apply byte stuffing to all bytes (LENGTH through CRC)

5. Add START delimiter (``0x7E``) before stuffed data

6. Add END delimiter (``0x7F``) after stuffed data


Byte Stuffing
-------------

Apply byte stuffing to all bytes between START and END. For escape sequences,
see :ref:`byte-stuffing` in Packet Format.


CRC Calculation
---------------

Calculate CRC-16-CCITT over the unstuffed packet content. For algorithm
parameters, see :ref:`packet-crc` in Packet Format.


Transmission Requirements
*************************

Appliance Transmission Rules
----------------------------

Appliances MUST NOT transmit data messages (``0x30``–``0x35``) or
:ref:`PING_RESPONSE <msg-ping-response>` (``0x3F``) unless:

- Responding to :ref:`DISCOVERY_REQUEST <msg-discovery-request>` with
  :ref:`DEVICE_ANNOUNCE <msg-device-announce>`, OR
- Responding to :ref:`PING_REQUEST <msg-ping-request>` with
  :ref:`PING_RESPONSE <msg-ping-response>`, OR
- Telemetry broadcasting has been enabled via
  :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`, OR
- Appliance is in E_STOP state (see Emergency Stop Behavior)

**Rationale:** This prevents boot synchronization errors by ensuring the
controller is ready to receive data before the appliance begins broadcasting.


Telemetry Default State
-----------------------

- Telemetry broadcasts are **disabled** by default on boot
- Controller MUST explicitly enable telemetry with
  :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`
- No data messages (except :ref:`PING_RESPONSE <msg-ping-response>` and E_STOP
  broadcasts) are sent until telemetry is enabled


Controller Recovery
-------------------

If the controller's receive buffer becomes out of sync (repeated decode errors):

1. Send :ref:`TELEMETRY_CONFIG <msg-telemetry-config>` with ``enabled=0`` to stop
   incoming telemetry
2. Clear receive buffer and reset decoder state
3. Wait for silence (100ms with no bytes)
4. Send :ref:`TELEMETRY_CONFIG <msg-telemetry-config>` with ``enabled=1`` to resume
   telemetry

This is particularly useful during boot or after communication errors when
packet boundaries are lost.


.. _impl-estop:

Emergency Stop Behavior
***********************

Emergency stop requires special handling for safety-critical shutdown. See
:ref:`msg-state-command` for command details.


Controller Behavior
-------------------

When transmitting :ref:`STATE_COMMAND <msg-state-command>` with ``mode=EMERGENCY``:

1. Retransmit the EMERGENCY command every 250ms
2. Continue retransmitting until :ref:`STATE_DATA <msg-state-data>` received with
   ``state=E_STOP``
3. Once confirmed, stop retransmitting

.. code-block:: text

   Controller sends: STATE_COMMAND (mode=EMERGENCY)
   Controller waits: 250ms
   Controller sends: STATE_COMMAND (mode=EMERGENCY)
   ...
   Appliance enters: E_STOP state
   Appliance begins: Broadcasting telemetry every 250ms
   Controller receives: STATE_DATA (state=E_STOP)
   Controller stops: Retransmitting EMERGENCY command


Appliance Behavior
------------------

When entering emergency stop state:

1. Suspend the communication timeout (timeout MUST NOT cause transition to IDLE)
2. Ignore ALL received commands (including :ref:`PING_REQUEST <msg-ping-request>`
   and :ref:`SEND_TELEMETRY <msg-send-telemetry>`)
3. Transmit all telemetry messages every 250ms:

   - :ref:`STATE_DATA <msg-state-data>` (with ``state=E_STOP`` and appropriate
     error code)
   - :ref:`MOTOR_DATA <msg-motor-data>` (for each motor)
   - :ref:`TEMPERATURE_DATA <msg-temperature-data>` (for each sensor)
   - :ref:`PUMP_DATA <msg-pump-data>`
   - :ref:`GLOW_DATA <msg-glow-data>`

4. Continue broadcasting regardless of :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`
   state

**Recovery:** Emergency stop can ONLY be cleared by:

- Power cycle (complete power loss and restoration)
- Hardware reset (physical reset button or watchdog)

Software commands MUST NOT clear emergency stop state.


Broadcast Emergency Stop
------------------------

Controllers can send :ref:`STATE_COMMAND <msg-state-command>` with
``mode=EMERGENCY`` to the broadcast address to stop all appliances simultaneously.
Per broadcast rules, appliances process the command but do NOT respond.

.. code-block:: text

   Controller sends: STATE_COMMAND (mode=EMERGENCY, ADDRESS=broadcast)
   Appliance 1: Receives, enters E_STOP, does NOT respond
   Appliance 2: Receives, enters E_STOP, does NOT respond
   Appliance 3: Receives, enters E_STOP, does NOT respond
   Controller: Does NOT expect responses (broadcast command)

Because broadcast commands do not receive responses, the controller MUST verify
each appliance has entered E_STOP by checking subsequent
:ref:`STATE_DATA <msg-state-data>` messages. Each appliance in E_STOP broadcasts
telemetry every 250ms regardless of whether the original command was addressed
or broadcast.

**Recommended Approach:**

1. Send :ref:`STATE_COMMAND <msg-state-command>` (mode=EMERGENCY) to broadcast
   address
2. Continue retransmitting every 250ms
3. Monitor incoming :ref:`STATE_DATA <msg-state-data>` messages
4. Track which appliances have reported ``state=E_STOP``
5. Stop retransmitting only when ALL appliances are confirmed in E_STOP


Error Handling
**************

Transmit Errors
---------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Error
     - Action
   * - Buffer full
     - Drop oldest packet or block until space available
   * - UART error
     - Log error, attempt retransmit once


Receive Errors
--------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Error
     - Action
   * - CRC failure
     - Discard packet silently, resync to next START byte
   * - Framing error
     - Discard packet silently, resync to next START byte
   * - Inter-byte timeout
     - Discard partial packet after 100ms silence
   * - Invalid command
     - Send :ref:`ERROR_INVALID_CMD <msg-error-invalid-cmd>` with appropriate error
       code


Recovery Strategies
-------------------

**Consecutive CRC failures:**

- On 3 consecutive CRC failures, check for baud rate mismatch
- Verify physical layer configuration matches between devices

**Persistent framing errors:**

- Check physical connection (cable, connectors)
- Verify termination resistors (RS-485)
- Check for electrical noise or interference

**No response from appliance:**

- Retransmit important commands if no acknowledgment received
- For critical commands (emergency stop), continue retransmitting until
  confirmed via :ref:`STATE_DATA <msg-state-data>`


Broadcast Retry Requirements
****************************

Controllers Only
----------------

If a controller enables broadcast mode on an appliance via
:ref:`TELEMETRY_CONFIG <msg-telemetry-config>`, the controller MUST retransmit
the broadcast enable command every time a
:ref:`PING_RESPONSE <msg-ping-response>` is received from that appliance if the
controller has NOT received a corresponding broadcast data message.

**Rationale:** This provides automatic recovery if:

- Appliance was power-cycled (telemetry disabled on boot)
- Appliance reset for any reason
- Communication was interrupted and broadcasting stopped

**Implementation:**

1. Track which broadcasts have been enabled per appliance
2. Track which broadcast data has been received
3. On :ref:`PING_RESPONSE <msg-ping-response>`, compare enabled vs received
4. If enabled but no data received, retransmit
   :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`


.. _impl-data-types:

Data Type Encoding
******************

All multi-byte integers MUST use **little-endian** byte order.

All payload structures MUST be explicitly **packed** (no padding between
fields). Use compiler-specific attributes:

- GCC/Clang: ``__attribute__((packed))``
- MSVC: ``#pragma pack(1)``

.. list-table::
   :header-rows: 1
   :widths: 15 15 35 35

   * - Type
     - Size
     - Format
     - Range
   * - u8
     - 1 byte
     - Unsigned integer
     - 0 to 255
   * - i8
     - 1 byte
     - Signed integer
     - −128 to 127
   * - i32
     - 4 bytes
     - Signed integer, little-endian
     - −2,147,483,648 to 2,147,483,647
   * - u32
     - 4 bytes
     - Unsigned integer, little-endian
     - 0 to 4,294,967,295
   * - u64
     - 8 bytes
     - Unsigned integer, little-endian
     - 0 to 18,446,744,073,709,551,615
   * - f64
     - 8 bytes
     - IEEE 754 double, little-endian
     - ±1.7×10³⁰⁸


Encoding Examples
-----------------

**Float encoding (225.5°C):**

.. code-block:: text

   IEEE 754 double: 0x406C280000000000
   Little-endian:   00 00 00 00 00 28 6C 40

**Address encoding (0x123456789ABCDEF0):**

.. code-block:: text

   Big-endian (standard): 12 34 56 78 9A BC DE F0
   Little-endian (wire):  F0 DE BC 9A 78 56 34 12


Performance Characteristics
***************************

This section provides guidance on expected protocol performance.


Throughput
----------

Telemetry bandwidth depends on the number of peripherals and configured
interval.

**At 100ms Telemetry Interval:**

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - Configuration
     - Bytes per Interval
     - Throughput
   * - 1 motor, 1 temperature sensor
     - ~70 bytes
     - ~700 bytes/sec
   * - 3 motors, 3 temperature sensors
     - ~200 bytes
     - ~2000 bytes/sec

**At 500ms Telemetry Interval (LIN networks):**

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - Configuration
     - Bytes per Interval
     - Throughput
   * - 1 motor, 1 temperature sensor
     - ~70 bytes
     - ~140 bytes/sec
   * - 3 motors, 3 temperature sensors
     - ~200 bytes
     - ~400 bytes/sec

**Bandwidth Utilization:**

At 115200 baud (effective ~11,520 bytes/sec) or 230400 baud (~23,040 bytes/sec),
telemetry overhead is typically 1–17% of available bandwidth depending on
interval and peripheral count.


Latency
-------

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Operation
     - Latency
   * - Command processing
     - < 5ms (typical)
   * - Telemetry delay (100ms interval)
     - 0–100ms
   * - Telemetry delay (500ms interval)
     - 0–500ms
   * - Multi-hop routing (per hop)
     - 50–200ms


Reliability
-----------

- **CRC-16-CCITT:** Detects all single-bit and double-bit errors
- **Byte stuffing:** Prevents false START/END detection in data
- **Framing:** Robust resynchronization on errors
- **Timeout mode:** Automatic IDLE transition on communication loss
