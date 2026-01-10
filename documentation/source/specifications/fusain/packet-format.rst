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
     - Payload length in bytes (0–114)
   * - ADDRESS
     - 8 bytes
     - 64-bit device address (little-endian)
   * - MSG_TYPE
     - 1 byte
     - Message type identifier
   * - PAYLOAD
     - 0–114 bytes
     - Message-specific data
   * - CRC
     - 2 bytes
     - CRC-16-CCITT checksum (big-endian)
   * - END
     - 1 byte
     - End delimiter (``0x7F``)

**Packet Size:** 14 bytes overhead + payload = 14–128 bytes total.


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
  after a random delay (0–50ms)


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

The CRC is calculated over: LENGTH + ADDRESS + MSG_TYPE + PAYLOAD.

The START and END delimiters are NOT included in the CRC calculation.

**Byte Order**

The CRC MUST be transmitted big-endian (MSB first, then LSB). This is the only
field transmitted in big-endian order; all other multi-byte fields use
little-endian.


.. _byte-stuffing:

Byte Stuffing
*************

To prevent delimiter bytes from appearing in the packet data, byte stuffing is
applied to all bytes between START and END.

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

1. Build the packet (LENGTH, ADDRESS, MSG_TYPE, PAYLOAD)
2. Calculate CRC over the packet
3. Append CRC to the packet
4. Apply byte stuffing to the entire packet (including CRC)
5. Add START delimiter before and END delimiter after

When receiving:

1. Detect START delimiter
2. Read bytes until END delimiter, or 256 bytes (see :ref:`impl-buffers`)

   a. If buffer limit reached, discard and return to step 1

3. Apply byte unstuffing
4. Extract CRC from the unstuffed data
5. Verify CRC over LENGTH + ADDRESS + MSG_TYPE + PAYLOAD
6. Process the packet if CRC is valid


Data Types
**********

Multi-byte integers in the payload use little-endian byte order. For encoding
details and examples, see :ref:`impl-data-types` in the Implementation Guide.

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Type
     - Encoding
   * - u8
     - Unsigned 8-bit integer
   * - i8
     - Signed 8-bit integer
   * - i32
     - Signed 32-bit integer, little-endian
   * - u32
     - Unsigned 32-bit integer, little-endian
   * - u64
     - Unsigned 64-bit integer, little-endian
   * - f64
     - IEEE 754 double-precision float, little-endian
