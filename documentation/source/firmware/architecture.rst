Architecture
############

.. note::
   This documentation is under construction.

Firmware Architecture Overview
******************************

Thermoquad firmware is built on Zephyr RTOS with a modular architecture.

Platform
========

* **MCU**: Raspberry Pi Pico 2 (RP2350A / RP2354A)
* **RTOS**: Zephyr
* **Build System**: West + CMake + Taskfile

Design Principles
=================

* Modular, loosely-coupled components
* Event-driven communication via Zbus
* Safety-first design with fault detection
* Shared libraries for common functionality

Directory Structure
===================

.. code-block:: text

   Thermoquad/
   ├── apps/           # Firmware applications
   │   ├── helios/     # ICU firmware
   │   └── slate/      # Controller firmware
   ├── modules/lib/    # Shared libraries
   │   └── fusain/     # Protocol library
   └── boards/         # Custom board definitions
