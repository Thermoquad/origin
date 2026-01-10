Overview
########

:term:`Fusain` is the serial communication protocol used between Thermoquad devices.

Fusain Protocol v2.0 provides:

* Reliable packet framing with CRC-16-CCITT
* Byte stuffing for data transparency
* Telemetry bundles for efficient data transfer
* Command/response messaging


.. _fusain-device-roles:

Device Roles
************

Fusain defines several device roles that participate in communication.


Primary Roles
-------------

These are the fundamental device types in a Fusain network.

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Role
     - Description
   * - **Controller**
     - Sends command packets, receives data packets. Examples: UI controller,
       automation system, phone app.
   * - **Appliance**
     - Receives command packets, sends data packets. Examples: :term:`Helios` ICU,
       burner control unit.
   * - **Monitor**
     - Receives packets but does not send commands or data. Examples:
       display-only device, data logger, diagnostic tool.


Extended Roles
--------------

These roles describe additional functions a device may perform.

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Role
     - Description
   * - **Router**
     - A controller that forwards packets between physical layers. Routes
       commands from remote controllers to appliances, and forwards telemetry
       back to subscribers. See :doc:`packet-routing`.
   * - **Client**
     - A device responsible for user interactions, displaying information and
       sending control commands. A thermostat fulfills the client role.
       :term:`Luna` is a client device, :term:`Roastee` is a client application.

A single device may fulfill multiple roles. For example, :term:`Slate` acts as both a
controller (sending commands to Helios) and a router (forwarding commands from
remote clients).


Data Types
**********

Fusain documentation uses two levels of type abstraction:

- **High-level types** (:doc:`messages`) — Semantic types as decoded in
  implementation structs
- **Wire-level types** (:doc:`packet-payloads`) — Binary encoding on the wire

Type Mapping
------------

.. list-table::
   :header-rows: 1
   :widths: 20 25 55

   * - High-Level
     - Wire-Level
     - Notes
   * - bool
     - u32
     - 0 = false, non-zero = true.
   * - int
     - u8, i32, u32
     - Signed or unsigned depending on field semantics. Small counts (1-255)
       may use u8.
   * - decimal
     - f64
     - IEEE 754 double-precision float.
   * - enum
     - u8, u32
     - Integer with defined value set. Size varies by field.

For the exact wire-level type of each field, see :doc:`packet-payloads`. All
multi-byte values use little-endian byte order.
