Burn Cycle
##########

The operational cycle of a burner from startup to shutdown.

This document describes the operational behavior. For the serial protocol
commands and telemetry used to control and monitor these operations, see
:doc:`/specifications/fusain/index`.


.. _burn-cycle-ignition:

Ignition
********

When the user requests heat (via :ref:`msg-state-command` with ``mode=HEAT``),
the system begins the ignition sequence. The appliance transitions through
PREHEAT and PREHEAT_STAGE_2 states (reported in :ref:`msg-state-data`).

The motor starts first, spinning up to the ignition RPM (monitored via
:ref:`msg-motor-data`). This serves two purposes: airflow prevents the glow
plug from overheating, and verifies the motor is functioning before committing
to ignition. Most air heaters use a
single motor that drives both the combustion intake fan and :term:`heat exchanger` fan
on a common shaft. The system waits for the motor to stabilize within 5% of
target RPM. If the motor fails to reach this speed, it indicates a hardware
failure requiring maintenance.

Once airflow is established, the :term:`glow plug` lights (status reported via
:ref:`msg-glow-data`). The glow plug heats a metal mesh inside the combustion
chamber. This mesh will vaporize and ignite the fuel when it arrives.

After the glow plug warmup period, the fuel pump begins pulsing at the ignition
pump rate (events reported via :ref:`msg-pump-data`). This rich mixture improves
ignition reliability. Fuel travels through the lines, reaches the combustion
chamber, contacts the hot mesh, vaporizes, and ignites.

The system has a configured timeout to reach the combustion establishing
temperature (monitored via :ref:`msg-temperature-data`). This generous window
allows time for the fuel pump to prime empty lines on a cold start, which can
take a minute or more. If the temperature is not reached within this timeout,
ignition has failed—typically due to an empty tank, air in the fuel lines, a
blocked filter, or a faulty glow plug. The appliance reports error code
IGNITION_FAIL in :ref:`msg-state-data`.


.. _burn-cycle-preheating:

Preheating
**********

When the heat exchanger reaches the combustion establishing temperature,
combustion is starting but not yet stable (PREHEAT_STAGE_2 state in
:ref:`msg-state-data`). The system adjusts the fuel/air mixture to optimize
the burn.

Motor RPM increases to the preheating RPM, providing more airflow for complete
combustion. The :term:`pump pulse rate` decreases to the preheating rate,
creating a leaner mixture that produces less smoke and unburned fuel. The richer
mixture during ignition ensures the flame catches; the leaner mixture now
ensures it burns cleanly.

Temperature continues rising. When the heat exchanger reaches the combustion
stable temperature, combustion is self-sustaining. The glow plug extinguishes—it
is no longer needed to maintain the flame.


.. _burn-cycle-heating:

Heating
*******

With stable combustion established, the system enters normal operation (HEATING
state in :ref:`msg-state-data`).

The user controls heat output by setting the :term:`pump pulse rate` (via the
``argument`` field of :ref:`msg-state-command` with ``mode=HEAT``, or directly
via :ref:`msg-pump-command`). More fuel means more combustion, which requires
higher fan speed to maintain the target temperature. The faster fan pushes more
hot air into the heated space. A :term:`PID controller` (configured via
:ref:`msg-temperature-config`) automatically adjusts motor RPM to maintain a
constant heat exchanger temperature regardless of the pump rate.

The PID uses inverted control because the motor drives the heat exchanger fan.
More airflow over the heat exchanger removes more heat, lowering its
temperature. So when temperature rises above the target, motor RPM increases to
push more air and bring temperature back down. When temperature drops below the
target, motor RPM decreases to reduce cooling.

Motor RPM range is learned dynamically based on the heater's characteristics.

If the PID cannot maintain temperature, additional protection engages. At the
pump reduction threshold, the fuel pump rate is reduced by 50% until temperature
drops back to the pump recovery threshold. At the pump disable threshold, the
fuel pump is disabled entirely until temperature drops below that threshold.
These measures reduce heat input when the fan alone cannot keep up.

The system monitors for flame-out. If temperature drops below the combustion
establishing threshold during normal heating, combustion has failed and the
system transitions to an error state (error code FLAME_OUT in
:ref:`msg-state-data`).

The heater continues in this state until the user requests shutdown.


.. _burn-cycle-cooldown:

Cooldown
********

When the user requests shutdown (via :ref:`msg-state-command` with ``mode=IDLE``),
the fuel pump stops immediately but the motor keeps running (COOLING state in
:ref:`msg-state-data`). The combustion chamber is hot and contains residual fuel
that must be handled safely.

While the heat exchanger is above the cooldown glow start temperature, the motor
runs at the cooldown RPM to dissipate heat. The glow plug remains off during
this phase.

Between the cooldown glow start and cooldown complete temperatures, the glow
plug lights again. This burns off residual fuel and carbon deposits in the
combustion chamber. Without this step, unburned fuel would form carbon deposits
as the chamber cools, eventually clogging the system.

Once the heat exchanger drops below the cooldown complete temperature, the glow
plug extinguishes. The motor continues running for the fan cooldown period to
fully cool the system before stopping. After this timer completes, the system
returns to Idle.

The heater should remain powered until the glow plug extinguishes. For safety,
wait for the full fan cooldown to complete before moving or unplugging the
heater.

.. note::

   If the heater is being automatically controlled, its power source or relay
   should not be disconnected until the glow plug extinguishes.

This cooldown sequence runs whenever the heater has reached operating
temperature, even after an error condition.


Fault Conditions
****************

Two abnormal conditions can interrupt the burn cycle.

**Ignition Failure**

If the heat exchanger does not reach the combustion establishing temperature
within the ignition timeout, ignition has failed. If the temperature rose above
the cooldown complete threshold, the system performs the cooldown sequence
before entering the error state. The user should check the fuel supply, prime
the lines if necessary, and clear the fault before attempting another start.

**Flame-Out**

If temperature drops below the combustion establishing threshold during normal
heating, combustion has been lost. The system performs the normal cooldown
sequence (if temperature is above the cooldown complete threshold) before
entering the error state.


Related Specifications
**********************

- :doc:`/specifications/fuel-profiles/index` — Temperature thresholds and timing
  parameters for different fuel types
- :ref:`Emergency Stop Behavior <impl-estop>` — Protocol details for E_STOP state
