Diesel
######

Standard diesel fuel (#2 diesel, heating oil, red diesel).

All temperatures are measured at the :term:`heat exchanger` (exterior of burn chamber).
For burn cycle behavior, see :doc:`/specifications/burn-cycle/index`.


Temperature Thresholds
**********************

.. list-table::
   :header-rows: 1
   :widths: 40 20 40

   * - Threshold
     - Temperature
     - Purpose
   * - Combustion establishing
     - 190°C
     - Transition to leaner mixture
   * - Combustion stable
     - 200°C
     - :term:`Glow plug` off, :term:`PID controller` on
   * - Normal operation
     - 215°C
     - PID target temperature
   * - Pump reduction
     - 230°C
     - Reduce pump rate by 50%
   * - Pump recovery
     - 210°C
     - Restore normal pump rate
   * - Pump disable
     - 250°C
     - Disable pump until below threshold
   * - Flame-out detection
     - 190°C
     - Combustion failure
   * - Cooldown glow start
     - 180°C
     - Begin carbon burnoff
   * - Cooldown complete
     - 120°C
     - Glow plug extinguishes, fan cooldown timer starts
   * - Emergency stop
     - 275°C
     - Component protection


Timing and Settings
*******************

.. list-table::
   :header-rows: 1
   :widths: 50 20 30

   * - Parameter
     - Value
     - Notes
   * - Glow plug warmup
     - 60 seconds
     - Time before fuel injection
   * - Ignition timeout
     - 5 minutes
     - Maximum time to reach combustion
   * - Ignition pump rate
     - 500ms
     - Rich mixture for ignition
   * - Preheating pump rate
     - 250ms
     - Leaner mixture for stable burn
   * - Maximum pump rate
     - 5000ms
     - Slowest fuel delivery (validation limit)
   * - Ignition RPM
     - 2500
     - Initial airflow
   * - Preheating RPM
     - 2800
     - Increased airflow
   * - Cooldown RPM
     - 2500
     - Heat dissipation
   * - Fan cooldown timer
     - 5 minutes
     - Run after glow plug extinguishes
