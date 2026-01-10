Building Firmware
#################

.. note::
   This documentation is under construction.

Prerequisites
*************

* Zephyr SDK installed
* West workspace initialized
* Python venv activated

Build Commands
**************

Always use Taskfile commands for building:

.. code-block:: bash

   # Build firmware
   task build-firmware

   # Clean and rebuild
   task rebuild-firmware

   # Check firmware size
   task firmware-stats

Flashing
********

.. warning::
   Never flash firmware automatically. Always verify the build before flashing.

.. code-block:: bash

   # Flash to device (manual step)
   task flash-firmware
