Taskfile Commands
#################

.. note::
   This documentation is under construction.

Thermoquad uses Taskfile for build automation.

Why Taskfile?
*************

* Consistent build environment
* Includes formatting and validation
* Simpler than direct West/CMake commands

Common Tasks
************

.. code-block:: bash

   # Build firmware
   task build-firmware

   # Clean and rebuild
   task rebuild-firmware

   # Flash firmware
   task flash-firmware

   # Run tests
   task test

   # Format code
   task format
