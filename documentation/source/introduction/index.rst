.. _introducing_thermoquad:

Introduction
############

Thermoquad is an open source ecosystem for Chinese diesel heaters. Think of it
like the open source parts of Android, but for heating systems.

Chinese diesel heaters are affordable and widely available, but their stock
control systems are limited, proprietary, and often incompatible between
manufacturers. Thermoquad provides open hardware designs, open firmware, and
open protocols that anyone can use, modify, and improve.

The ecosystem includes firmware for ignition control and thermostat management,
hardware designs for controllers and interfaces, and software for monitoring
and integration. All components communicate using the Fusain protocol, an open
serial communication standard designed for reliability.


Who is Thermoquad For?
**********************

Thermoquad is designed to serve four distinct groups of users:

**Tuners & Experimenters**
   For those who want to push their heater beyond stock capabilities—adjusting
   fuel/air ratios for cleaner burns, extending temperature ranges, fine-tuning
   startup sequences, or experimenting with different fuel types. Thermoquad
   provides full control over every parameter with real-time telemetry.

   *Your setup:* Replace your stock control board with Hades running Helios.
   Add a Block for wireless connectivity. Use Roastee on your laptop to monitor
   live telemetry, adjust parameters, and log data while you experiment.

**Users Who Want a Better Experience**
   For those who just want their heater to work well. Stock controllers are
   clunky, displays are hard to read, and voice prompts with strange accents
   are annoying. Thermoquad provides clear temperature displays, intuitive
   operation, and quiet, unobtrusive feedback.

   *Your setup:* Keep your existing heater and wiring. Add a Block that
   connects via LIN using your stock cables. Control everything with Luna's
   simple 3-button OLED interface—perfect for adjusting temperature from your
   sleeping bag. Or use Roastee on your phone when you want more detail.

**Repairers & DIYers**
   For those whose ICU died and can't find a compatible replacement, or whose
   controller stopped talking to their heater. Thermoquad solves the
   compatibility nightmare with one ICU design that works with common heater
   hardware, open documentation for diagnosing problems, standard connectors
   and pinouts, and detailed logs for maintenance tracking.

   *Your setup:* Replace your dead stock board with Hades. Pair it with a Block
   and you're back in business—no hunting for obscure replacement parts. Use
   Roastee to check runtime hours, error history, and maintenance intervals.

**Integrators & Builders**
   For those building something bigger—a van, boat, workshop, or off-grid
   cabin—who need multiple heaters working together, integration with
   automation systems, custom damper and zone control, and remote monitoring.
   The open Fusain protocol makes integration straightforward.

   *Your setup:* Install Hades boards in each heater, connected via RS-485 to a
   central Block. Use Roastee or Home Assistant to orchestrate zone
   temperatures, schedules, and coordinated operation. Wall-mount a Luna for
   quick manual control. Roastee connects over WiFi for remote monitoring.


Project Components
******************

**Firmware**

*Helios*
   The Ignition Control Unit firmware—the brain that controls your heater.
   Runs on Hades hardware, or a Raspberry Pi Pico 2 for prototyping.

*Slate*
   The thermostat and control hub firmware. Manages temperature regulation and
   serves as the integration point for automation systems. Runs on Block and
   Luna hardware, or a Raspberry Pi Pico 2 for prototyping.

**Hardware**

All Thermoquad hardware is based on the RP2350.

*Hades*
   The ICU board that runs Helios. Connects to heater components and replaces
   the stock control board.

*Block (working name)*
   A wireless bridge that runs Slate. Features LIN for retrofitting heaters
   using stock wiring, RS-485 for new builds or customized rebuilds, WiFi and
   Bluetooth for user connectivity, and 4 RGB status LEDs. Designed to be IP68
   compliant.

*Luna*
   A handheld interface with an OLED display and 3 buttons. Battery powered and
   IP68 compliant, designed to be easy to use in dark settings like a tent. Can
   also be wall mounted in hardwired installations, requiring only power
   (5V–36V). Connects wirelessly to a Block.

**Software**

*Roastee*
   A progressive web app for desktops and phones. Connects to Blocks and Lunas
   via WiFi or Bluetooth for remote monitoring and control.

*Heliostat*
   A development tool for monitoring and debugging the Fusain protocol.

**Protocol**

*Fusain*
   The communication protocol that ties everything together. Open, documented,
   and designed for extensibility. See :doc:`/specifications/fusain/overview`.


Project Goals
*************

**Open**
   All designs, firmware, and protocols are open source. Use them, modify them,
   share them.

**Compatible**
   Work with existing heater hardware. You shouldn't need to buy a new heater
   or run new wires to use Thermoquad.

**Documented**
   Good documentation is as important as good code. If it's not documented, it
   doesn't exist.


Getting Started
***************

* :doc:`/getting-started/index` — Set up your development environment
* :doc:`/firmware/helios/index` — Learn about the Helios ICU firmware
* :doc:`/specifications/fusain/overview` — Understand the communication protocol
* :doc:`/hardware/index` — Hardware specifications and pinouts
