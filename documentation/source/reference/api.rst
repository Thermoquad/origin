API Reference
#############

.. note::
   This documentation is under construction.

Fusain Library API
******************

Header: ``<fusain/fusain.h>``

Functions
=========

``fusain_encode()``
   Encode a message into a packet.

``fusain_decode()``
   Decode a packet into a message.

``fusain_crc16()``
   Calculate CRC-16-CCITT checksum.

Types
=====

``struct fusain_packet``
   Decoded packet structure.

``enum fusain_msg_type``
   Message type enumeration.
