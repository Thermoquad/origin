.. _fusain-v3-proposal:

Fusain Protocol v3.0 Proposal (CBOR)
####################################

.. note::

   This is a **proposal** for Fusain v3.0. It replaces v2.0, which was only
   deployed in lab environments.

Overview
********

Fusain v3.0 proposes replacing the hand-crafted binary payloads with CBOR
(Concise Binary Object Representation) encoded payloads, using Zephyr's native
**zcbor** library.

Why CBOR?
=========

1. **Zephyr's Standard** - MCUmgr (Zephyr's device management protocol) uses CBOR
2. **Native Support** - zcbor is already in the Zephyr module ecosystem
3. **Schema-Driven** - CDDL schemas enable code generation and validation
4. **Extensibility** - Add optional fields without breaking compatibility
5. **Self-Describing** - Easier debugging than raw binary structs

What Changes
============

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - Aspect
     - v2.0 (Previous)
     - v3.0 (Proposed)
   * - Payload format
     - Packed C structs
     - CBOR-encoded maps
   * - Schema
     - C header definitions
     - CDDL schema file
   * - Code generation
     - Manual
     - zcbor-generated
   * - Optional fields
     - Padding bytes
     - Native CBOR optionals
   * - Extensibility
     - Breaking changes
     - Backward compatible
   * - Framing layer
     - START/END/CRC
     - **Unchanged**

What Stays the Same
===================

The framing layer is unchanged:

.. code-block:: text

   [START][LENGTH][ADDRESS(8)][CBOR_PAYLOAD][CRC(2)][END]
       │      │         │            │          │      │
       │      │         │            │          │      └─ 0x7F
       │      │         │            │          └─ CRC-16-CCITT
       │      │         │            └─ CBOR-encoded message
       │      │         └─ 64-bit device address
       │      └─ Payload length (now CBOR length)
       └─ 0x7E

- Byte stuffing (escape sequences)
- CRC-16-CCITT error detection
- 64-bit addressing
- Message type ranges (0x10-0x1F, 0x20-0x2F, 0x30-0x3F, 0xE0-0xEF)

Byte Order Rationale
====================

The framing layer uses two byte orders:

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - Field
     - Byte Order
     - Rationale
   * - ADDRESS
     - Little-endian
     - Native byte order for ARM Cortex-M and x86 processors
   * - CRC
     - Big-endian
     - Historical convention from CRC-16-CCITT telecommunications usage

**Why little-endian for ADDRESS?**

The target platforms (RP2350 ARM Cortex-M33, x86 desktop tools) are little-endian.
Using native byte order allows direct memory access without byte-swapping:

.. code-block:: c

   // Direct read - no conversion needed on little-endian hardware
   uint64_t address = *(uint64_t*)&packet->address;

**Why big-endian for CRC?**

CRC-16-CCITT originated in ITU-T telecommunications standards (X.25, HDLC) where
network byte order (big-endian) was the convention. Many protocols follow this:

- HDLC, SDLC - Big-endian CRC
- PPP Frame Check Sequence - Big-endian CRC
- Kermit, XMODEM - Big-endian CRC

Maintaining this convention ensures compatibility with existing CRC implementations
and aligns with developer expectations for CRC-16-CCITT.

**v3.0 Consideration**

This byte order convention is retained in v3.0. CBOR payloads use their own
encoding rules (CBOR integers are big-endian per RFC 8949), but the framing
layer (ADDRESS, CRC) remains unchanged.

CDDL Schema
***********

The complete schema is in :download:`fusain-v3-cbor.cddl`.

Message Structure
=================

All messages are encoded as a CBOR array: ``[type, payload]``

.. code-block:: cddl

   fusain-message = [
       type: msg-type,
       payload: message-payload,
   ]

This allows efficient parsing: read the type value first, then decode the
appropriate payload structure.

Example: State Data
===================

**v2.0 (packed struct, 16 bytes):**

.. code-block:: c

   typedef struct __attribute__((packed)) {
     uint32_t error;      // 4 bytes
     int32_t code;        // 4 bytes
     uint32_t state;      // 4 bytes
     uint32_t timestamp;  // 4 bytes
   } fusain_data_state_t;

**v3.0 (CBOR map, ~12-15 bytes):**

.. code-block:: cddl

   state-data-payload = {
       0 => error: bool,           ; 1 byte (bool is 1 byte in CBOR)
       1 => code: error-code,      ; 1-2 bytes (small int)
       2 => state: state,          ; 1 byte
       3 => timestamp: timestamp,  ; 5 bytes (uint32 + type)
   }

CBOR encoding (hex): ``82 18 30 A4 00 F4 01 00 02 05 03 1A 00 01 E2 40``

Where: ``82`` = array(2), ``18 30`` = uint(0x30), ``A4...`` = map(4) payload

Integration
***********

CMake Setup (Zephyr)
====================

.. code-block:: cmake

   # In your CMakeLists.txt (zcbor is already available via Zephyr)

   # For zcbor code generation:
   find_package(Python3 REQUIRED)

   set(FUSAIN_CDDL ${CMAKE_CURRENT_SOURCE_DIR}/fusain-v3-cbor.cddl)

   # Generate encoder/decoder code from CDDL
   add_custom_command(
     OUTPUT
       ${CMAKE_CURRENT_BINARY_DIR}/fusain_cbor_decode.c
       ${CMAKE_CURRENT_BINARY_DIR}/fusain_cbor_encode.c
       ${CMAKE_CURRENT_BINARY_DIR}/fusain_cbor_types.h
     COMMAND ${Python3_EXECUTABLE} -m zcbor code
       --cddl ${FUSAIN_CDDL}
       --decode --encode
       --output-c ${CMAKE_CURRENT_BINARY_DIR}/fusain_cbor_decode.c
       --output-c ${CMAKE_CURRENT_BINARY_DIR}/fusain_cbor_encode.c
       --output-h ${CMAKE_CURRENT_BINARY_DIR}/fusain_cbor_types.h
       -t fusain-message state-data-payload motor-data-payload
     DEPENDS ${FUSAIN_CDDL}
   )

Kconfig
=======

.. code-block:: kconfig

   # In prj.conf
   CONFIG_ZCBOR=y
   CONFIG_ZCBOR_CANONICAL=y  # Deterministic encoding
   CONFIG_NET_BUF=y          # Pool-based buffer management

Buffer Management
=================

In Zephyr mode, use ``net_buf`` for packet buffer management instead of static
arrays. This provides:

- **Pool-based allocation**: No heap fragmentation on long-running systems
- **Reference counting**: Safe buffer sharing between threads and ISRs
- **Consistent patterns**: Aligns with Zephyr subsystems (Bluetooth, networking)
- **Zero-copy potential**: Pass buffers between layers without copying

**Buffer Pool Definition:**

.. code-block:: c

   #include <zephyr/net/buf.h>

   // Define buffer pool for Fusain packets
   // Size: max CBOR payload + framing overhead
   #define FUSAIN_BUF_SIZE    280
   #define FUSAIN_BUF_COUNT   4

   NET_BUF_POOL_DEFINE(fusain_pool, FUSAIN_BUF_COUNT, FUSAIN_BUF_SIZE,
                       sizeof(struct fusain_buf_ctx), NULL);

   // Per-buffer context (optional metadata)
   struct fusain_buf_ctx {
       uint64_t address;
       uint8_t msg_type;
   };

**Usage Pattern:**

.. code-block:: c

   // Allocate buffer for encoding
   struct net_buf *buf = net_buf_alloc(&fusain_pool, K_NO_WAIT);
   if (!buf) {
       return -ENOMEM;
   }

   // Encode CBOR payload directly into net_buf
   uint8_t *payload = net_buf_tail(buf);
   int len = fusain_encode_state_data(payload, net_buf_tailroom(buf), ...);
   if (len > 0) {
       net_buf_add(buf, len);
   }

   // Pass to transport layer (zero-copy)
   fusain_transport_send(buf);

   // Release when done (reference counted)
   net_buf_unref(buf);

**Integration with zcbor:**

zcbor operates on raw byte arrays, which ``net_buf`` provides via
``net_buf_tail()`` for encoding and ``buf->data`` for decoding:

.. code-block:: c

   // Decoding from net_buf
   int fusain_decode_from_buf(struct net_buf *buf, struct fusain_message *msg)
   {
       ZCBOR_STATE_D(zs, 1, buf->data, buf->len, 1, 0);
       // ... decode using zs ...
   }

CRC Implementation
==================

For v3.0, use Zephyr's native CRC in Zephyr mode while maintaining the built-in
implementation for standalone builds:

.. code-block:: c

   // In fusain.c or fusain_crc.h

   #ifdef CONFIG_ZEPHYR
     #include <zephyr/sys/crc.h>
     #define fusain_crc16(data, len) crc16_itu_t(0xFFFF, data, len)
   #else
     // Use built-in implementation for standalone builds
     uint16_t fusain_crc16(const uint8_t *data, size_t len);
   #endif

**Benefits:**

- Zephyr mode uses optimized, well-tested CRC implementation
- Standalone mode remains self-contained with no external dependencies
- Same API for both modes

**Byte Order:**

The CRC value is transmitted big-endian regardless of which implementation is used:

.. code-block:: c

   uint16_t crc = fusain_crc16(data, len);
   buffer[offset++] = (crc >> 8) & 0xFF;  // MSB first
   buffer[offset++] = crc & 0xFF;         // LSB second

Logging
=======

In Zephyr mode, register a logging module for debug output:

.. code-block:: c

   #include <zephyr/logging/log.h>
   LOG_MODULE_REGISTER(fusain, CONFIG_FUSAIN_LOG_LEVEL);

Add a Kconfig option for log level control:

.. code-block:: kconfig

   # In Kconfig
   module = FUSAIN
   module-str = Fusain Protocol
   source "subsys/logging/Kconfig.template.log_config"

**Benefits:**

- Consistent logging interface across Thermoquad firmwares
- Runtime log level control via shell or Kconfig
- Automatic function/file/line annotation in debug builds
- Deferred logging mode for ISR-safe operation

**Usage:**

.. code-block:: c

   LOG_DBG("Decoding packet, len=%d", len);
   LOG_INF("State transition: %s -> %s", old_state, new_state);
   LOG_WRN("CRC mismatch: expected=0x%04x, got=0x%04x", expected, actual);
   LOG_ERR("Buffer overflow, dropping %d bytes", overflow);

Packet Reception Architecture
=============================

For platforms using UART polling (such as RP2350 which does not support async UART),
use a ring buffer for byte accumulation combined with a message queue for thread-safe
packet handoff:

.. code-block:: none

   UART Polling Thread          Processing Thread
          │                            │
          ▼                            │
     [Poll UART]                       │
          │                            │
          ▼                            │
     [Ring Buffer] ◄─ accumulate bytes │
          │                            │
          ▼                            │
     [Detect complete packet]          │
          │                            │
          ▼                            │
     [net_buf alloc + copy]            │
          │                            │
          ▼                            ▼
     [Message Queue] ──────────► [k_msgq_get()]
                                       │
                                       ▼
                                 [CBOR decode]

**Components:**

1. **Ring Buffer** (``struct ring_buf``): Accumulates raw UART bytes. Handles
   partial packet reception gracefully when bytes arrive across multiple polls.

2. **Packet Detection**: Scans ring buffer for START byte, validates LENGTH,
   checks for END byte at expected position.

3. **Message Queue** (``k_msgq``): Passes ``net_buf`` pointers between threads.
   The polling thread produces, the processing thread consumes.

**Example Implementation:**

.. code-block:: c

   #include <zephyr/sys/ring_buffer.h>
   #include <zephyr/net/buf.h>

   RING_BUF_DECLARE(uart_rx_ring, 256);
   K_MSGQ_DEFINE(packet_queue, sizeof(struct net_buf *), 8, 4);
   NET_BUF_POOL_DEFINE(fusain_pool, 8, FUSAIN_MAX_PACKET_SIZE, 0, NULL);

   // In UART polling thread
   void uart_poll_thread(void)
   {
       uint8_t byte;
       while (1) {
           while (uart_poll_in(uart_dev, &byte) == 0) {
               ring_buf_put(&uart_rx_ring, &byte, 1);
           }

           // Check for complete packet
           struct net_buf *buf = try_extract_packet(&uart_rx_ring);
           if (buf) {
               k_msgq_put(&packet_queue, &buf, K_NO_WAIT);
           }

           k_sleep(K_MSEC(1));
       }
   }

   // In processing thread
   void process_thread(void)
   {
       struct net_buf *buf;
       while (1) {
           if (k_msgq_get(&packet_queue, &buf, K_FOREVER) == 0) {
               fusain_decode_from_buf(buf, &msg);
               net_buf_unref(buf);
           }
       }
   }

**Benefits:**

- Decouples UART timing from packet processing
- Ring buffer handles byte-at-a-time reception efficiently
- Message queue provides backpressure if processing is slow
- Works with UART polling on platforms without async UART support

Usage Example
*************

Encoding State Data
===================

.. code-block:: c

   #include <zcbor_encode.h>
   #include "fusain_cbor_types.h"

   int fusain_encode_state_data(uint8_t *buffer, size_t buffer_size,
                                bool error, int code, int state, uint32_t ts)
   {
       ZCBOR_STATE_E(zs, 1, buffer, buffer_size, 1);

       // Encode: [MSG_STATE_DATA, {0: error, 1: code, 2: state, 3: ts}]
       bool ok = zcbor_list_start_encode(zs, 2);
       ok = ok && zcbor_uint32_put(zs, 0x30);  // MSG_STATE_DATA

       ok = ok && zcbor_map_start_encode(zs, 4);
       ok = ok && zcbor_uint32_put(zs, 0) && zcbor_bool_put(zs, error);
       ok = ok && zcbor_uint32_put(zs, 1) && zcbor_int32_put(zs, code);
       ok = ok && zcbor_uint32_put(zs, 2) && zcbor_uint32_put(zs, state);
       ok = ok && zcbor_uint32_put(zs, 3) && zcbor_uint32_put(zs, ts);
       ok = ok && zcbor_map_end_encode(zs, 4);

       ok = ok && zcbor_list_end_encode(zs, 2);

       return ok ? (buffer_size - zs->payload_end + zs->payload) : -1;
   }

Decoding State Data
===================

.. code-block:: c

   #include <zcbor_decode.h>
   #include "fusain_cbor_types.h"

   int fusain_decode_state_data(const uint8_t *buffer, size_t len,
                                struct state_data_payload *out)
   {
       ZCBOR_STATE_D(zs, 1, buffer, len, 1, 0);

       uint32_t msg_type;
       bool ok = zcbor_list_start_decode(zs);
       ok = ok && zcbor_uint32_decode(zs, &msg_type);

       if (!ok || msg_type != 0x30) {
           return -1;  // Not a STATE_DATA message
       }

       // Use generated decoder for the payload
       ok = ok && cbor_decode_state_data_payload(zs, out);
       ok = ok && zcbor_list_end_decode(zs);

       return ok ? 0 : -1;
   }

Size Comparison
***************

.. list-table::
   :header-rows: 1
   :widths: 40 20 20 20

   * - Message
     - v2.0 Size
     - v3.0 Size
     - Delta
   * - STATE_DATA
     - 16 bytes
     - ~13 bytes
     - -19%
   * - MOTOR_DATA
     - 32 bytes
     - ~18 bytes
     - -44%
   * - PING_RESPONSE
     - 4 bytes
     - 7 bytes
     - +75%
   * - PING_REQUEST
     - 0 bytes
     - 1 byte
     - +1 byte

**Note:** Sizes shown are payload-only (the CBOR map), not including the
``[type, payload]`` message wrapper (adds 3 bytes). Small messages get
slightly larger due to CBOR overhead. Complex messages with optional fields
get smaller.

Documentation Changes
*********************

If this proposal is accepted, the following documentation updates are required:

Sphinx Documentation
====================

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - File
     - Changes Required
   * - ``specifications/fusain/overview.rst``
     - Update version to v3.0, add CBOR/zcbor mention
   * - ``specifications/fusain/packet-format.rst``
     - Update payload section to describe CBOR encoding, add CDDL reference
   * - ``specifications/fusain/packet-payloads.rst``
     - **Replace entirely** with CBOR payload documentation, reference CDDL schema
   * - ``specifications/fusain/messages.rst``
     - Update payload descriptions to reference CDDL types
   * - ``specifications/fusain/implementation.rst``
     - Rewrite for zcbor-based implementation, add CMake/Kconfig examples
   * - ``specifications/fusain/v3-proposal.rst``
     - **Delete** (proposal becomes the specification)
   * - ``specifications/fusain/fusain-v3-cbor.cddl``
     - **Rename** to ``fusain.cddl``, becomes canonical schema

New Documentation
=================

The following new pages should be added:

1. **CDDL Schema Reference** (``specifications/fusain/cddl-schema.rst``)

   - Rendered view of the CDDL schema with explanations
   - Map key definitions (why integer keys)
   - Optional field semantics

2. **CBOR Debugging** (``tools/cbor-debugging.rst``)

   - How to decode CBOR payloads for debugging
   - Recommended tools (cbor.me, cbor-diag)
   - Heliostat updates for CBOR support

Library Documentation
=====================

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - File
     - Changes Required
   * - ``modules/lib/fusain/CLAUDE.md``
     - Update architecture section for CBOR, document generated code
   * - ``modules/lib/fusain/README.md``
     - Add zcbor dependency, update build instructions, document CDDL schema
   * - ``modules/lib/fusain/include/fusain/fusain.h``
     - Remove packed struct definitions (replaced by generated types)

Code Changes
============

The following source files require updates:

**Fusain Library** (``modules/lib/fusain/``)

.. code-block:: text

   fusain/
   ├── fusain.cddl              # NEW: CDDL schema (canonical)
   ├── CMakeLists.txt           # Add zcbor code generation
   ├── Kconfig                  # Add CONFIG_ZCBOR dependency
   ├── include/fusain/
   │   ├── fusain.h             # Keep framing API, remove payload structs
   │   └── fusain_types.h       # GENERATED from CDDL
   └── src/
       ├── fusain.c             # Keep framing, update payload handling
       ├── fusain_encode.c      # GENERATED from CDDL
       └── fusain_decode.c      # GENERATED from CDDL

**Helios Firmware** (``apps/helios/``)

- Update ``serial_handler.c`` to use zcbor encode/decode
- Update ``prj.conf`` to enable ``CONFIG_ZCBOR``
- Regenerate after CDDL changes

**Slate Firmware** (``apps/slate/``)

- Same updates as Helios

**Heliostat Tool** (``tools/heliostat/``)

- Update ``pkg/fusain/`` to decode CBOR payloads
- Add CBOR diagnostic output mode
- Consider using ``fxamacker/cbor`` Go library

Dual-Mode CMake Build
=====================

The Fusain library currently supports dual-mode builds:

1. **Zephyr Mode**: As a Zephyr module (detected via ``ZEPHYR_BASE``)
2. **Standalone Mode**: As a portable C library for desktop tools and testing

This dual-mode capability is essential for:

- Running standalone tests with 100% code coverage
- Building Heliostat and other desktop tools
- Integration testing without Zephyr

**Challenge: zcbor is Zephyr-specific**

zcbor is a Zephyr module and not easily available outside the Zephyr ecosystem.
This creates a dependency conflict for standalone builds.

**Proposed Solution: CBOR Abstraction Layer**

.. code-block:: text

   fusain/
   ├── src/
   │   ├── fusain.c              # Framing (unchanged)
   │   ├── fusain_cbor_zephyr.c  # zcbor implementation (Zephyr mode)
   │   └── fusain_cbor_tinycbor.c # tinycbor implementation (standalone mode)
   └── CMakeLists.txt            # Conditional source selection

**CMakeLists.txt Changes:**

.. code-block:: cmake

   if(ZEPHYR_BASE)
     # ============================================================
     # ZEPHYR MODE - Use zcbor (native Zephyr module)
     # ============================================================
     if(CONFIG_FUSAIN)
       zephyr_library()
       zephyr_library_sources(
         ${ZEPHYR_CURRENT_MODULE_DIR}/src/fusain.c
         ${ZEPHYR_CURRENT_MODULE_DIR}/src/fusain_cbor_zephyr.c
       )

       # CDDL code generation (Zephyr build system handles zcbor)
       # Generated files: fusain_cbor_types.h, fusain_cbor_encode.c, fusain_cbor_decode.c
     endif()
   else()
     # ============================================================
     # STANDALONE MODE - Use tinycbor or other portable CBOR library
     # ============================================================
     project(fusain VERSION 3.0.0 LANGUAGES C)

     # Option 1: Bundle tinycbor as a submodule
     add_subdirectory(extern/tinycbor EXCLUDE_FROM_ALL)

     add_library(fusain STATIC
       src/fusain.c
       src/fusain_cbor_tinycbor.c
     )
     target_link_libraries(fusain PRIVATE tinycbor)

     # Pre-generated types from CDDL (committed to repo)
     # OR: Run zcbor code generation via Python at configure time
   endif()

**Alternative: Pre-Generated Code**

For simpler maintenance, pre-generate the CBOR encode/decode code and commit it:

.. code-block:: text

   fusain/
   ├── generated/                # Committed generated code
   │   ├── fusain_cbor_types.h
   │   ├── fusain_cbor_encode.c
   │   └── fusain_cbor_decode.c
   └── scripts/
       └── regenerate_cbor.sh    # Script to regenerate from CDDL

This approach:

- Eliminates build-time code generation complexity
- Keeps standalone builds simple
- Requires manual regeneration when CDDL changes

**Standalone CBOR Library Options:**

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - Library
     - Size
     - Notes
   * - `tinycbor <https://github.com/intel/tinycbor>`_
     - ~10KB
     - Intel/Qt maintained, widely used, easy integration
   * - `cn-cbor <https://github.com/jimsch/cn-cbor>`_
     - ~5KB
     - Minimal, header-only option available
   * - `libcbor <https://github.com/PJK/libcbor>`_
     - ~50KB
     - Full-featured, may be overkill for standalone tools

**Recommendation:** Use tinycbor for standalone mode. It's well-maintained,
lightweight, and provides similar encode/decode APIs to zcbor.

**Test Infrastructure Impact:**

The standalone test infrastructure (``tests/standalone/``) will need updates:

1. Link against tinycbor instead of zcbor
2. Update ztest compatibility layer if needed
3. Ensure same test coverage for both CBOR backends

**Scope:**

This dual-mode complexity only affects the C library (``modules/lib/fusain/``).
Heliostat (Go) is unaffected—it will simply use
`fxamacker/cbor/v2 <https://github.com/fxamacker/cbor>`_ for CBOR support.

CRC Selection Rationale
***********************

Fusain uses CRC-16-CCITT (polynomial 0x1021, initial value 0xFFFF) for error
detection. This section documents the rationale for this choice and considers
alternatives.

Current CRC: CRC-16-CCITT
=========================

**Configuration:**

- Polynomial: 0x1021 (CCITT)
- Initial value: 0xFFFF
- No input/output reflection (ITU-T variant)
- 2-byte overhead per packet

**Hamming Distance Performance:**

Based on `Koopman's CRC research <https://users.ece.cmu.edu/~koopman/crc/>`_:

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - Hamming Distance
     - Max Data Length
     - Error Detection
   * - HD=4
     - 32,751 bits (~4KB)
     - All 1, 2, 3-bit errors
   * - HD=5
     - 241 bits (~30 bytes)
     - All 1-4 bit errors
   * - HD=6
     - 135 bits (~17 bytes)
     - All 1-5 bit errors

For Fusain packets (max ~260 bytes = 2,080 bits), CRC-16-CCITT provides:

- **HD=4**: Detects all 1, 2, and 3-bit errors
- Burst error detection: Any burst ≤16 bits
- Undetected error probability: ~1/65,536 for random errors

Why CRC-16-CCITT Is Appropriate
===============================

1. **Sufficient for Packet Sizes**

   Fusain packets are well under the 32,751-bit HD=4 limit. At typical
   packet sizes (50-150 bytes), CRC-16-CCITT reliably detects the most
   common error patterns.

2. **Industry Standard**

   CRC-16-CCITT is widely used in embedded protocols:

   - X.25, HDLC, SDLC link layers
   - Kermit, XMODEM file transfer
   - Bluetooth baseband
   - PPP frame check sequence

3. **Zephyr Native Support**

   Zephyr provides optimized O(n) implementations:

   .. code-block:: c

      #include <zephyr/sys/crc.h>

      // ITU-T variant (no reflection) - used by Fusain
      uint16_t crc = crc16_itu_t(0xFFFF, data, len);

      // CCITT variant (with reflection)
      uint16_t crc = crc16_ccitt(0xFFFF, data, len);

4. **Low Overhead**

   2 bytes per packet is acceptable for our message sizes. Smaller CRCs
   (CRC-8) would provide insufficient HD for our packet lengths.

Alternative: CRC-32K/4.2 (Koopman)
==================================

For applications requiring stronger error detection, Zephyr provides
Koopman's CRC-32K/4.2 polynomial:

**Configuration:**

- Polynomial: 0x93A409EB
- 4-byte overhead per packet

**Hamming Distance Performance:**

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - Hamming Distance
     - Max Data Length
     - Error Detection
   * - HD=4
     - 2,147,483,615 bits
     - All 1-3 bit errors (essentially unlimited)
   * - HD=5
     - 6,167 bits (~770 bytes)
     - All 1-4 bit errors
   * - HD=6
     - 6,167 bits (~770 bytes)
     - All 1-5 bit errors
   * - HD=7
     - 148 bits (~18 bytes)
     - All 1-6 bit errors

For Fusain packets (max ~260 bytes = 2,080 bits), CRC-32K/4.2 provides:

- **HD=6**: Detects all 1, 2, 3, 4, and 5-bit errors
- Significantly stronger than CRC-16-CCITT

**Zephyr API:**

.. code-block:: c

   #include <zephyr/sys/crc.h>

   uint32_t crc = crc32_k_4_2_update(0xFFFFFFFF, data, len);

**Trade-offs:**

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - Aspect
     - CRC-16-CCITT
     - CRC-32K/4.2
   * - Overhead
     - 2 bytes
     - 4 bytes
   * - HD at 260 bytes
     - HD=4
     - HD=6
   * - Undetected error prob.
     - ~1/65,536
     - ~1/4,294,967,296
   * - Computation
     - Faster (16-bit)
     - Slightly slower (32-bit)
   * - Industry adoption
     - Ubiquitous
     - Emerging (Koopman research)

Recommendation
==============

**Retain CRC-16-CCITT for Fusain v3.0.**

Rationale:

1. **HD=4 is industry-accepted** for embedded serial protocols. Most protocols
   (Modbus, HDLC, Bluetooth) use 16-bit CRCs.

2. **Error environment is controlled**: Fusain operates over short point-to-point
   links (LIN bus, RS-485, UART). The bit error rate in these environments is
   typically low enough that HD=4 provides adequate protection.

3. **Overhead matters**: For small messages like PING (4-6 bytes), the 2-byte
   CRC overhead is already significant. A 4-byte CRC would be 50-100% overhead.

If future requirements demand stronger error detection (e.g., longer cable runs,
noisier environments, safety certification), CRC-32K/4.2 can be adopted as a
v3.1 enhancement with a new framing variant.

Documentation Updates
=====================

If this proposal is accepted, add the following to the specification:

1. **packet-format.rst**: Add "CRC Selection" section explaining:

   - Why CRC-16-CCITT was chosen
   - Hamming distance properties
   - Link to Koopman's research

2. **implementation.rst**: Add CRC implementation guidance:

   - Zephyr API usage (``crc16_itu_t``)
   - Test vectors for validation
   - Common implementation pitfalls

Decision
********

.. admonition:: Open Question

   Should we proceed with Fusain v3.0 (CBOR)?

   **Pros:**

   - Aligns with Zephyr ecosystem (MCUmgr uses CBOR)
   - Better extensibility for future features
   - Schema validation catches errors at compile time
   - Native zcbor support already in workspace

   **Cons:**

   - Slight overhead for small messages
   - Requires regenerating code when schema changes
   - Learning curve for CDDL syntax
   - More complex debugging (need CBOR tools)

References
**********

CBOR and zcbor
==============

- `zcbor GitHub <https://github.com/NordicSemiconductor/zcbor>`_
- `CBOR RFC 8949 <https://www.rfc-editor.org/rfc/rfc8949.html>`_
- `CDDL RFC 8610 <https://datatracker.ietf.org/doc/rfc8610/>`_

CRC Research
============

- `Koopman CRC Zoo <https://users.ece.cmu.edu/~koopman/crc/>`_ - Comprehensive CRC polynomial database
- `CRC Polynomial Selection for Embedded Networks <https://users.ece.cmu.edu/~koopman/roses/dsn04/koopman04_crc_poly_embedded.pdf>`_ - Koopman & Chakravarty, DSN 2004

Fusain
======

- :ref:`Fusain v2.0 Specification <fusain-protocol>`
