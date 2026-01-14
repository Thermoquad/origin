TCP Transport
#############

This document specifies how to send and receive :term:`Fusain` packets over TCP and
TCP/TLS connections.


Overview
********

TCP provides a reliable, stream-based transport layer for Fusain communication
between :ref:`controllers <fusain-device-roles>`, gateways, and remote applications.

**Use Cases:**

- Controller-to-gateway communication
- Inter-controller routing
- Integration with backend services


Transport Format
****************

Fusain packets are transmitted as a continuous byte stream over TCP.


Stream Format
-------------

Fusain packets are transmitted in their complete wire format:

- START delimiter (``0x7E``)
- Byte-stuffed content (LENGTH, ADDRESS, MSG_TYPE, PAYLOAD, CRC)
- END delimiter (``0x7F``)

For packet structure and encoding, see :doc:`packet-format`.


Message Framing
---------------

Unlike WebSocket, TCP provides no built-in message boundaries. Receivers MUST
parse the byte stream to identify packet boundaries using START (``0x7E``) and
END (``0x7F``) delimiters.

The byte stuffing mechanism (see :ref:`byte-stuffing`) ensures delimiters only
appear at packet boundaries, enabling reliable framing.


Stream Reassembly
-----------------

TCP may deliver data in arbitrary chunks. Implementations MUST:

1. Buffer incoming bytes until a complete packet is received
2. Handle packets split across multiple ``recv()`` calls
3. Handle multiple packets delivered in a single ``recv()`` call

For buffer sizing, see :ref:`impl-buffers`.


Connection Handling
*******************

Port Assignment
---------------

The default port for Fusain over TCP is implementation-defined. Recommended
ports:

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - Port
     - Protocol
     - Use Case
   * - 4850
     - TCP
     - Unencrypted (development only)
   * - 4851
     - TCP/TLS
     - Encrypted (production)


Connection Establishment
------------------------

1. :ref:`Client <fusain-device-roles>` initiates TCP connection to gateway
2. For TLS, perform TLS handshake after TCP connection
3. Client sends :ref:`DISCOVERY_REQUEST <msg-discovery-request>` to initiate session
4. Gateway responds with :ref:`DEVICE_ANNOUNCE <msg-device-announce>` for each
   accessible appliance
5. Begin normal Fusain communication

For session initiation details, see :ref:`session-initiation` in Packet Routing.


Heartbeat
---------

Clients SHOULD send :ref:`PING_REQUEST <msg-ping-request>` packets periodically to:

- Maintain router subscription timeouts (routers remove subscriptions after
  60 seconds without :ref:`PING_REQUEST <msg-ping-request>`)
- Detect connection failures (TCP keepalive may be insufficient)
- Verify connectivity to the gateway

RECOMMENDED interval: 10-15 seconds.

.. note::

   Client :ref:`PING_REQUEST <msg-ping-request>` maintains router subscription
   timeouts only. Routers handle appliance communication timeouts independently.
   See :doc:`packet-routing` for details on ping behavior in routed topologies.


TCP Keepalive
-------------

Enable TCP keepalive as a secondary failure detection mechanism:

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - Parameter
     - Recommended Value
   * - Keepalive time
     - 60 seconds
   * - Keepalive interval
     - 10 seconds
   * - Keepalive probes
     - 3

TCP keepalive detects connection failures at the transport layer but does not
reset application-layer timeouts. :ref:`PING_REQUEST <msg-ping-request>` is still
required.


TLS Configuration
*****************

Production deployments MUST use TLS for encryption and authentication.


Protocol Version
----------------

- Minimum: TLS 1.2
- Recommended: TLS 1.3


Certificate Validation
----------------------

Clients MUST validate server certificates. Self-signed certificates MAY be used
with certificate pinning.


Cipher Suites
-------------

Use modern cipher suites with forward secrecy:

- TLS 1.3: TLS_AES_256_GCM_SHA384, TLS_CHACHA20_POLY1305_SHA256
- TLS 1.2: ECDHE-RSA-AES256-GCM-SHA384, ECDHE-RSA-CHACHA20-POLY1305


Client Authentication
---------------------

Gateways MAY require client certificates for mutual TLS (mTLS). This provides
strong authentication for controller-to-gateway connections.

Client identity from certificates (Subject DN or fingerprint) MAY be used for
appliance filtering during session initiation. See :ref:`session-initiation`
for details.


Gateway Implementation
**********************

Server Requirements
-------------------

TCP gateways that bridge to serial Fusain devices MUST:

1. Accept TCP connections on configured port
2. Parse incoming byte stream to extract Fusain packets
3. Forward complete Fusain packets without modification
4. Handle multiple concurrent TCP clients
5. Implement connection-based routing (see :doc:`packet-routing`)


Stream Processing
-----------------

The gateway maintains a receive buffer per connection:

.. code-block:: text

   TCP Client                      Gateway                   Serial Device
        |                            |                            |
        |-- TCP stream (bytes) ----->|                            |
        |                            |-- Parse packets            |
        |                            |-- Serial bytes ----------->|
        |                            |                            |
        |                            |<-- Serial bytes -----------|
        |                            |-- Encode packet            |
        |<-- TCP stream (bytes) -----|                            |


Multi-Client Support
--------------------

When multiple TCP clients connect to the same gateway:

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
- Discard the malformed packet
- Continue parsing from next START delimiter
- Do NOT close the TCP connection


Connection Errors
-----------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Error
     - Action
   * - Connection reset
     - Close connection, attempt reconnect with backoff
   * - Connection timeout
     - Close connection, attempt reconnect with backoff
   * - TLS handshake failure
     - Log error, do not retry immediately


Reconnection
------------

Clients SHOULD implement automatic reconnection with exponential backoff:

- Initial delay: 1 second
- Maximum delay: 30 seconds
- Backoff multiplier: 2x


Comparison with WebSocket
*************************

.. list-table::
   :header-rows: 1
   :widths: 25 35 40

   * - Feature
     - TCP
     - WebSocket
   * - Message framing
     - Manual (delimiter-based)
     - Automatic (frame boundaries)
   * - Browser support
     - No
     - Yes
   * - Proxy traversal
     - Limited
     - Good (HTTP upgrade)
   * - Overhead
     - Lower
     - Higher (frame headers)
   * - Typical use
     - Backend services
     - Web applications

Use TCP for server-to-server communication and backend integration. Use
WebSocket for browser-based applications.
