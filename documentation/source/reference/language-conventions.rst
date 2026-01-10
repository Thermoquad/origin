Language Use & Conventions
##########################

This document defines the language conventions used throughout the Thermoquad
documentation.


Requirement Keywords
********************

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this documentation are
to be interpreted as described in `RFC 2119`_.

.. _RFC 2119: https://www.rfc-editor.org/rfc/rfc2119.txt


Keyword Definitions
-------------------

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Keyword
     - Meaning
   * - MUST / REQUIRED
     - Absolute requirement. Implementations that do not comply are non-conformant.
   * - MUST NOT
     - Absolute prohibition. Implementations that violate this are non-conformant.
   * - SHOULD / RECOMMENDED
     - There may be valid reasons to ignore this requirement, but implications
       must be understood and carefully weighed.
   * - SHOULD NOT / NOT RECOMMENDED
     - There may be valid reasons to allow this behavior, but implications must
       be understood and carefully weighed.
   * - MAY / OPTIONAL
     - Truly optional. Implementations may or may not include this feature.


Keyword Preference
------------------

This documentation prefers **MUST** over **SHALL** for absolute requirements.

Both terms have identical meaning per RFC 2119, but "MUST" is clearer and more
commonly understood. Use "MUST" in new documentation. Existing uses of "SHALL"
are acceptable but should be updated to "MUST" when documents are revised.

**Example:**

- Preferred: "Appliances **MUST** respond with their own address"
- Acceptable: "Appliances **SHALL** respond with their own address"


Terminology
***********

Silent Ignore
-------------

Throughout this documentation, "ignore" means "silently ignore" unless
otherwise specified. The receiver:

- Discards the packet or data
- Does NOT send any response
- Does NOT send any error message
- MAY log the event for debugging purposes


Notation Conventions
********************

Hexadecimal Values
------------------

Hexadecimal values are written with the ``0x`` prefix:

- Byte values: ``0x7E``, ``0xFF``
- Addresses: ``0x0000000000000000`` (broadcast), ``0xFFFFFFFFFFFFFFFF`` (stateless)
- Message types: ``0x10`` (MOTOR_CONFIG), ``0x30`` (STATE_DATA)


Byte Order
----------

- **Little-endian** is the default byte order for multi-byte integers
- **Big-endian** is used only for CRC-16 (explicitly noted where applicable)


Ranges
------

Ranges are expressed as:

- Inclusive: "0 to 114" means 0, 1, 2, ... 114 (115 values)
- Exclusive: "0 to 114 exclusive" means 0, 1, 2, ... 113 (114 values)

When not specified, ranges are inclusive.


Units
-----

Standard units used in this documentation:

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - Quantity
     - Unit
     - Abbreviation
   * - Time
     - milliseconds
     - ms
   * - Temperature
     - degrees Celsius
     - °C
   * - Frequency
     - revolutions per minute
     - RPM
   * - PWM period/duty
     - microseconds
     - μs
   * - Baud rate
     - bits per second
     - baud (e.g., 115200 baud)


References
**********

- `RFC 2119: Key words for use in RFCs to Indicate Requirement Levels`_
- `IEEE 754: IEEE Standard for Floating-Point Arithmetic`_

.. _RFC 2119\: Key words for use in RFCs to Indicate Requirement Levels: https://www.rfc-editor.org/rfc/rfc2119.txt
.. _IEEE 754\: IEEE Standard for Floating-Point Arithmetic: https://standards.ieee.org/standard/754-2019.html
