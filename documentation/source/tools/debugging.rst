Debugging
#########

.. note::
   This documentation is under construction.

Debugging tools and techniques for Thermoquad development.

Serial Debugging
****************

Use Heliostat for protocol-level debugging:

.. code-block:: bash

   heliostat error_detection --port /dev/ttyUSB0

Logging
*******

Zephyr logging is configured via Kconfig:

.. code-block:: kconfig

   CONFIG_LOG=y
   CONFIG_LOG_DEFAULT_LEVEL=3

GDB Debugging
*************

.. code-block:: bash

   west debug
