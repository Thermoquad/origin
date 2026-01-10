Helios ICU
##########

Helios is the Ignition Control Unit firmware for liquid fuel burner control.

.. note::
   This documentation is under construction.

Overview
********

Helios provides:

* Burner state machine management
* Temperature PID control
* Motor, pump, and glow plug control
* Serial telemetry via Fusain protocol
* Safety interlocks and fault detection

Architecture
************

Helios uses Zephyr's Zbus for inter-module communication:

* **State Machine** - Main operational logic
* **Temperature Controller** - PID-based temperature regulation
* **Motor Controller** - Combustion air blower control
* **Pump Controller** - Fuel pump control
* **Glow Controller** - Igniter control
* **Serial Handler** - Fusain protocol communication
