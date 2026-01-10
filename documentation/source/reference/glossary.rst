Glossary
########

.. glossary::
   :sorted:

   Block
      A wireless bridge that runs :term:`Slate` firmware, acting as a
      :ref:`router <fusain-device-roles>` and :ref:`controller <fusain-device-roles>`
      between WiFi/Bluetooth and hardware buses. Features :term:`LIN` for
      retrofitting heaters using stock wiring, :term:`RS-485` for new builds,
      WiFi and Bluetooth for user connectivity, and 4 RGB status LEDs. Designed
      to be :term:`IP68 <IP Code>` compliant.

   Fusain
      The serial communication protocol used between Thermoquad devices. Open,
      documented, and designed for extensibility.

   Glow Plug
      A resistive heating element used to ignite fuel in diesel heaters. Draws
      approximately 10 amps and heats a metal mesh in the combustion chamber.
      Fuel vaporizes and ignites on contact with the hot mesh. Appears to be
      ceramic construction; further research is needed.

   Hades
      The :term:`ICU` board that runs :term:`Helios` firmware. Connects to heater
      components and replaces the stock control board. Based on the :term:`RP2350`.

   Heat Exchanger
      The chamber where combustion heat transfers to air blown into the heated
      space. In diesel heaters, the combustion chamber is surrounded by the heat
      exchanger. A fan blows ambient air over the heat exchanger's outer surface,
      absorbing heat and delivering warm air to the cabin or space.

   Helios
      The Ignition Control Unit firmware for burner control. Runs on :term:`Hades`
      hardware, or a Raspberry Pi Pico 2 for prototyping.

   Heliostat
      Development tool for :term:`Fusain` protocol analysis and debugging.

   ICU
      Ignition Control Unit. The controller that manages burner operation including
      ignition, combustion, and shutdown sequences.

   IP Code
      Ingress Protection Code, a classification system defined by IEC 60529 that
      rates protection against solid objects and water. Format is IP[X][Y] where
      the first digit (0-6) indicates solid particle protection and the second
      digit (0-9) indicates water protection. IP68 means dust tight with
      continuous water immersion capability. See `IP Code on Wikipedia
      <https://en.wikipedia.org/wiki/IP_code>`_.

   LIN
      Local Interconnect Network. A low-cost single-wire serial protocol used in
      automotive applications, operating at up to 20 kbit/s. Used in Thermoquad
      for retrofitting heaters with stock wiring. See `LIN on Wikipedia
      <https://en.wikipedia.org/wiki/Local_Interconnect_Network>`_.

   Luna
      A handheld :ref:`client <fusain-device-roles>` device with an OLED display
      and 3 buttons. Battery powered and :term:`IP68 <IP Code>` compliant,
      designed for use in dark settings. Can be wall mounted with hardwired
      power (5V-36V). Connects wirelessly to a :term:`Block`.

   PID Controller
      Proportional-Integral-Derivative controller. A control loop feedback
      mechanism that continuously calculates an error between a setpoint and
      measured value, adjusting output to minimize error. Used in Thermoquad
      to maintain heat exchanger temperature by adjusting fan speed. See
      `PID Controller on Wikipedia
      <https://en.wikipedia.org/wiki/Proportional–integral–derivative_controller>`_.

   Pump Pulse Rate
      How often the fuel pump receives power, expressed in milliseconds. A pump
      has a pulse duration (typically 50ms) that drives the solenoid and moves
      the piston, pumping fuel. A spring then pushes the piston back to its
      reset position during the recovery period (also typically 50ms). The
      maximum pump rate is the pulse duration plus the recovery period.

   Roastee
      A :ref:`client <fusain-device-roles>` progressive web app for desktops and
      phones. Connects to :term:`Block` and :term:`Luna` devices via WiFi or
      Bluetooth for remote monitoring and control.

   RP2350
      Raspberry Pi's dual-core Cortex-M33 microcontroller. All Thermoquad
      hardware is based on this chip.

   RS-485
      A serial communication standard defining electrical characteristics for
      balanced, multipoint systems. Uses differential signaling on two wires,
      supporting distances up to 1200 meters and speeds up to 10 Mbps. Used in
      Thermoquad for new builds and multi-heater installations. See `RS-485
      on Wikipedia <https://en.wikipedia.org/wiki/RS-485>`_.

   Slate
      The thermostat and control hub firmware. Manages temperature regulation
      and serves as the integration point for automation systems. Runs on
      :term:`Block` and :term:`Luna` hardware, or a Raspberry Pi Pico 2 for
      prototyping.

   West
      Zephyr's meta-tool for workspace management.

   Zbus
      Zephyr's publish-subscribe messaging system.
