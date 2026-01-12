Fusain Error Communication
##########################

:Date: 2026-01-12
:Author: Thermoquad
:Status: Research Complete
:Related: :doc:`/specifications/fusain/messages`

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

This document proposes an enhanced error communication scheme for Fusain protocol
error messages. The current scheme uses two error codes (1 and 2) which are
insufficient to identify specific validation failures.

**Problem:**

- ERROR_INVALID_CMD only reports "invalid parameter" (code 1) or "invalid index" (code 2)
- No way to identify which field caused the validation failure
- No way to communicate why a specific value was rejected
- ERROR_STATE_REJECT only reports current state, not rejection reason

**Solution:**

Extend error message payloads with optional fields:

- ``rejected_field`` — CBOR key of the field that caused the error
- ``constraint`` — Specific constraint that was violated
- ``rejection_reason`` — Why the state machine rejected the command

**Impact:**

- No breaking changes (new fields are optional)
- Controllers can provide better error messages to users
- Debugging and diagnostics significantly improved
- Minimal payload size increase (~2-4 bytes per error)

Background
**********

Current Error Messages
======================

The Fusain specification defines two error message types:

**ERROR_INVALID_CMD (0xE0)**

Sent when command validation fails. Current payload:

.. list-table::
   :header-rows: 1
   :widths: 10 20 70

   * - Key
     - Field
     - Description
   * - 0
     - error_code
     - 1 = Invalid parameter, 2 = Invalid device index

**ERROR_STATE_REJECT (0xE1)**

Sent when state machine rejects command. Current payload:

.. list-table::
   :header-rows: 1
   :widths: 10 20 70

   * - Key
     - Field
     - Description
   * - 0
     - error_code
     - Current state that rejected the command

Outstanding Notes
=================

The specification contains two notes indicating planned improvements:

1. **ERROR_INVALID_CMD** (messages.rst:1421-1424):

   *"A mechanism to identify which specific field caused the validation error is
   planned for future expansion."*

2. **ERROR_STATE_REJECT** (messages.rst:1459-1462):

   *"A mechanism to communicate the specific reason why the state rejected the
   command is planned for future expansion."*

Validation Failure Analysis
***************************

ERROR_INVALID_CMD Scenarios
===========================

The following validation failures all report as error code 1 (invalid parameter)
or code 2 (invalid index), with no way to distinguish between them:

**Code 1 — Invalid Parameter Value**

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Command
     - Field
     - Constraint
   * - MOTOR_CONFIG
     - pwm_period
     - Must be > 0
   * - MOTOR_CONFIG
     - pid_kp/ki/kd
     - Must not be NaN or Infinity
   * - MOTOR_CONFIG
     - max_rpm
     - Must be > min_rpm
   * - MOTOR_CONFIG
     - min_pwm_duty
     - Must be < pwm_period
   * - PUMP_CONFIG
     - pulse_ms
     - Must be > 0
   * - PUMP_CONFIG
     - recovery_ms
     - Must be > 0
   * - TEMPERATURE_CONFIG
     - pid_kp/ki/kd
     - Must not be NaN or Infinity
   * - GLOW_CONFIG
     - max_duration
     - Must be > 0
   * - STATE_COMMAND
     - mode
     - Must be 0, 1, 2, or 255 (VALUE_INVALID if out of range)
   * - STATE_COMMAND
     - argument (FAN)
     - Must be 0 or min_rpm to max_rpm
   * - STATE_COMMAND
     - argument (HEAT)
     - Must be 0 or (pulse+recovery) to max_rate
   * - MOTOR_COMMAND
     - rpm
     - Must be 0 or min_rpm to max_rpm
   * - PUMP_COMMAND
     - rate_ms
     - Must be 0 or (pulse+recovery) to max_rate
   * - GLOW_COMMAND
     - duration
     - Must be 0 to max_duration
   * - GLOW_COMMAND
     - glow (key 0)
     - Cannot light already-lit glow plug (OPERATION_BLOCKED)
   * - TEMPERATURE_COMMAND
     - type
     - Must be 0-4 (VALUE_INVALID if out of range)
   * - TEMPERATURE_COMMAND
     - target_temperature
     - Must not be NaN or Infinity
   * - SEND_TELEMETRY
     - telemetry_type
     - Must be 0-4 (VALUE_INVALID if out of range)

**Code 2 — Invalid Device Index**

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - Command
     - Field
     - Constraint
   * - MOTOR_CONFIG
     - motor
     - 0 to motor_count-1
   * - PUMP_CONFIG
     - pump
     - 0 to pump_count-1
   * - TEMPERATURE_CONFIG
     - thermometer
     - 0 to thermometer_count-1
   * - GLOW_CONFIG
     - glow
     - 0 to glow_count-1
   * - MOTOR_COMMAND
     - motor
     - 0 to motor_count-1
   * - PUMP_COMMAND
     - pump
     - 0 to pump_count-1
   * - GLOW_COMMAND
     - glow
     - 0 to glow_count-1
   * - TEMPERATURE_COMMAND
     - thermometer
     - 0 to thermometer_count-1
   * - TEMPERATURE_COMMAND
     - motor_index
     - 0 to motor_count-1 (when type=WATCH_MOTOR)
   * - SEND_TELEMETRY
     - index
     - 0 to count-1 or 0xFFFFFFFF

ERROR_STATE_REJECT Scenarios
============================

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - Command
     - Context
     - Reason
   * - GLOW_COMMAND
     - HEAT mode active
     - Glow controlled by state machine
   * - TEMPERATURE_COMMAND (SET_TARGET)
     - Not in HEATING state
     - Operation only valid in HEATING

Proposed Solution
*****************

Extended ERROR_INVALID_CMD Payload
==================================

Add optional fields to provide detailed error information:

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - error_code
     - int
     - Error category (1 or 2, unchanged)
   * - 1
     - rejected_field
     - uint
     - CBOR key of the field that failed validation (optional)
   * - 2
     - constraint
     - uint
     - Constraint violation type (optional, see values below)

**Constraint Values**

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - Value
     - Name
     - Description
   * - 0
     - UNSPECIFIED
     - No specific constraint (backward compatible)
   * - 1
     - VALUE_TOO_LOW
     - Value below minimum allowed
   * - 2
     - VALUE_TOO_HIGH
     - Value above maximum allowed
   * - 3
     - VALUE_INVALID
     - Value is NaN, Infinity, or otherwise invalid
   * - 4
     - VALUE_CONFLICT
     - Value conflicts with another field (e.g., max < min)
   * - 5
     - INDEX_NOT_FOUND
     - Device index does not exist (use with error_code 2)
   * - 6
     - FIELD_REQUIRED
     - Required field is missing
   * - 7
     - TYPE_MISMATCH
     - Field has wrong CBOR type
   * - 8
     - OPERATION_BLOCKED
     - Operation not allowed (e.g., light already-lit glow)
   * - 9
     - VALUE_IN_GAP
     - Value in invalid gap (e.g., rpm between 1 and min_rpm-1)

Extended ERROR_STATE_REJECT Payload
===================================

Add optional field to explain rejection reason:

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - error_code
     - uint
     - Current state (unchanged)
   * - 1
     - rejection_reason
     - uint
     - Why the state rejected the command (optional)

**Rejection Reason Values**

.. list-table::
   :header-rows: 1
   :widths: 10 30 60

   * - Value
     - Name
     - Description
   * - 0
     - UNSPECIFIED
     - No specific reason (backward compatible)
   * - 1
     - RESOURCE_CONTROLLED
     - Resource controlled by state machine (e.g., glow in HEAT)
   * - 2
     - INVALID_IN_STATE
     - Operation not valid in current state
   * - 3
     - TRANSITION_BLOCKED
     - State transition not allowed from current state (reserved for future use)

Wire Format Examples
********************

ERROR_INVALID_CMD Examples
==========================

**Current format (backward compatible):**

.. code-block:: text

   CBOR: [0xE0, {0: 1}]
   Meaning: Invalid parameter (unspecified)

**Extended format with field identification:**

.. code-block:: text

   CBOR: [0xE0, {0: 1, 1: 1, 2: 2}]
   Meaning: Invalid parameter, field key 1 (rpm in MOTOR_COMMAND), value too high

**Extended format for index error:**

.. code-block:: text

   CBOR: [0xE0, {0: 2, 1: 0, 2: 5}]
   Meaning: Invalid index, field key 0 (motor), index not found

ERROR_STATE_REJECT Examples
===========================

**Current format (backward compatible):**

.. code-block:: text

   CBOR: [0xE1, {0: 5}]
   Meaning: Rejected by state 5 (HEATING)

**Extended format with rejection reason:**

.. code-block:: text

   CBOR: [0xE1, {0: 5, 1: 1}]
   Meaning: Rejected by HEATING state, resource controlled by state machine

Payload Size Analysis
*********************

.. list-table::
   :header-rows: 1
   :widths: 40 20 20 20

   * - Configuration
     - Payload (bytes)
     - CBOR Overhead
     - Total
   * - Current ERROR_INVALID_CMD
     - 3
     - 2
     - 5
   * - Extended (with field)
     - 5
     - 2
     - 7
   * - Extended (with field + constraint)
     - 7
     - 2
     - 9
   * - Current ERROR_STATE_REJECT
     - 3
     - 2
     - 5
   * - Extended (with reason)
     - 5
     - 2
     - 7

**Impact:** 2-4 additional bytes per error message. Given errors are infrequent
and not time-critical, this overhead is acceptable.

Implementation Notes
********************

Backward Compatibility
======================

- New fields are **optional** — old receivers ignore unknown keys
- Old error codes (1, 2) retain their meaning
- Implementations MAY send extended errors to all receivers
- No protocol version negotiation required

Appliance Implementation
========================

When validation fails, appliances SHOULD:

1. Set ``error_code`` to appropriate category (1 or 2)
2. Set ``rejected_field`` to the CBOR key that failed (if known)
3. Set ``constraint`` to the specific violation type (if applicable)

For state rejection:

1. Set ``error_code`` to current state value
2. Set ``rejection_reason`` to explain why (if applicable)

Controller Implementation
=========================

Controllers SHOULD:

1. Always handle the base ``error_code`` (required field)
2. Use ``rejected_field`` to highlight the problematic input
3. Use ``constraint`` to provide specific error messages
4. Fall back to generic messages if extended fields are absent

Example error message generation:

.. code-block:: typescript

   function formatError(error: ErrorPayload): string {
     const { error_code, rejected_field, constraint } = error;

     if (rejected_field === undefined) {
       return error_code === 1 ? "Invalid parameter" : "Invalid device index";
     }

     const fieldName = FIELD_NAMES[rejected_field] ?? `field ${rejected_field}`;
     const constraintMsg = CONSTRAINT_MESSAGES[constraint] ?? "";

     return `${fieldName}: ${constraintMsg}`;
   }

Design Decisions
================

**Command Identification**

The error payload does not include the original message type that caused the error.
This is intentional:

- Fusain is a request-response protocol where errors are direct responses to commands
- Controllers track which command they sent and can correlate the error response
- Adding a message type field would increase payload size for minimal benefit

If a controller sends multiple commands without waiting for responses (pipelining),
it must track the order and correlate errors sequentially.

Specification Updates Required
******************************

The following specification files require updates:

1. **messages.rst**

   - Update ERROR_INVALID_CMD section with new optional fields
   - Update ERROR_STATE_REJECT section with rejection_reason
   - Remove the two "planned for future expansion" notes
   - Add constraint and rejection_reason value tables

2. **packet-payloads.rst**

   - Update ERROR_INVALID_CMD payload table
   - Update ERROR_STATE_REJECT payload table
   - Add new field descriptions

3. **fusain.cddl** (if used)

   - Update error message schemas

Consideration: OTA Update Errors
********************************

The error communication scheme could be extended to support OTA (Over-The-Air)
firmware update errors as described in the :doc:`rp2350-flash-usage` research.

OTA Error Scenarios
===================

During Fusain-based OTA transfers (particularly proxy updates where Slate relays
firmware to Helios), several error conditions could occur:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Error Scenario
     - Current Scheme Applicability
   * - Invalid firmware header
     - ERROR_INVALID_CMD with constraint VALUE_INVALID
   * - Chunk CRC mismatch
     - ERROR_INVALID_CMD with constraint VALUE_INVALID
   * - Version incompatibility
     - ERROR_INVALID_CMD with constraint VALUE_CONFLICT
   * - Flash write failure
     - Requires new error type or constraint
   * - Image validation failure
     - ERROR_INVALID_CMD with constraint VALUE_INVALID
   * - Update rejected (wrong state)
     - ERROR_STATE_REJECT with reason INVALID_IN_STATE

Potential Extensions
====================

If OTA update support is added to Fusain, the constraint values could be extended:

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - Value
     - Name
     - Description
   * - 10
     - FLASH_WRITE_FAILED
     - Flash write operation failed (hardware error)
   * - 11
     - IMAGE_TOO_LARGE
     - Firmware image exceeds available slot size
   * - 12
     - SIGNATURE_INVALID
     - Firmware signature verification failed
   * - 13
     - VERSION_DOWNGRADE
     - Firmware version older than current (if blocked)

Similarly, rejection reasons could be extended:

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - Value
     - Name
     - Description
   * - 4
     - UPDATE_IN_PROGRESS
     - Another update is already in progress
   * - 5
     - INSUFFICIENT_BATTERY
     - Battery too low for safe update

Recommendation
==============

The current error scheme provides a foundation for OTA error reporting. When OTA
message types are defined, the constraint and rejection_reason value ranges should
be reserved:

- **Constraints 0-9:** General validation errors (defined in this document)
- **Constraints 10-19:** Reserved for OTA/flash errors
- **Constraints 20-255:** Available for future use

- **Rejection reasons 0-3:** General state rejections (defined in this document)
- **Rejection reasons 4-7:** Reserved for OTA-specific rejections
- **Rejection reasons 8-255:** Available for future use

This reservation ensures the error scheme can grow to support OTA without
breaking compatibility with the base error types defined here.

Conclusion
**********

The proposed error communication scheme:

1. **Resolves outstanding specification notes** — Both TODO notes are addressed
2. **Maintains backward compatibility** — New fields are optional
3. **Enables precise error reporting** — Field-level error identification
4. **Minimal overhead** — 2-4 bytes per error message
5. **No string dependencies** — All information encoded as integers

**Recommendation:** Implement the extended error payload scheme as specified.
This enables meaningful error messages without string transmission, supporting
the constraint that "sending errors as strings is unacceptable."
