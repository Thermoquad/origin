WebSocket Transport
###################

This document specifies how to send and receive :term:`Fusain` packets over WebSocket
connections.


Overview
********

WebSocket provides a reliable, bidirectional transport layer for Fusain
communication between web applications and Fusain-enabled devices or gateways.

**Use Cases:**

- Browser-based :ref:`controller <fusain-device-roles>` applications
- Mobile web apps for device monitoring
- Remote access through WebSocket gateways


Transport Format
****************

Fusain packets are transmitted as binary WebSocket frames.


Frame Type
----------

All Fusain messages MUST be sent as binary frames (opcode ``0x02``), not text
frames.

.. code-block:: javascript

   websocket.binaryType = 'arraybuffer';


Packet Encoding
---------------

Fusain packets are transmitted in their complete wire format, including:

- START delimiter (``0x7E``)
- Byte-stuffed content (LENGTH, ADDRESS, MSG_TYPE, PAYLOAD, CRC)
- END delimiter (``0x7F``)

This preserves compatibility with serial transport and allows the same
encoding/decoding logic to be used across all transports.


Message Boundaries
------------------

Each WebSocket binary frame contains exactly one complete Fusain packet. Unlike
serial transport, WebSocket frame boundaries define packet boundaries.

Implementations SHOULD still validate the START and END delimiters to detect
malformed packets.


Connection Handling
*******************

Session Initiation
------------------

After WebSocket connection is established:

1. :ref:`Client <fusain-device-roles>` sends :ref:`DISCOVERY_REQUEST <msg-discovery-request>` to initiate session
2. Gateway responds with :ref:`DEVICE_ANNOUNCE <msg-device-announce>` for each
   accessible appliance
3. Begin normal Fusain communication

For session initiation details, see :ref:`session-initiation` in Packet Routing.


Authentication
--------------

For appliance filtering based on client identity, gateways MAY use:

- **Client certificates:** Subject DN or certificate fingerprint (preferred)
- **Authorization header:** Bearer token or other credential

When client certificates are not used, include credentials in the WebSocket
handshake:

.. code-block:: text

   GET /fusain HTTP/1.1
   Host: gateway.example.com
   Upgrade: websocket
   Connection: Upgrade
   Authorization: Bearer <token>

The gateway validates credentials during the HTTP upgrade and associates the
authenticated identity with the WebSocket connection for appliance filtering.


Heartbeat
---------

Clients SHOULD send :ref:`PING_REQUEST <msg-ping-request>` packets periodically to:

- Maintain router subscription timeouts (routers remove subscriptions after
  60 seconds without :ref:`PING_REQUEST <msg-ping-request>`)
- Maintain the connection through proxies and firewalls
- Detect connection failures

RECOMMENDED interval: 10-15 seconds.

.. note::

   Client :ref:`PING_REQUEST <msg-ping-request>` maintains router subscription
   timeouts only. Routers handle appliance communication timeouts independently.
   See :doc:`packet-routing` for details on ping behavior in routed topologies.


Reconnection
------------

Clients SHOULD implement automatic reconnection with exponential backoff.


Packet Encoding
***************

The following JavaScript implementations correspond to the algorithms specified
in :doc:`packet-format`.

CRC-16-CCITT
------------

See :ref:`packet-crc` for algorithm parameters.

.. code-block:: javascript

   function crc16ccitt(data) {
     let crc = 0xFFFF;
     for (const byte of data) {
       crc ^= byte << 8;
       for (let i = 0; i < 8; i++) {
         if (crc & 0x8000) {
           crc = (crc << 1) ^ 0x1021;
         } else {
           crc = crc << 1;
         }
       }
       crc &= 0xFFFF;
     }
     return crc;
   }


Byte Stuffing
-------------

See :ref:`byte-stuffing` for escape sequences.

.. code-block:: javascript

   const START = 0x7E;
   const END = 0x7F;
   const ESC = 0x7D;

   function byteStuff(data) {
     const result = [];
     for (const byte of data) {
       if (byte === START) {
         result.push(ESC, 0x5E);
       } else if (byte === END) {
         result.push(ESC, 0x5F);
       } else if (byte === ESC) {
         result.push(ESC, 0x5D);
       } else {
         result.push(byte);
       }
     }
     return new Uint8Array(result);
   }

   function byteUnstuff(data) {
     const result = [];
     let i = 0;
     while (i < data.length) {
       if (data[i] === ESC && i + 1 < data.length) {
         result.push(data[i + 1] ^ 0x20);
         i += 2;
       } else {
         result.push(data[i]);
         i += 1;
       }
     }
     return new Uint8Array(result);
   }


Gateway Implementation
**********************

Server Requirements
-------------------

WebSocket gateways that bridge to serial Fusain devices MUST:

1. Accept binary WebSocket frames only
2. Forward complete Fusain packets without modification
3. Handle multiple concurrent WebSocket clients
4. Implement connection-based routing (see :doc:`packet-routing`)


Packet Forwarding
-----------------

The gateway acts as a transparent bridge:

.. code-block:: text

   WebSocket Client                Gateway                 Serial Device
        |                            |                          |
        |-- Binary Frame (packet) -->|                          |
        |                            |-- Serial bytes --------->|
        |                            |                          |
        |                            |<-- Serial bytes ---------|
        |<-- Binary Frame (packet) --|                          |


Multi-Client Support
--------------------

When multiple WebSocket clients connect to the same gateway:

- Command packets from any client are forwarded to the serial bus
- Data packets from appliances are forwarded only to clients that have
  subscribed via :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` (see
  :doc:`packet-routing`)
- Clients MUST send :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` to receive
  telemetry from specific appliances


Error Handling
**************

Decode Errors
-------------

When a packet fails to decode (CRC error, invalid format):

- Log the error for debugging
- Discard the packet
- Do NOT send an error response over WebSocket

.. code-block:: javascript

   ws.onmessage = (event) => {
     try {
       const packet = decodePacket(new Uint8Array(event.data));
       handlePacket(packet);
     } catch (error) {
       console.warn('Ignoring malformed packet:', error.message);
     }
   };


Connection Errors
-----------------

Handle WebSocket errors gracefully:

.. code-block:: javascript

   ws.onerror = (error) => {
     console.error('WebSocket error:', error);
   };

   ws.onclose = (event) => {
     if (!event.wasClean) {
       console.warn('Connection lost, reconnecting...');
       scheduleReconnect();
     }
   };
