Packet Routing
##############

This document specifies multi-hop packet routing for :term:`Fusain` protocol
communication across different physical layers.


Overview
********

:ref:`Controllers <fusain-device-roles>` can act as :ref:`routers <fusain-device-roles>` to forward commands and data between other
controllers and :ref:`appliances <fusain-device-roles>` across different physical layers (WiFi, Bluetooth,
TCP ↔ LIN, RS-485). For physical layer specifications, see :doc:`physical-layer`.

**Use Case:** A remote controller wants to control or monitor an appliance
through an intermediate controller that has physical connectivity. For network
transports, see :doc:`tcp` and :doc:`websocket`.

.. code-block:: text

   Remote Controller (WiFi)
       |
       | Fusain over WiFi/BT/TCP
       |
   Router Controller (WiFi + LIN)
       |
       | Fusain over LIN
       |
   Appliance (LIN)


Routing Architecture
********************

Routers are transparent forwarders - they do not modify packet contents. The
ADDRESS field always contains the original source or destination address,
regardless of how many hops the packet traverses.


Address Field Handling
----------------------

**Command Packets (Controller → Appliance):**

- ADDRESS field contains the final destination appliance address
- Router inspects ADDRESS field to determine if forwarding is needed
- If ADDRESS ≠ router's address AND appliance is on a different physical layer,
  router forwards the packet WITHOUT modifying ADDRESS field
- Destination appliance receives packet with its own address (as expected)

**Data Packets (Appliance → Controller):**

- ADDRESS field contains the original source appliance address (sender)
- Router receives packet from appliance via physical layer
- Router checks subscription table for subscribers to this appliance
- Router forwards copy to each subscriber WITHOUT modifying ADDRESS field
- Subscriber receives packet with original appliance address (source)

**Rationale:** Preserving the ADDRESS field allows end devices to identify the
true source/destination regardless of routing hops.


.. _session-initiation:

Session Initiation
******************

When a :ref:`client <fusain-device-roles>` controller connects to a router controller, it MUST initiate the
session with a discovery handshake.


Discovery Handshake
-------------------

1. Client controller sends :ref:`msg-discovery-request` to router
2. Router responds with :ref:`msg-device-announce` for each accessible appliance
3. Router sends a final :ref:`DEVICE_ANNOUNCE <msg-device-announce>` with stateless
   address and all fields set to 0 to mark end of discovery
4. Client controller now knows which appliances are available through this router

**Accessible Appliances**

A router announces ALL appliances it can reach, including:

- Appliances on its own hardware bus (LIN, RS-485)
- Appliances learned from other routers (cascading discovery)

This enables multi-hop routing - a client connected to Router A can discover
appliances that are only reachable through Router B.

.. code-block:: text

   Client Controller              Router Controller
        |                               |
        |-- DISCOVERY_REQUEST --------->|
        |                               |
        |<-- DEVICE_ANNOUNCE (App 1) ---|
        |<-- DEVICE_ANNOUNCE (App 2) ---|
        |<-- DEVICE_ANNOUNCE (App N) ---|
        |<-- DEVICE_ANNOUNCE (end) -----|  [stateless address, all 0]
        |                               |
        |   [Session established]       |

**End-of-Discovery Marker**

The final :ref:`DEVICE_ANNOUNCE <msg-device-announce>` uses the stateless address
(``0xFFFFFFFFFFFFFFFF``) with all capability fields set to 0. This:

- Marks the end of the device list
- Allows clients to know when discovery is complete
- Is the ONLY packet sent when no appliances are available


Appliance Filtering
-------------------

For authenticated connections (TCP/TLS, WebSocket with TLS), routers MAY filter
the list of announced appliances based on client credentials.

**Credential Sources:**

- **TCP/TLS with client certificates:** Subject DN or certificate fingerprint
- **WebSocket with client certificates:** Subject DN or certificate fingerprint
- **WebSocket without client certificates:** ``Authorization`` header

This allows multi-tenant deployments where different clients have access to
different subsets of appliances.

**Implementation Notes:**

- Filtering is applied to :ref:`DEVICE_ANNOUNCE <msg-device-announce>` responses
  during discovery
- Filtered appliances are not announced to unauthorized clients
- Command forwarding SHOULD also enforce the same access control
- If a client sends a command to an unauthorized appliance, the router SHOULD
  silently drop the packet


Unauthenticated Connections
---------------------------

For unauthenticated connections (plain TCP, development environments), routers
announce all accessible appliances. This is acceptable for:

- Development and testing
- Trusted network environments
- Single-tenant deployments


Ping Behavior
*************

Ping commands maintain connection health at each hop independently. There are
two levels of ping in a routed topology:

1. **Client → Router:** Maintains subscription timeout (60 seconds)
2. **Router → Appliance:** Maintains appliance communication timeout (30 seconds)


Hardware Bus Constraint
-----------------------

A hardware bus (LIN, RS-485, plain UART) supports only ONE controller (see
:doc:`physical-layer` for bus specifications). The controller connected to the
hardware bus is responsible for maintaining the ping cycle with ALL appliances
on that bus.

In a routed topology, the router is the controller on the hardware bus and MUST
send :ref:`PING_REQUEST <msg-ping-request>` to appliances to maintain their
communication timeout.


Client Behavior
---------------

Controllers acting as clients MUST broadcast :ref:`PING_REQUEST <msg-ping-request>`
commands using the broadcast address (``0x0000000000000000``). This allows routers
to:

- Track client connection health
- Maintain subscription timeouts
- Detect client disconnection

.. note::

   Client :ref:`PING_REQUEST <msg-ping-request>` does NOT reset appliance
   communication timeouts. Routers handle ping locally and do not forward
   :ref:`PING_REQUEST <msg-ping-request>` to appliances.


Router Behavior
---------------

**Responding to Clients:**

Routers MUST respond to client :ref:`PING_REQUEST <msg-ping-request>` commands
with :ref:`PING_RESPONSE <msg-ping-response>` using the stateless address
(``0xFFFFFFFFFFFFFFFF``).

.. code-block:: text

   Client Controller              Router Controller
        |                               |
        |-- PING_REQUEST (broadcast) -->|
        |                               |
        |<-- PING_RESPONSE (router) ----|
        |                               |

**Maintaining Appliance Timeouts:**

Routers MUST send :ref:`PING_REQUEST <msg-ping-request>` to each appliance on
their hardware bus to maintain appliance communication timeouts. RECOMMENDED
interval: 10-15 seconds.

.. code-block:: text

   Router Controller              Appliance (hardware bus)
        |                               |
        |-- PING_REQUEST (addressed) -->|
        |                               |
        |<-- PING_RESPONSE (appliance) -|
        |                               |

:ref:`PING_REQUEST <msg-ping-request>` to appliances MUST use the appliance's
address (not broadcast). On multi-appliance networks, the router pings each
appliance individually.

If the router fails to send :ref:`PING_REQUEST <msg-ping-request>`, appliances
will timeout (default 30 seconds), transition to IDLE, and disable telemetry.

**Rationale:**

- The stateless address identifies responses as coming from a router, not an
  appliance
- Clients can distinguish router responses from appliance responses
- Routers do NOT forward client :ref:`PING_REQUEST <msg-ping-request>` to
  appliances (handled locally)
- Ping is handled at each hop independently

**PING_RESPONSE Forwarding:**

Routers MUST NOT forward :ref:`PING_RESPONSE <msg-ping-response>` messages
(``0x3F``) from appliances to subscribers. Ping is handled at each hop
independently.


Command Forwarding
******************

Remote controllers send commands to appliances through routers.

Forwarding Process
------------------

.. code-block:: text

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


Router Behavior
---------------

When a router receives a command (``0x10``-``0x2F``) with ADDRESS ≠ own address:

1. Determine which connection the destination device is reachable through
2. Forward packet unchanged to that connection
3. Router SHOULD maintain a device reachability table mapping appliance
   addresses to next-hop connections (hardware bus or router connection)
4. If device location is unknown, router MUST drop the packet silently

**Device Reachability Table**

Routers track how to reach each appliance:

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - Appliance Address
     - Next Hop
     - Type
   * - 0xAABBCCDDEEFF0011
     - LIN bus
     - Hardware
   * - 0x1122334455667788
     - TCP connection to Router B
     - Router

When a router receives :ref:`DEVICE_ANNOUNCE <msg-device-announce>` from another
router, it records that the announced appliance is reachable via that router
connection.


Data Subscriptions
******************

Remote controllers subscribe to telemetry from appliances through routers using
the :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` command.

.. note::

   :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` and
   :ref:`DATA_UNSUBSCRIBE <msg-data-unsubscribe>` are Controller → Controller
   commands. They are NOT sent to appliances.


DATA_SUBSCRIPTION (0x14)
------------------------

Subscribe to receive copies of ALL data messages from a specific appliance.
For message specification, see :ref:`msg-data-subscription`.

When a router receives :ref:`DATA_SUBSCRIPTION <msg-data-subscription>`:

1. Associates the subscription with the connection on which it was received
2. Adds the connection to the routing table for the specified appliance_address
3. When data messages (``0x30``-``0x34``) or error messages (``0xE0``-``0xE1``)
   are received from appliance_address, forwards those messages to the
   subscribed connection

:ref:`DEVICE_ANNOUNCE <msg-device-announce>` (``0x35``) is NOT forwarded via
subscriptions - it is only sent during discovery.

Routers MUST NOT forward packets with reserved message types
(``0x18``-``0x1E``, ``0x26``-``0x2E``, ``0x36``-``0x3E``, ``0xE2``-``0xEF``).


DATA_UNSUBSCRIBE (0x15)
-----------------------

Remove a previously established data subscription.
For message specification, see :ref:`msg-data-unsubscribe`.

When a router receives :ref:`DATA_UNSUBSCRIBE <msg-data-unsubscribe>`:

1. Identifies the connection on which the command was received
2. Removes the subscription for the specified appliance_address from that
   connection
3. No longer forwards data messages from that appliance to the connection

If no subscription exists for this appliance on the connection, the command is
silently ignored.


Data Forwarding
***************

Routers forward telemetry data from appliances to subscribed controllers.

Forwarding Process
------------------

.. code-block:: text

   1. Remote Controller sends DISCOVERY_REQUEST to Router Controller

   2. Router Controller responds with DEVICE_ANNOUNCE for each appliance,
      then end marker

   3. Remote Controller → Router Controller (via WiFi):
      MSG_TYPE: DATA_SUBSCRIPTION (0x14)
      ADDRESS: Stateless address (0xFFFFFFFFFFFFFFFF)
      appliance_address: Appliance address (from discovery)

   4. Router Controller processes subscription:
      - Adds Remote Controller to routing table for Appliance
      - Notes subscription came via WiFi physical layer

   5. Appliance sends telemetry (via LIN):
      MSG_TYPE: MOTOR_DATA, TEMPERATURE_DATA, STATE_DATA
      ADDRESS: Appliance's own address (source)

   6. Router Controller receives telemetry:
      - Processes locally if needed
      - Checks routing table: Remote Controller subscribed to Appliance data
      - Forwards copy to Remote Controller via WiFi

   7. Remote Controller receives telemetry from Appliance through Router


Routing Table
*************

Routers maintain a subscription table to track which connections want data from
which appliances.

Table Structure
---------------

.. list-table::
   :header-rows: 1
   :widths: 30 30 20 20

   * - Subscriber Connection
     - Appliance Address
     - Physical Layer
     - Last Activity
   * - WiFi connection #1
     - 0xFEDCBA0987654321
     - WiFi
     - 2026-01-05 14:23:10
   * - BT connection #2
     - 0xFEDCBA0987654321
     - Bluetooth
     - 2026-01-05 14:22:45


Connection-Based Routing
------------------------

In a multi-drop topology, there is one controller and multiple appliances on
the same bus. When a remote controller connects to a router, the connection is
point-to-point (WiFi, Bluetooth, TCP, etc.). Subscriptions are associated with
the connection, not a packet-level source address.


Table Capacity
--------------

- Routers MAY limit the number of active subscriptions (implementation-defined)
- RECOMMENDED minimum: 10 concurrent subscriptions per router


Duplicate Subscriptions
-----------------------

If the same connection already has a subscription to the same appliance, the
subscription is refreshed (timeout reset).


Subscription Lifecycle
**********************

Subscriptions are NOT persistent - they are lost on power cycle, reset, or
connection close.


Subscription Timeout
--------------------

- Router controllers SHOULD implement subscription timeout
- RECOMMENDED: 60 seconds without :ref:`PING_REQUEST <msg-ping-request>` from
  subscriber
- When timeout elapses, router removes the subscription
- This prevents stale routing table entries for disconnected controllers


Maintaining Subscriptions
-------------------------

Subscriber controllers SHOULD periodically re-send
:ref:`DATA_SUBSCRIPTION <msg-data-subscription>` to maintain routing. This ensures
subscriptions survive:

- Brief network interruptions
- Router restarts
- Timeout timer expiration


Telemetry Management
********************

In routed topologies, the controller connected to the hardware bus (the router)
is responsible for managing telemetry from appliances.


Telemetry Configuration
-----------------------

Clients connected to a router SHOULD NOT send
:ref:`TELEMETRY_CONFIG <msg-telemetry-config>` commands directly to appliances.
Instead, clients SHOULD rely on the bus controller to:

- Poll appliances for telemetry at appropriate intervals
- Configure appliance telemetry intervals via
  :ref:`TELEMETRY_CONFIG <msg-telemetry-config>`

**Rationale:**

- Only one controller can manage the hardware bus
- The router already maintains the ping cycle with appliances
- Conflicting telemetry configurations from multiple clients could cause issues
- The router can aggregate and forward telemetry to multiple subscribers


Recommended Patterns
--------------------

**Pattern 1: Router-Configured Telemetry**

The router configures appliance telemetry intervals during initialization.
Clients subscribe to receive forwarded telemetry data.

.. code-block:: text

   Router                              Appliance
      |                                    |
      |-- TELEMETRY_CONFIG (interval) ---->|
      |                                    |
      |<-- Periodic telemetry data --------|
      |                                    |
      +-- Forward to subscribed clients


**Pattern 2: Client-Requested Configuration**

Clients request the router to change telemetry settings via an
application-layer protocol. The router translates these requests into
:ref:`TELEMETRY_CONFIG <msg-telemetry-config>` commands.

.. note::

   Fusain does not define a standard message for clients to request telemetry
   configuration changes. Implementations may define application-specific
   commands for this purpose.


Implementation Requirements
***************************

Packet Inspection
-----------------

Router MUST inspect ADDRESS and MSG_TYPE fields of all received packets:

- If ADDRESS matches router's own address → process locally
- If ADDRESS is stateless address (``0xFFFFFFFFFFFFFFFF``) → process locally
- If MSG_TYPE is :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` (``0x14``) or
  :ref:`DATA_UNSUBSCRIBE <msg-data-unsubscribe>` (``0x15``) → process locally
  (these are never forwarded)
- Otherwise, if ADDRESS doesn't match → check if forwarding is needed


Command Forwarding
------------------

When router receives command (``0x10``-``0x2F``) with ADDRESS ≠ own address:

- **Exception:** :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` (``0x14``) and
  :ref:`DATA_UNSUBSCRIBE <msg-data-unsubscribe>` (``0x15``) are NEVER
  forwarded - they are processed locally regardless of ADDRESS field
- For other commands, forward packet to the next-hop connection where the
  destination device is reachable (hardware bus or upstream router)
- Router SHOULD maintain device reachability table (see Command Forwarding
  section under Router Behavior)
- If device location unknown, MUST drop the packet silently


Data Forwarding
---------------

When router receives data message (``0x30``-``0x34``) or error message
(``0xE0``-``0xE1``):

- Check subscription table for matching appliance_address
- Forward copy to each subscribed connection

**Error messages** (:ref:`ERROR_INVALID_CMD <msg-error-invalid-cmd>`,
:ref:`ERROR_STATE_REJECT <msg-error-state-reject>`) are forwarded to all
subscribers so clients can detect command failures.

**:ref:`DEVICE_ANNOUNCE <msg-device-announce>` (0x35) is NOT forwarded** via
subscriptions. It is only sent during discovery, not as ongoing telemetry.

**:ref:`PING_RESPONSE <msg-ping-response>` (0x3F) is NOT forwarded.** Routers
handle ping locally and do not forward :ref:`PING_RESPONSE <msg-ping-response>`
messages from appliances to subscribers. This ensures subscribers receive ping
responses only from the router itself.


Subscription Timeout
--------------------

Router SHOULD remove subscriptions after 60 seconds without
:ref:`PING_REQUEST <msg-ping-request>` from subscriber. This prevents stale
routing table entries.


Loop Prevention
---------------

Routers SHOULD NOT forward packets back to the physical layer they were
received from. This prevents routing loops in multi-router networks.


Example Scenario
****************

Remote burner control from a phone app:

**Setup:**

- Appliance: :term:`Helios` burner :term:`ICU` (LIN bus, address ``0xAABBCCDDEEFF0011``)
- Router: :term:`Slate` controller (WiFi + LIN, address ``0x1122334455667788``)
- Remote: Phone app with WiFi controller (address ``0x9988776655443322``)

**Connection:**

1. Phone app connects to Router via WiFi
2. Phone app sends :ref:`DISCOVERY_REQUEST <msg-discovery-request>` to Router
3. Router responds with :ref:`DEVICE_ANNOUNCE <msg-device-announce>` for Appliance,
   then end marker
4. Phone app sends :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` to Router for
   Appliance

**Control:**

1. User taps "Start Heating" in app
2. Phone app sends :ref:`STATE_COMMAND <msg-state-command>` (mode=HEAT) with
   ADDRESS=Appliance to Router
3. Router forwards command to Appliance via LIN
4. Appliance starts heating

**Monitoring:**

1. Appliance sends telemetry every 500ms to Router via LIN
2. Router checks subscription table, sees Phone app is subscribed
3. Router forwards telemetry to Phone app via WiFi
4. Phone app displays temperature, RPM, state in real-time

**Disconnection:**

1. User closes app or moves out of WiFi range
2. Router detects no :ref:`PING_REQUEST <msg-ping-request>` from Phone app for 60
   seconds
3. Router removes Phone app subscription
4. Router stops forwarding telemetry (saves WiFi bandwidth)


Multi-Hop Routing
*****************

Fusain supports multi-hop routing where packets traverse multiple routers.
Router-to-router connections are fully supported, enabling complex network
topologies.

Supported Topologies
--------------------

.. code-block:: text

   Single-Hop (Client → Router → Appliance):

   Client ----TCP/WiFi---- Router ----LIN/RS-485---- Appliance


   Multi-Hop (Client → Router → Router → Appliance):

   Client ----TCP---- Router A ----TCP---- Router B ----LIN---- Appliance


   Mesh (Multiple paths through router network):

   Client ----+
              |
          Router A ----+---- Router C ----LIN---- Appliance 1
              |        |
          Router B ----+---- Router D ----RS-485---- Appliance 2

In all topologies, each router-to-router connection follows the same protocol
as client-to-router connections (discovery handshake, subscriptions, ping).


Single-Hop Example
------------------

Controller A (WiFi) wants telemetry from Appliance C (LIN bus) through
Controller B (WiFi+LIN bridge):

1. Controller A establishes point-to-point connection to Controller B (via WiFi)

2. Controller A sends :ref:`DISCOVERY_REQUEST <msg-discovery-request>` to
   Controller B

3. Controller B responds with :ref:`DEVICE_ANNOUNCE <msg-device-announce>` for
   Appliance C, then end marker

4. Controller A sends :ref:`DATA_SUBSCRIPTION <msg-data-subscription>` to
   Controller B:

   - ADDRESS: Stateless address (0xFFFFFFFFFFFFFFFF)
   - appliance_address: Appliance C's address

5. Controller B receives subscription, associates it with the WiFi connection
   to Controller A

6. When Appliance C sends telemetry:

   - Controller B receives packet from LIN bus
   - Controller B checks routing table, sees Controller A's connection is
     subscribed
   - Controller B forwards packet to Controller A over the WiFi connection

7. Controller A receives telemetry from Appliance C through Controller B


Multi-Hop Example (Router-to-Router)
------------------------------------

Client wants telemetry from Appliance through two routers:

.. code-block:: text

   Client ----TCP---- Router A ----TCP---- Router B ----LIN---- Appliance

**Setup:**

1. Router B maintains the hardware bus with Appliance (sends
   :ref:`PING_REQUEST <msg-ping-request>`, receives telemetry)

2. Router A connects to Router B via TCP and performs discovery handshake

3. Router A subscribes to Appliance data from Router B

4. Client connects to Router A via TCP and performs discovery handshake

5. Client subscribes to Appliance data from Router A

**Data Flow:**

.. code-block:: text

   Appliance → Router B → Router A → Client
       |           |           |         |
       |  (LIN)    |  (TCP)    |  (TCP)  |
       +-----------+-----------+---------+

1. Appliance sends telemetry to Router B (via LIN)
2. Router B forwards to Router A (Router A is subscribed)
3. Router A forwards to Client (Client is subscribed)

**Command Flow:**

.. code-block:: text

   Client → Router A → Router B → Appliance
       |         |           |         |
       | (TCP)   |  (TCP)    |  (LIN)  |
       +---------+-----------+---------+

1. Client sends command with ADDRESS = Appliance address
2. Router A forwards to Router B (appliance not on local bus)
3. Router B forwards to Appliance (via LIN)

**Ping at Each Hop:**

Each connection maintains its own ping cycle independently:

- Client sends :ref:`PING_REQUEST <msg-ping-request>` to Router A (maintains
  subscription)
- Router A sends :ref:`PING_REQUEST <msg-ping-request>` to Router B (maintains
  subscription)
- Router B sends :ref:`PING_REQUEST <msg-ping-request>` to Appliance (maintains
  communication timeout)


Considerations
**************

Bandwidth
---------

Router must handle combined traffic from all subscribers and appliances.
Controller-to-controller links are expected to be high-bandwidth (WiFi,
Bluetooth, TCP). Lower-bandwidth links (e.g., LoRa) may require future protocol
extensions for filtering.


Latency
-------

Multi-hop adds latency, typically 50-200ms per hop. Plan accordingly for
time-sensitive applications.


Security
--------

Routers SHOULD validate subscriber permissions. The specific mechanism is
implementation-defined.


Scalability
-----------

Routers MAY limit maximum subscriptions. RECOMMENDED minimum: 10 concurrent
subscriptions per router.


Physical Layer Bridging
-----------------------

Router must handle different baud rates and frame sizes between physical
layers. For example, WiFi operates at high speeds while LIN operates at
19.2 kbaud with fragmented frames.
