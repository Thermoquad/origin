Implementation Guide
####################

This guide covers implementing the :term:`Fusain` protocol in firmware and software
applications. For protocol specification, see :doc:`packet-format`. For message
definitions, see :doc:`messages`. For communication patterns and telemetry, see
:doc:`communication-patterns`.


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
   * - Unstuffed content (127-byte packet minus START and END delimiters)
     - 127 − 2 = 125 bytes
   * - Worst-case stuffing (every byte escaped)
     - 125 × 2 = 250 bytes
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
     - 127 bytes
   * - Worst-case stuffing
     - 250 bytes
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
to maintain communication. For physical layer timing, see :doc:`physical-layer`.


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
   * - READ_PAYLOAD
     - Reading CBOR payload bytes (length from LENGTH field)
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
       | [0x7F received] → Validate CRC → Decode CBOR → Process message
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

1. Encode the message as CBOR: ``[type, payload_map]``

2. Build the unstuffed packet:

   - LENGTH (1 byte): CBOR payload length
   - ADDRESS (8 bytes): destination or source address
   - PAYLOAD (0–114 bytes): CBOR-encoded message

3. Calculate CRC-16-CCITT over the unstuffed packet

4. Append CRC (2 bytes, big-endian: MSB first, then LSB)

5. Apply byte stuffing to all bytes (LENGTH through CRC)

6. Add START delimiter (``0x7E``) before stuffed data

7. Add END delimiter (``0x7F``) after stuffed data


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


.. _impl-cbor:

CBOR Encoding
*************

Fusain payloads use CBOR (Concise Binary Object Representation) encoding.
The canonical schema is defined in ``fusain.cddl``.

Message Structure
-----------------

All messages are encoded as a CBOR array with two elements:

.. code-block:: text

   [type, payload]

- **type**: Message type identifier (uint, e.g., ``0x30`` for STATE_DATA)
- **payload**: CBOR map with integer keys, or ``nil`` for empty payloads

Example STATE_DATA encoding:

.. code-block:: text

   [0x30, {0: false, 1: 0, 2: 1, 3: 12345}]

   CBOR bytes: 82 18 30 A4 00 F4 01 00 02 01 03 19 30 39

zcbor Integration (Zephyr)
--------------------------

For Zephyr builds, use zcbor for CBOR encoding/decoding. Enable in Kconfig:

.. code-block:: kconfig

   CONFIG_ZCBOR=y
   CONFIG_ZCBOR_CANONICAL=y

Generate encode/decode functions from the CDDL schema:

.. code-block:: cmake

   zcbor_generate(
     fusain_cbor
     ${CMAKE_CURRENT_SOURCE_DIR}/fusain.cddl
     ${CMAKE_CURRENT_BINARY_DIR}/generated
     --decode --encode
   )

Usage example:

.. code-block:: c

   #include "fusain_cbor_types.h"
   #include "fusain_cbor_encode.h"

   int fusain_encode_state_data(uint8_t *buf, size_t buf_size,
                                bool error, int code, int state, uint32_t ts)
   {
       ZCBOR_STATE_E(zs, 1, buf, buf_size, 1);

       bool ok = zcbor_list_start_encode(zs, 2);
       ok = ok && zcbor_uint32_put(zs, 0x30);  // MSG_STATE_DATA

       ok = ok && zcbor_map_start_encode(zs, 4);
       ok = ok && zcbor_uint32_put(zs, 0) && zcbor_bool_put(zs, error);
       ok = ok && zcbor_uint32_put(zs, 1) && zcbor_int32_put(zs, code);
       ok = ok && zcbor_uint32_put(zs, 2) && zcbor_uint32_put(zs, state);
       ok = ok && zcbor_uint32_put(zs, 3) && zcbor_uint32_put(zs, ts);
       ok = ok && zcbor_map_end_encode(zs, 4);

       ok = ok && zcbor_list_end_encode(zs, 2);

       return ok ? (buf_size - zs->payload_end + zs->payload) : -1;
   }


.. _impl-zephyr:

Zephyr Integration
******************

For Zephyr RTOS builds, these subsystems are recommended for Fusain
implementations.

CRC Implementation
------------------

Use Zephyr's native CRC in Zephyr mode:

.. code-block:: c

   #ifdef CONFIG_ZEPHYR
     #include <zephyr/sys/crc.h>
     #define fusain_crc16(data, len) crc16_itu_t(0xFFFF, data, len)
   #else
     uint16_t fusain_crc16(const uint8_t *data, size_t len);
   #endif

Buffer Management
-----------------

Use ``net_buf`` for packet buffer management:

.. code-block:: c

   #include <zephyr/net/buf.h>

   NET_BUF_POOL_DEFINE(fusain_pool, 8, FUSAIN_MAX_PACKET_SIZE, 0, NULL);

   struct net_buf *buf = net_buf_alloc(&fusain_pool, K_NO_WAIT);
   // ... use buffer ...
   net_buf_unref(buf);

Benefits:

- Pool-based allocation (no heap fragmentation)
- Reference counting for safe buffer sharing
- Consistent patterns across Zephyr subsystems

Logging
-------

Register a logging module for debug output:

.. code-block:: c

   #include <zephyr/logging/log.h>
   LOG_MODULE_REGISTER(fusain, CONFIG_FUSAIN_LOG_LEVEL);

Add Kconfig for log level control:

.. code-block:: kconfig

   module = FUSAIN
   module-str = Fusain Protocol
   source "subsys/logging/Kconfig.template.log_config"

Packet Reception Architecture
-----------------------------

For platforms using UART polling (such as RP2350), use a ring buffer for
byte accumulation combined with a message queue for thread-safe packet handoff:

.. code-block:: c

   #include <zephyr/sys/ring_buffer.h>
   #include <zephyr/net/buf.h>

   RING_BUF_DECLARE(uart_rx_ring, 256);
   K_MSGQ_DEFINE(packet_queue, sizeof(struct net_buf *), 8, 4);

   // UART polling thread
   void uart_poll_thread(void)
   {
       uint8_t byte;
       while (1) {
           while (uart_poll_in(uart_dev, &byte) == 0) {
               ring_buf_put(&uart_rx_ring, &byte, 1);
           }

           struct net_buf *buf = try_extract_packet(&uart_rx_ring);
           if (buf) {
               k_msgq_put(&packet_queue, &buf, K_NO_WAIT);
           }

           k_sleep(K_MSEC(1));
       }
   }

   // Processing thread
   void process_thread(void)
   {
       struct net_buf *buf;
       while (1) {
           if (k_msgq_get(&packet_queue, &buf, K_FOREVER) == 0) {
               fusain_decode_from_buf(buf, &msg);
               net_buf_unref(buf);
           }
       }
   }


Framing Layer Types
-------------------

The framing layer (outside CBOR) uses fixed-size types:

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Field
     - Encoding
   * - ADDRESS
     - 64-bit unsigned integer, little-endian
   * - CRC
     - 16-bit unsigned integer, big-endian


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
