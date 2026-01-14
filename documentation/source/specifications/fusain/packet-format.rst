Packet Format
#############

:term:`Fusain` uses a binary packet format with delimiters, addressing, and error
detection.


Structure
*********

All packets follow this structure:

.. list-table::
   :header-rows: 1
   :widths: 15 15 70

   * - Field
     - Size
     - Description
   * - START
     - 1 byte
     - Start delimiter (``0x7E``)
   * - LENGTH
     - 1 byte
     - Payload length in bytes (0-114)
   * - ADDRESS
     - 8 bytes
     - 64-bit device address (little-endian)
   * - PAYLOAD
     - 0-114 bytes
     - CBOR-encoded message (includes type and data)
   * - CRC
     - 2 bytes
     - CRC-16-CCITT checksum (big-endian)
   * - END
     - 1 byte
     - End delimiter (``0x7F``)

**Packet Size:** 13 bytes overhead + payload = 13-127 bytes total (unstuffed).

**Wire Format Size:** Byte stuffing expands packets on the wire:

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - Calculation
     - Size
   * - Unstuffed content (LENGTH + ADDRESS + PAYLOAD + CRC)
     - 1 + 8 + 114 + 2 = 125 bytes max
   * - Worst-case stuffing (every byte escaped)
     - 125 × 2 = 250 bytes
   * - Add START and END delimiters
     - 250 + 2 = 252 bytes
   * - **Maximum wire size**
     - **252 bytes (2,016 bits)**

For buffer sizing, see :ref:`impl-buffers`.

**Payload Structure:** The CBOR payload is encoded as a two-element array
``[type, data]`` where ``type`` is the message type identifier (see
:doc:`messages`) and ``data`` is a CBOR map with message-specific fields
(see :doc:`packet-payloads`).


Addressing
**********

The ADDRESS field identifies the source or destination device.

**Command packets** (:ref:`controller <fusain-device-roles>` to :ref:`appliance <fusain-device-roles>`): ADDRESS is the destination.

**Data packets** (appliance to controller): ADDRESS is the source.

**Special Addresses**

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Address
     - Purpose
   * - ``0x0000000000000000``
     - Broadcast (all devices)
   * - ``0xFFFFFFFFFFFFFFFF``
     - Stateless (routers, subscriptions)

**Address Assignment**

Addresses SHOULD be globally unique. Appliances MAY use a MAC address, serial
number, or other hardware identifier. Appliances MUST include their own address
in data packets. Appliances MUST silently ignore packets addressed to other
devices.

**Broadcast Behavior**

When an appliance receives a broadcast command:

- The command MUST be processed normally
- The appliance MUST NOT respond (to prevent bus collisions)
- Exception: :ref:`DISCOVERY_REQUEST <msg-discovery-request>` triggers a response
  after a random delay (0-50ms)


.. _packet-crc:

CRC Calculation
***************

Fusain uses CRC-16-CCITT for error detection.

**Algorithm Parameters**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Parameter
     - Value
   * - Polynomial
     - ``0x1021``
   * - Initial value
     - ``0xFFFF``
   * - XOR out
     - ``0x0000``
   * - Reflect in
     - False
   * - Reflect out
     - False

**Coverage**

The CRC is calculated over: LENGTH + ADDRESS + PAYLOAD.

The START and END delimiters are NOT included in the CRC calculation.

**Byte Order**

The CRC MUST be transmitted big-endian (MSB first, then LSB).


Byte Order Rationale
====================

The framing layer uses two byte orders:

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - Field
     - Byte Order
     - Rationale
   * - ADDRESS
     - Little-endian
     - Native byte order for ARM Cortex-M and x86 processors
   * - CRC
     - Big-endian
     - Historical convention from CRC-16-CCITT telecommunications usage

**Why little-endian for ADDRESS?**

The target platforms (RP2350 ARM Cortex-M33, x86 desktop tools) are little-endian.
Using native byte order allows direct memory access without byte-swapping:

.. code-block:: c

   // Direct read - no conversion needed on little-endian hardware
   uint64_t address = *(uint64_t*)&packet->address;

**Why big-endian for CRC?**

CRC-16-CCITT originated in ITU-T telecommunications standards (X.25, HDLC) where
network byte order (big-endian) was the convention. Many protocols follow this:

- HDLC, SDLC - Big-endian CRC
- PPP Frame Check Sequence - Big-endian CRC
- Kermit, XMODEM - Big-endian CRC

Maintaining this convention ensures compatibility with existing CRC implementations
and aligns with developer expectations for CRC-16-CCITT.


CRC Selection Rationale
=======================

Fusain uses CRC-16-CCITT for error detection. This section documents the rationale
for this choice.

**Hamming Distance Performance**

Based on `Koopman's CRC research <https://users.ece.cmu.edu/~koopman/crc/>`_:

.. list-table::
   :header-rows: 1
   :widths: 25 30 45

   * - Hamming Distance
     - Max Data Length
     - Error Detection
   * - HD=4
     - 32,751 bits (~4KB)
     - All 1, 2, 3-bit errors
   * - HD=5
     - 241 bits (~30 bytes)
     - All 1-4 bit errors
   * - HD=6
     - 135 bits (~17 bytes)
     - All 1-5 bit errors

For Fusain packets (max 252 bytes = 2,016 bits on wire), CRC-16-CCITT provides:

- **HD=4**: Detects all 1, 2, and 3-bit errors
- Burst error detection: Any burst ≤16 bits
- Undetected error probability: ~1/65,536 for random errors

**Why CRC-16-CCITT Is Appropriate**

1. **Sufficient for Packet Sizes**: Fusain packets are well under the 32,751-bit
   HD=4 limit.

2. **Industry Standard**: CRC-16-CCITT is widely used in embedded protocols
   (X.25, HDLC, Bluetooth, PPP).

3. **Zephyr Native Support**: Zephyr provides optimized implementations via
   ``crc16_itu_t()``. See :ref:`impl-zephyr` for integration details.

4. **Low Overhead**: 2 bytes per packet is acceptable for our message sizes.

**Alternative Consideration**

For applications requiring stronger error detection, CRC-32K/4.2 (Koopman
polynomial 0x93A409EB) provides HD=6 at packet sizes up to ~770 bytes. This
could be adopted as a future enhancement if needed for noisier environments
or safety certification.

**References**

- `Koopman CRC Zoo <https://users.ece.cmu.edu/~koopman/crc/>`_
- `CRC Polynomial Selection for Embedded Networks <https://users.ece.cmu.edu/~koopman/roses/dsn04/koopman04_crc_poly_embedded.pdf>`_


.. _byte-stuffing:

Byte Stuffing
*************

To prevent delimiter bytes from appearing in the packet data, byte stuffing is
applied to the packet content (LENGTH, ADDRESS, PAYLOAD, and CRC) before the
START and END delimiters are added. The delimiters themselves MUST NOT be escaped.

**Escape Sequences**

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - Byte
     - Escaped As
     - Purpose
   * - ``0x7E``
     - ``0x7D 0x5E``
     - Escape START delimiter
   * - ``0x7F``
     - ``0x7D 0x5F``
     - Escape END delimiter
   * - ``0x7D``
     - ``0x7D 0x5D``
     - Escape the escape byte

**Processing Order**

When transmitting:

1. Encode the message as CBOR (type + data map)
2. Build the packet (LENGTH, ADDRESS, CBOR payload)
3. Calculate CRC over the packet
4. Append CRC to the packet
5. Apply byte stuffing to the entire packet (including CRC)
6. Add START delimiter before and END delimiter after

When receiving:

1. Detect START delimiter
2. Read bytes until END delimiter, or 256 bytes (see :ref:`impl-buffers`)

   a. If buffer limit reached, discard and return to step 1

3. Apply byte unstuffing
4. Extract CRC from the unstuffed data
5. Verify CRC over LENGTH + ADDRESS + PAYLOAD
6. Decode CBOR payload to extract message type and data
7. Process the message if CRC is valid


Data Types
**********

Payloads use CBOR encoding. The canonical schema is defined in ``fusain.cddl``.
For implementation details, see :doc:`implementation`. For communication
patterns and telemetry, see :doc:`communication-patterns`.

**Framing Layer Types**

These fixed-size types are used in the packet framing (not CBOR-encoded):

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Type
     - Encoding
   * - ADDRESS
     - Unsigned 64-bit integer, little-endian
   * - CRC
     - Unsigned 16-bit integer, big-endian

**CBOR Payload Types**

Message payloads use CBOR's self-describing encoding:

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Type
     - CBOR Encoding
   * - uint
     - Variable-length unsigned integer (1-9 bytes depending on value)
   * - int
     - Variable-length signed integer
   * - float
     - IEEE 754 float (half, single, or double precision)
   * - bool
     - Single byte: ``0xF4`` (false) or ``0xF5`` (true)
   * - tstr
     - UTF-8 text string with length prefix
   * - nil
     - Single byte: ``0xF6`` (null/empty payload)
