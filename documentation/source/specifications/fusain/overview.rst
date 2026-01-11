.. _fusain-protocol:

Overview
########

:term:`Fusain` is the serial communication protocol used between Thermoquad devices.
For message definitions, see :doc:`messages`. For implementation guidance, see
:doc:`implementation`.

Fusain provides:

* Reliable packet framing with CRC-16-CCITT
* Byte stuffing for data transparency
* CBOR-encoded payloads with schema-driven validation
* Telemetry bundles for efficient data transfer
* Command/response messaging
* Optional fields without padding bytes


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

Fusain payloads use CBOR (Concise Binary Object Representation) encoding with
a CDDL schema for validation. The schema is defined in ``fusain.cddl``. For
zcbor integration in Zephyr, see :ref:`impl-cbor`.

Type Mapping
------------

.. list-table::
   :header-rows: 1
   :widths: 20 25 55

   * - High-Level
     - CBOR Type
     - Notes
   * - bool
     - CBOR bool
     - ``true`` (0xF5) or ``false`` (0xF4)
   * - int
     - CBOR uint/int
     - Variable-length encoding. Small values (0-23) encode in 1 byte.
       Wire-level docs (:doc:`packet-payloads`, CDDL) specify ``uint`` or
       ``int`` based on signedness.
   * - float
     - CBOR float
     - IEEE 754 single or double precision.
   * - decimal
     - CBOR float
     - Alias for float. Used in :doc:`messages` for PID gains and temperatures.
   * - timestamp
     - CBOR uint
     - 32-bit milliseconds since boot.
   * - enum
     - CBOR uint
     - Integer with defined value set per CDDL schema.

For payload structure details, see :doc:`packet-payloads`. CBOR uses its own
byte ordering (big-endian for multi-byte integers in the encoding).


Implementation Notes
********************

Thermistor Support
------------------

The Fusain protocol transmits temperature values in degrees Celsius. The
conversion from raw ADC readings to temperature is firmware-specific and not
defined by the protocol. Implementations may support different thermistor
types by using appropriate lookup tables or Steinhart-Hart coefficients.

ADC filtering, smoothing, and sample averaging are also implementation details
left to firmware. The protocol only specifies the temperature value format
(CBOR float) and PID controller gains.

.. note::

   Protocol messages for configuring thermistor parameters (lookup tables,
   Steinhart-Hart coefficients) are planned for future expansion.
