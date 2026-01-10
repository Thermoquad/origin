Fusain Protocol Specification
#############################

.. note::
   This documentation is under construction.

Fusain Protocol v2.0 - Serial Communication Protocol for Thermoquad Systems.

Document Status
***************

This is the normative specification for the Fusain protocol.

See :doc:`/specifications/fusain/overview` for implementation guidance.

Packet Format
*************

.. code-block:: text

   [START] [LENGTH] [TYPE] [PAYLOAD...] [CRC_L] [CRC_H] [END]

* START: 0x7E
* END: 0x7F
* LENGTH: Payload length (1 byte)
* TYPE: Message type (1 byte)
* CRC: CRC-16-CCITT (2 bytes, little-endian)
