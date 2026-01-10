Heliostat
#########

Heliostat is a serial protocol analyzer for Fusain protocol debugging.

.. note::
   This documentation is under construction.

Overview
********

Heliostat provides:

* Real-time packet decoding
* Error detection and statistics
* TUI interface with live updates
* Reusable Go package for protocol handling

Usage
*****

.. code-block:: bash

   # Run in TUI mode (default)
   heliostat --port /dev/ttyUSB0

   # Error detection mode
   heliostat error_detection --port /dev/ttyUSB0

   # Raw log mode
   heliostat raw_log --port /dev/ttyUSB0

Building
********

.. code-block:: bash

   cd tools/heliostat
   go build
