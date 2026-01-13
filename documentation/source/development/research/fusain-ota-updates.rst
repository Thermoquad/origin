Fusain OTA Updates
##################

:Date: 2026-01-12
:Author: Thermoquad
:Status: Research Complete
:Related: :doc:`rp2350-flash-usage`, :doc:`fusain-error-communication`

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

This document proposes Fusain protocol message types for Over-The-Air (OTA)
firmware updates. The design supports both self-update and proxy update
scenarios in multi-appliance networks.

**Key Design Decisions:**

- **Chunk-based transfer:** Firmware sent in small chunks (≤96 bytes) to fit
  within Fusain's 114-byte payload limit
- **Resumable uploads:** Hash-based deduplication allows interrupted transfers
  to resume from last successful chunk
- **MCUboot compatible:** Uses standard MCUboot image format (32-byte header,
  TLVs, SHA256 hash)
- **Multi-appliance safe:** Each appliance addressed individually; no broadcast
  OTA commands
- **Error reporting:** Integrates with extended error communication scheme

**Message Types:**

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - Message
     - Type Code
     - Purpose
   * - OTA_START
     - 0x40
     - Initiate firmware upload (size, hash, version)
   * - OTA_DATA
     - 0x41
     - Transfer firmware chunk (offset, data)
   * - OTA_VERIFY
     - 0x42
     - Request image verification
   * - OTA_ACTIVATE
     - 0x43
     - Mark image for boot (test or permanent)
   * - OTA_QUERY
     - 0x44
     - Request update status
   * - OTA_STATUS
     - 0x45
     - Report update status (response)
   * - OTA_ABORT
     - 0x4F
     - Cancel in-progress update

Background
**********

MCUboot Image Format
====================

MCUboot uses a standardized image format compatible with Fusain OTA:

**Image Header (32-byte struct, typically 0x200 padded in image):**

.. code-block:: c

   struct image_header {
       uint32_t magic;              // 0x96f3b83d
       uint32_t load_addr;          // Load address (or 0)
       uint16_t header_size;        // Header size in image (typically 0x200)
       uint16_t protect_tlv_size;   // Protected TLV size
       uint32_t image_size;         // Image size (excluding header)
       uint32_t flags;              // IMAGE_F_* flags
       struct image_version version; // Version (major.minor.rev.build)
       uint32_t _pad1;
   };

**Image Version:**

.. code-block:: c

   struct image_version {
       uint8_t  major;
       uint8_t  minor;
       uint16_t revision;
       uint32_t build_num;
   };

**TLV Types (relevant for OTA):**

- ``IMAGE_TLV_SHA256`` (0x10): SHA256 hash of image
- ``IMAGE_TLV_ECDSA_SIG`` (0x22): ECDSA signature
- ``IMAGE_TLV_KEYHASH`` (0x01): Public key hash

Flash Partition Layout
======================

From :doc:`rp2350-flash-usage`, the MCUboot partition layout on 4MB flash:

.. code-block:: none

   Address       Size    Purpose
   ──────────────────────────────────────
   0x00000000    64 KB   MCUboot bootloader
   0x00010000    832 KB  slot0_partition (primary/active)
   0x000E0000    832 KB  slot1_partition (staging/upgrade)
   0x001B0000    2.3 MB  storage_partition (NVS/littlefs)

Current Thermoquad firmware sizes:

- **Helios ICU:** ~124 KB (fits with headroom)
- **Slate Controller:** ~397 KB (fits with headroom)

Fusain Addressing
=================

Fusain uses 64-bit device addresses (see :doc:`/specifications/fusain/packet-format`):

- Each appliance has a unique address (MAC, serial number, or UUID)
- Commands include destination address
- Appliances ignore packets not addressed to them
- Broadcast (``0x0000000000000000``) is NOT used for OTA

This addressing scheme inherently supports multi-appliance networks: each
OTA transfer is addressed to a specific device.

Supported Transports
====================

OTA updates require transports that support the full Fusain payload size (114 bytes):

- **UART/Serial:** Primary transport for appliance updates (Slate → Helios)
- **WebSocket:** Primary transport for controller updates (Roastee → Slate)
- **TCP:** Supported for wired connections

**Not supported for OTA:**

- **BLE:** OTA requires payloads larger than the default BLE MTU (20 bytes).
  While extended MTU negotiation exists, support varies across devices and
  cannot be guaranteed. Use UART or WebSocket instead.

OTA Scenarios
*************

Scenario 1: Self-Update
=======================

A device receives firmware for itself over Fusain.

**Example:** Slate receives its own firmware update from Roastee via WebSocket.

.. code-block:: none

   Roastee (Web) ─── WebSocket ──→ Slate

**Flow:**

1. Roastee sends OTA_START with firmware metadata
2. Slate validates header, erases slot1
3. Roastee sends OTA_DATA chunks
4. Slate writes chunks to slot1
5. Roastee sends OTA_VERIFY
6. Slate validates image hash
7. Roastee sends OTA_ACTIVATE
8. Slate marks slot1 for boot and reboots

Scenario 2: Proxy Update (Slate → Helios)
=========================================

Slate receives Helios firmware and relays it over Fusain serial.

**Example:** Roastee sends Helios firmware to Slate, which forwards to Helios.

.. code-block:: none

   Roastee (Web) ─── WebSocket ──→ Slate ─── Fusain/UART ──→ Helios

**Flow:**

1. Roastee sends Helios firmware to Slate
2. Slate stores firmware in littlefs (``/lfs/helios_update.bin``)
3. Slate verifies complete image (hash, optionally signature)
4. Slate sends OTA_START to Helios
5. Slate sends OTA_DATA chunks from stored file
6. Slate sends OTA_VERIFY to Helios
7. Helios validates and responds with status
8. Slate sends OTA_ACTIVATE to Helios
9. Helios marks slot1 for boot and reboots

**Why buffer on Slate?**

- Allows verification before transfer
- Enables resumable transfers if interrupted
- Multiple retry attempts without re-downloading
- Decouples network latency from UART timing

Scenario 3: Multi-Appliance Update
==================================

Update multiple appliances in a network sequentially.

**Example:** Update Helios-A, then Helios-B via Slate router.

.. code-block:: none

   Roastee ──→ Slate (Router) ──→ Helios-A (0x1234...)
                              └─→ Helios-B (0x5678...)

**Constraints:**

- Only ONE appliance can be updated at a time
- Each OTA command is addressed to specific device
- Slate buffers firmware once, transfers to each appliance sequentially
- Progress tracked per-device

**Flow:**

1. Roastee uploads Helios firmware to Slate (once)
2. Slate stores in littlefs
3. Slate sends OTA commands to Helios-A (full transfer)
4. Helios-A reboots and confirms
5. Slate sends OTA commands to Helios-B (full transfer)
6. Helios-B reboots and confirms
7. Slate deletes buffered firmware

Proposed Message Types
**********************

Message Type Allocation
=======================

OTA messages use a dedicated range (``0x40-0x4F``), extending Fusain's message type
organization:

- ``0x10-0x1F``: Configuration commands
- ``0x20-0x2F``: Control commands
- ``0x30-0x3F``: Telemetry data
- ``0x40-0x4F``: **OTA messages (new)**
- ``0xE0-0xEF``: Error messages

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Type
     - Name
     - Purpose
   * - 0x40
     - OTA_START
     - Begin firmware upload
   * - 0x41
     - OTA_DATA
     - Transfer firmware chunk
   * - 0x42
     - OTA_VERIFY
     - Request image verification
   * - 0x43
     - OTA_ACTIVATE
     - Mark image for boot
   * - 0x44
     - OTA_QUERY
     - Request update status
   * - 0x45
     - OTA_STATUS
     - Report update status
   * - 0x46–0x4E
     - *Reserved*
     - Future OTA extensions
   * - 0x4F
     - OTA_ABORT
     - Cancel update

OTA_START (0x40)
================

Initiate a firmware upload session.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - size
     - uint
     - Total firmware image size in bytes
   * - 1
     - hash
     - bytes
     - SHA256 hash of complete image (32 bytes)
   * - 2 (?)
     - version
     - array
     - Version [major, minor, rev, build] (optional)
   * - 3 (?)
     - slot
     - uint
     - Target slot (default 1 for upgrade slot, optional)

**Behavior:**

1. Appliance validates size fits in slot
2. Appliance checks if upload already in progress with same hash (resume)
3. If new upload, appliance erases target slot
4. On success, appliance responds with OTA_STATUS (state: RECEIVING)
5. On error, appliance responds with ERROR_INVALID_CMD or ERROR_STATE_REJECT

**Errors:**

- Size exceeds slot: ERROR_INVALID_CMD (error_code: 1, rejected_field: 0, constraint: IMAGE_TOO_LARGE)
- Update in progress (different hash): ERROR_STATE_REJECT (error_code: <current_state>, rejection_reason: UPDATE_IN_PROGRESS)
- Invalid state: ERROR_STATE_REJECT (error_code: <current_state>, rejection_reason: INVALID_IN_STATE)

OTA_DATA (0x41)
===============

Transfer a chunk of firmware data.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - offset
     - uint
     - Byte offset in image (0-based)
   * - 1
     - data
     - bytes
     - Chunk data (≤96 bytes recommended)

**Behavior:**

1. Appliance validates offset matches expected position
2. Appliance writes data to flash at slot1 + offset
3. Appliance updates internal state (next expected offset)
4. On success, appliance responds with OTA_STATUS (state: RECEIVING, offset: next expected byte)
5. On error, appliance responds with ERROR_INVALID_CMD or ERROR_STATE_REJECT

**Chunk Size Considerations:**

- Fusain payload limit: 114 bytes
- CBOR overhead: ~10 bytes (type, map, keys, offset encoding)
- Recommended chunk size: **96 bytes** (allows for CBOR overhead)
- Smaller chunks increase transfer time but reduce retry cost

**Errors:**

- No upload in progress: ERROR_STATE_REJECT (error_code: <current_state>, rejection_reason: INVALID_IN_STATE)
- Offset mismatch: ERROR_INVALID_CMD (error_code: 1, rejected_field: 0, constraint: VALUE_CONFLICT)
- Flash write failed: ERROR_INVALID_CMD (error_code: 1, rejected_field: 1, constraint: FLASH_WRITE_FAILED)

OTA_VERIFY (0x42)
=================

Request verification of uploaded image.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0 (?)
     - hash
     - bytes
     - Expected SHA256 hash (optional, for confirmation)

**Behavior:**

1. Appliance computes SHA256 of received image
2. Appliance compares against hash from OTA_START (and payload if provided)
3. Appliance validates MCUboot header and TLVs
4. On success, appliance responds with OTA_STATUS (state: VERIFIED)
5. On error, appliance responds with ERROR_INVALID_CMD or ERROR_STATE_REJECT

**Errors:**

- No upload in progress: ERROR_STATE_REJECT (error_code: <current_state>, rejection_reason: INVALID_IN_STATE)
- Upload incomplete: ERROR_INVALID_CMD (error_code: 1, constraint: VALUE_TOO_LOW)
- Hash mismatch: ERROR_INVALID_CMD (error_code: 1, constraint: HASH_MISMATCH)
- Invalid header: ERROR_INVALID_CMD (error_code: 1, constraint: HEADER_INVALID)
- Signature invalid: ERROR_INVALID_CMD (error_code: 1, constraint: SIGNATURE_INVALID) — controllers only, when not in root mode

OTA_ACTIVATE (0x43)
===================

Mark uploaded image for boot.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - mode
     - uint
     - Activation mode (see values below)
   * - 1 (?)
     - reboot
     - bool
     - Reboot immediately after activation (default true)

**Activation Modes**

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - Name
     - Description
   * - 0
     - TEST
     - Test boot (reverts if not confirmed)
   * - 1
     - PERMANENT
     - Permanent boot (no revert)

**Behavior:**

1. Appliance validates image has been verified
2. Appliance marks slot1 for boot (test or permanent)
3. If reboot=true, appliance schedules reboot after response
4. On success, appliance responds with OTA_STATUS (state: ACTIVATED)
5. On error, appliance responds with ERROR_STATE_REJECT

**Errors:**

- Image not verified: ERROR_STATE_REJECT (error_code: <current_state>, rejection_reason: INVALID_IN_STATE)
- Unsafe state (e.g., heating active): ERROR_STATE_REJECT (error_code: <current_state>, rejection_reason: UNSAFE_STATE)

OTA_QUERY (0x44)
================

Request current update status.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - (none)
     -
     -
     - Empty payload

**Behavior:**

1. Appliance responds with OTA_STATUS

OTA_STATUS (0x45)
=================

Report update status. Sent by appliance in response to OTA commands or OTA_QUERY.

**Payload (Appliance → Controller)**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - 0
     - state
     - uint
     - Current OTA state (see values below)
   * - 1 (?)
     - offset
     - uint
     - Next expected byte offset (during upload)
   * - 2 (?)
     - version
     - array
     - Version of pending image (if applicable)

**OTA State Values**

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - Value
     - Name
     - Description
   * - 0
     - IDLE
     - No update in progress
   * - 1
     - RECEIVING
     - Upload in progress
   * - 2
     - RECEIVED
     - Upload complete, awaiting verification
   * - 3
     - VERIFIED
     - Image verified, awaiting activation
   * - 4
     - ACTIVATED
     - Image marked for boot, awaiting reboot

Errors are communicated via ERROR_INVALID_CMD or ERROR_STATE_REJECT messages
as defined in :doc:`fusain-error-communication`. After an error, state returns
to IDLE.

OTA_ABORT (0x4F)
================

Cancel an in-progress update.

**Payload Fields**

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Key
     - Field
     - Type
     - Description
   * - (none)
     -
     -
     - Empty payload

**Behavior:**

1. Appliance cancels any in-progress upload
2. Appliance clears upload state
3. Appliance does NOT erase partially written slot
4. Appliance responds with OTA_STATUS (state: IDLE)

Wire Format Examples
********************

OTA_START Example
=================

Initiate upload of 127,456 byte image:

.. code-block:: text

   CBOR: [0x40, {
     0: 127456,                    // size
     1: h'a1b2c3...32bytes...',   // SHA256 hash
     2: [1, 2, 0, 42]             // version 1.2.0+42
   }]

OTA_DATA Example
================

Send chunk at offset 4096:

.. code-block:: text

   CBOR: [0x41, {
     0: 4096,                      // offset
     1: h'0011223344...96bytes...' // data chunk
   }]

OTA_STATUS Response Example
===========================

Upload in progress at 50% (63,728 bytes received):

.. code-block:: text

   CBOR: [0x45, {
     0: 1,                         // state: RECEIVING
     1: 63728                      // next expected offset
   }]

CDDL Schema
***********

The following CDDL excerpt defines the OTA message payloads. This extends the
existing ``fusain.cddl`` schema.

.. code-block:: cddl

   ; ===========================================================================
   ; OTA Message Payloads (0x40-0x4F)
   ; ===========================================================================
   ;
   ; Message type values:
   ;   0x40=ota-start, 0x41=ota-data, 0x42=ota-verify, 0x43=ota-activate
   ;   0x44=ota-query, 0x45=ota-status, 0x4F=ota-abort

   ; OTA-specific types
   sha256-hash = bstr .size 32          ; SHA256 hash (32 bytes)
   image-version = [                    ; MCUboot version format
       uint .size 1,                    ; major
       uint .size 1,                    ; minor
       uint .size 2,                    ; revision
       uint .size 4,                    ; build number
   ]

   ; OTA state values: 0=idle, 1=receiving, 2=received, 3=verified, 4=activated
   ; Errors are communicated via ERROR_INVALID_CMD/ERROR_STATE_REJECT messages
   ota-state = uint .le 4

   ; Activation mode: 0=test (reverts if not confirmed), 1=permanent
   activation-mode = uint .le 1

   ; OTA_START (0x40) - Initiate firmware upload
   ota-start-payload = {
       0 => uint,                       ; size: Total image size in bytes
       1 => sha256-hash,                ; hash: SHA256 of complete image
       ? 2 => image-version,            ; version: Image version (optional)
       ? 3 => uint .size 1,             ; slot: Target slot, default 1 (optional)
   }

   ; OTA_DATA (0x41) - Transfer firmware chunk
   ota-data-payload = {
       0 => uint,                       ; offset: Byte offset in image (0-based)
       1 => bstr,                       ; data: Chunk data (≤96 bytes recommended)
   }

   ; OTA_VERIFY (0x42) - Request image verification
   ota-verify-payload = {
       ? 0 => sha256-hash,              ; hash: Expected hash for confirmation (optional)
   }

   ; OTA_ACTIVATE (0x43) - Mark image for boot
   ota-activate-payload = {
       0 => activation-mode,            ; mode: 0=test, 1=permanent
       ? 1 => bool,                     ; reboot: Reboot after activation (default true)
   }

   ; OTA_QUERY (0x44) - Request current status
   ; Empty payload (nil or empty map)
   ota-query-payload = {
   }

   ; OTA_STATUS (0x45) - Report update status
   ota-status-payload = {
       0 => ota-state,                  ; state: Current OTA state
       ? 1 => uint,                     ; offset: Next expected byte offset (during upload)
       ? 2 => image-version,            ; version: Version of pending image
   }

   ; OTA_ABORT (0x4F) - Cancel in-progress update
   ; Empty payload (nil or empty map)
   ota-abort-payload = {
   }

Error Handling
**************

Integration with Error Communication
====================================

OTA errors use the extended error communication scheme from
:doc:`fusain-error-communication`:

**OTA-Specific Constraint Values (10-19):**

.. list-table::
   :header-rows: 1
   :widths: 10 30 60

   * - Value
     - Name
     - Description
   * - 10
     - FLASH_WRITE_FAILED
     - Flash write operation failed
   * - 11
     - IMAGE_TOO_LARGE
     - Image exceeds slot size
   * - 12
     - SIGNATURE_INVALID
     - Signature verification failed
   * - 13
     - VERSION_DOWNGRADE
     - Version older than current (if blocked)
   * - 14
     - HASH_MISMATCH
     - Image hash doesn't match expected
   * - 15
     - HEADER_INVALID
     - MCUboot header invalid

**OTA-Specific Rejection Reasons (4-7):**

.. list-table::
   :header-rows: 1
   :widths: 10 30 60

   * - Value
     - Name
     - Description
   * - 4
     - UPDATE_IN_PROGRESS
     - Another update already in progress
   * - 5
     - UNSAFE_STATE
     - Device in state where update is unsafe (e.g., heating)

Error Response Examples
=======================

**Image too large:**

.. code-block:: text

   CBOR: [0xE0, {0: 1, 1: 0, 2: 11}]
   Meaning: Invalid parameter, field 0 (size), IMAGE_TOO_LARGE

**Update already in progress:**

.. code-block:: text

   CBOR: [0xE1, {0: 1, 1: 4}]
   Meaning: Rejected in RECEIVING state, UPDATE_IN_PROGRESS

**Flash write failed:**

.. code-block:: text

   CBOR: [0xE0, {0: 1, 1: 1, 2: 10}]
   Meaning: Invalid parameter, field 1 (data), FLASH_WRITE_FAILED

Multi-Appliance Considerations
******************************

Addressing
==========

OTA commands MUST be unicast (addressed to specific device):

- **DO NOT** use broadcast address for OTA
- Each appliance processes only its own updates
- Router (Slate) tracks update progress per-device

Sequential Updates
==================

When updating multiple appliances:

1. **One at a time:** Only one OTA transfer active per physical link
2. **Track progress:** Controller maintains state for each device
3. **Handle failures:** Failed update on one device doesn't affect others
4. **Shared buffer:** Slate can reuse stored firmware for multiple appliances

Update Coordination
===================

For systems with dependencies (e.g., Slate depends on Helios):

1. Update appliances first (Helios)
2. Verify appliance boots successfully
3. Update controller (Slate)
4. Verify controller boots and reconnects

**Version Compatibility:**

Consider adding protocol version negotiation or compatibility checks if
firmware versions have protocol-breaking changes.

Implementation Notes
********************

Appliance Implementation (Zephyr)
=================================

Integrate with Zephyr's img_mgmt module:

.. code-block:: c

   #include <zephyr/dfu/img_util.h>
   #include <zephyr/storage/flash_map.h>

   // State tracking
   struct ota_state {
       int area_id;           // Flash area (-1 if idle)
       size_t offset;         // Next expected offset
       size_t size;           // Total image size
       uint8_t hash[32];      // Expected SHA256
       enum ota_state state;
   };

   static struct ota_state ota;

   // Handle OTA_START
   int handle_ota_start(uint32_t size, const uint8_t *hash) {
       const struct flash_area *fa;
       int rc;

       // Open slot1
       rc = flash_area_open(FIXED_PARTITION_ID(slot1_partition), &fa);
       if (rc) return -1;

       // Validate size
       if (size > fa->fa_size) {
           flash_area_close(fa);
           return ERR_IMAGE_TOO_LARGE;
       }

       // Erase slot
       rc = flash_area_erase(fa, 0, fa->fa_size);
       if (rc) {
           flash_area_close(fa);
           return ERR_FLASH_ERASE_FAILED;
       }

       // Initialize state
       ota.area_id = fa->fa_id;
       ota.offset = 0;
       ota.size = size;
       memcpy(ota.hash, hash, 32);
       ota.state = OTA_RECEIVING;

       return 0;
   }

   // Handle OTA_DATA
   int handle_ota_data(uint32_t offset, const uint8_t *data, size_t len) {
       const struct flash_area *fa;
       int rc;

       if (ota.state != OTA_RECEIVING) {
           return ERR_INVALID_STATE;
       }

       if (offset != ota.offset) {
           return ERR_OFFSET_MISMATCH;
       }

       rc = flash_area_open(ota.area_id, &fa);
       if (rc) return -1;

       rc = flash_area_write(fa, offset, data, len);
       flash_area_close(fa);

       if (rc) {
           return ERR_FLASH_WRITE_FAILED;
       }

       ota.offset += len;

       if (ota.offset >= ota.size) {
           ota.state = OTA_RECEIVED;
       }

       return 0;
   }

Controller Implementation
=========================

For proxy updates, Slate buffers firmware and transfers:

.. code-block:: c

   // Transfer buffered firmware to appliance
   int transfer_firmware_to_appliance(uint64_t address, const char *path) {
       struct fs_file_t file;
       uint8_t chunk[96];
       size_t offset = 0;
       ssize_t bytes;
       int rc;

       // Open buffered firmware file
       fs_file_t_init(&file);
       rc = fs_open(&file, path, FS_O_READ);
       if (rc) return rc;

       // Get file size and hash
       // ... (read header, compute hash)

       // Send OTA_START
       rc = fusain_send_ota_start(address, size, hash);
       if (rc) goto cleanup;

       // Wait for OTA_STATUS response
       // ...

       // Send chunks
       while ((bytes = fs_read(&file, chunk, sizeof(chunk))) > 0) {
           rc = fusain_send_ota_data(address, offset, chunk, bytes);
           if (rc) goto cleanup;

           // Wait for OTA_STATUS response
           // ...

           offset += bytes;
       }

       // Send OTA_VERIFY
       rc = fusain_send_ota_verify(address, hash);
       // ...

       // Send OTA_ACTIVATE
       rc = fusain_send_ota_activate(address, OTA_MODE_TEST, true);
       // ...

   cleanup:
       fs_close(&file);
       return rc;
   }

Transfer Time Estimates
=======================

With 96-byte chunks at 115200 baud:

.. list-table::
   :header-rows: 1
   :widths: 25 25 25 25

   * - Firmware Size
     - Chunks
     - Transfer Time
     - With Overhead
   * - 124 KB (Helios)
     - ~1,330
     - ~15 sec
     - ~25 sec
   * - 397 KB (Slate)
     - ~4,250
     - ~47 sec
     - ~80 sec

*Overhead includes acknowledgments, processing time, and retries.*

Security Considerations
***********************

Trust Model
===========

Signature verification is enforced at the **controller level**, not the appliance level:

**Controllers (Slate):**

- Verify image signatures before initiating OTA transfer
- Enforce downgrade protection policies
- Gate all firmware updates to connected appliances
- Support "root mode" to allow unsigned images

**Appliances (Helios):**

- Accept images from controller without signature verification
- Trust the controller to have validated the image
- Only verify image integrity (hash) not authenticity (signature)

**Rationale:**

1. **Physical access bypasses everything** — Hardware has accessible programming
   pins (SWD/JTAG) when disassembled for servicing. Preventing custom firmware
   is impossible by design, so the security model acknowledges this reality.

2. **Controller as gateway** — All OTA updates flow through the controller,
   making it the natural enforcement point.

3. **Simpler appliances** — Appliances don't need crypto libraries for signature
   verification, reducing code size and complexity.

4. **User choice** — Users who want custom firmware can enable root mode on
   their controller, or use programming pins directly.

Root Mode
=========

Root mode allows controllers to accept and forward unsigned firmware images.

**Enabling Root Mode:**

Root mode is enabled via a persistent flag in flash. To prevent accidental
activation, enabling requires:

1. Physical button hold during boot (e.g., hold USER button for 5 seconds)
2. Controller displays warning and confirmation prompt
3. User confirms via UI or serial command
4. Flag is set in NVS/flash

**Behavior When Enabled:**

- Controller accepts unsigned images for OTA
- Controller displays "ROOT MODE" indicator in UI
- Signature verification is skipped, hash verification still required
- Downgrade protection can be optionally bypassed

**Behavior When Disabled (default):**

- Controller rejects unsigned images with ``SIGNATURE_INVALID`` error
- Only Thermoquad-signed official releases are accepted
- Downgrade protection is enforced

**Disabling Root Mode:**

- Toggle off via settings menu (requires confirmation)
- Or flash official firmware via programming pins (resets to locked)

Image Signing
=============

MCUboot supports image signing with:

- ECDSA (P-256, recommended)
- RSA (2048, 3072)
- Ed25519

**Recommendation:** Use ECDSA P-256 for Thermoquad:

- Compact signatures (~64 bytes)
- Hardware acceleration on RP2350
- Good balance of security and performance

**Signing Workflow:**

.. code-block:: bash

   # Sign image with Thermoquad production key
   imgtool sign --key thermoquad-prod.pem \
       --align 4 --version 1.2.0 --header-size 0x200 \
       --slot-size 0xD0000 \
       zephyr.bin signed-firmware.bin

Downgrade Protection
====================

Version downgrade blocking:

- **Enabled (default):** Reject OTA if version < current
- **Disabled:** Allow any version (requires root mode)

**Recommendation:** Enforce in production, allow bypass only in root mode.

Conclusion
**********

The proposed OTA message types provide:

1. **Complete update lifecycle:** Start, data, verify, activate, status, abort
2. **Resumable transfers:** Hash-based deduplication for interrupted uploads
3. **Multi-appliance support:** Addressed commands, sequential updates
4. **MCUboot integration:** Compatible with standard Zephyr DFU
5. **Error reporting:** Integrated with extended error communication scheme
6. **Proxy updates:** Slate can buffer and forward to Helios

**Next Steps:**

1. Add OTA message types to Fusain specification
2. Implement OTA handler in Fusain C library
3. Add OTA support to Heliostat for testing
4. Implement proxy update in Slate firmware

References
**********

- :doc:`rp2350-flash-usage` — Flash partitioning and storage options
- :doc:`fusain-error-communication` — Extended error reporting scheme
- `MCUboot Documentation <https://docs.mcuboot.com/>`_
- `Zephyr MCUmgr <https://docs.zephyrproject.org/latest/services/device_mgmt/mcumgr.html>`_
- `Zephyr DFU <https://docs.zephyrproject.org/latest/services/device_mgmt/dfu.html>`_
