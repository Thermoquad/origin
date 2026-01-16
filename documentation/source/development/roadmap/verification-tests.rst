Helios Verification Tests
#########################

These tests verify Helios behavior using Stan's simulated thermistor signals
before connecting to real heater hardware.

Prerequisites
*************

- Stan firmware with verification support (`stan#6 <https://github.com/Thermoquad/stan/issues/6>`_)
- Helios connected to test stand
- Heliostat in controller mode

Test Cases
**********

Test 1: Normal Burn Cycle
=========================

**Requirement:** Helios MUST complete a normal burn cycle from start to shutdown
without errors when temperature is simulated accurately.

**Procedure:**

1. Start Helios in IDLE state
2. Send STATE_COMMAND to enter FAN mode
3. Simulate temperature progression through preheat → ignition → heating
4. Send STATE_COMMAND to initiate shutdown
5. Verify Helios completes cooldown and returns to IDLE

**Pass Criteria:** No errors, all state transitions complete successfully.

**Notes:** No minimum time requirement for heating state.

Test 2: Emergency Stop
======================

**Requirement:** Helios MUST pass a controller-initiated emergency stop test
after reaching the heating state.

**Procedure:**

1. Start simulated burn cycle
2. Wait for Helios to enter HEATING state
3. Send EMERGENCY_STOP command
4. Verify Helios immediately transitions to emergency shutdown

**Pass Criteria:** Helios stops fuel delivery and enters cooldown immediately.

Test 3: Ping Timeout
====================

**Requirement:** Helios MUST begin cooldown after entering heating state, then
idle after ping request timeout.

**Procedure:**

1. Start simulated burn cycle
2. Wait for Helios to enter HEATING state
3. Stop sending PING_REQUEST packets
4. Wait for ping timeout (30 seconds)
5. Verify Helios begins cooldown
6. Simulate appropriate temperature decrease
7. Verify Helios returns to IDLE

**Pass Criteria:** Helios autonomously initiates cooldown after ping timeout.

Test 4: Flame-Out
=================

**Requirement:** Helios MUST pass a simulated flame-out test.

**Procedure:**

1. Start simulated burn cycle
2. Wait for Helios to enter HEATING state
3. Simulate rapid temperature drop (flame-out condition)
4. Verify Helios detects flame-out and responds appropriately

**Pass Criteria:** Helios detects flame-out and initiates appropriate response.

Test 5: Overheat Recovery
=========================

**Requirement:** Helios MUST pass a simulated overheat condition where the
temperature returns to normal.

**Procedure:**

1. Start simulated burn cycle
2. Wait for Helios to enter HEATING state
3. Simulate temperature exceeding overheat threshold
4. Verify Helios responds to overheat condition
5. Simulate temperature returning to normal range
6. Verify Helios recovers and continues operation

**Pass Criteria:** Helios detects overheat, responds appropriately, and recovers
when temperature normalizes.

Test 6: Overheat Emergency Stop
===============================

**Requirement:** Helios MUST pass a simulated overheat emergency stop.

**Procedure:**

1. Start simulated burn cycle
2. Wait for Helios to enter HEATING state
3. Simulate temperature exceeding emergency stop threshold
4. Verify Helios initiates emergency shutdown

**Pass Criteria:** Helios triggers emergency stop when temperature exceeds
critical threshold.

Results Template
****************

.. list-table::
   :header-rows: 1
   :widths: 40 20 40

   * - Test
     - Result
     - Notes
   * - Test 1: Normal Burn Cycle
     - ☐ Pass / ☐ Fail
     -
   * - Test 2: Emergency Stop
     - ☐ Pass / ☐ Fail
     -
   * - Test 3: Ping Timeout
     - ☐ Pass / ☐ Fail
     -
   * - Test 4: Flame-Out
     - ☐ Pass / ☐ Fail
     -
   * - Test 5: Overheat Recovery
     - ☐ Pass / ☐ Fail
     -
   * - Test 6: Overheat Emergency Stop
     - ☐ Pass / ☐ Fail
     -
