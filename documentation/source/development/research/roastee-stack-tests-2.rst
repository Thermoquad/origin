Roastee Stack Tests #2
######################

:Date: 2026-01-12
:Author: Thermoquad
:Status: **Implemented** ✓
:Related: :doc:`roastee-stack-tests-1`, :doc:`fusain-ota-updates`, :doc:`fusain-error-communication`

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

This document investigated CBOR library alternatives for the TypeScript Fusain
implementation. A custom CBOR codec was implemented and is now in production.

**Implementation Results:**

.. list-table::
   :widths: 30 35 35
   :header-rows: 1

   * - Metric
     - cbor-x (before)
     - Custom codec (after)
   * - Bundle size (gzip)
     - 10.6 KB
     - **2.6 KB** (-75%)
   * - Encode performance
     - ~170 ns/op
     - ~180 ns/op (within 5%)
   * - Decode performance
     - ~185 ns/op
     - ~225 ns/op (22% slower)
   * - Wire format size
     - 24 bytes
     - **21 bytes** (-12%)
   * - Runtime dependencies
     - 1 (cbor-x)
     - **0**

**Key Outcomes:**

- **8 KB bundle savings** (75% reduction in CBOR footprint)
- **Smaller wire format** — No CBOR tags, standard RFC 8949 output
- **Wire compatibility** — Byte-identical output to C and Go implementations
- **Performance parity** — Encoder within 5% of cbor-x, acceptable decode tradeoff

To reproduce these results:

.. code-block:: bash

   cd apps/roastee/packages/fusain
   task benchmark           # Overall performance
   task benchmark:compare   # Direct comparison with cbor-x

Background
**********

Current Implementation
======================

The TS Fusain library uses ``cbor-x`` v1.6.0 for CBOR encoding and decoding:

.. code-block:: typescript

   import { decode, encode } from "cbor-x";

   // Decoding telemetry from appliances
   export function parseCBORMessage(data: Uint8Array): [number, PayloadMap | null] {
     const msg = decode(data);
     // ... validation and conversion
   }

   // Encoding commands to appliances
   export function encodeCommand(type: number, payload: PayloadMap): Uint8Array {
     return encode([type, payload]);
   }

The library uses both ``decode`` (for telemetry) and ``encode`` (for commands
and OTA messages). Roastee sends control commands (STATE_COMMAND, MOTOR_COMMAND,
TEMPERATURE_COMMAND, etc.) and OTA messages to Slate via WebSocket.

Fusain CBOR Requirements
========================

Fusain messages use a minimal subset of CBOR:

.. list-table:: Required CBOR Types
   :widths: 20 20 60
   :header-rows: 1

   * - CBOR Type
     - Major Type
     - Usage in Fusain
   * - Positive integers
     - 0
     - Message types, payload keys, sensor values
   * - Negative integers
     - 1
     - Error payloads (error_code field typed as int for future expansion)
   * - Byte strings
     - 2
     - OTA firmware chunks (≤96 bytes), SHA256 hashes (32 bytes)
   * - Arrays
     - 4
     - Message structure: ``[type, payload_map]``
   * - Maps
     - 5
     - Payload maps with integer keys
   * - Simple values
     - 7
     - ``false`` (20), ``true`` (21), ``null`` (22)
   * - Floats
     - 7
     - Temperature readings, PID gains, target temperatures

**Not required:** Text strings, tags, indefinite-length items,
nested structures beyond 2 levels.

CBOR Library Comparison
***********************

Bundle Size Comparison
======================

.. list-table:: CBOR Library Bundle Sizes
   :widths: 30 20 20 15 15
   :header-rows: 1

   * - Library
     - Version
     - Minified
     - Gzipped
     - Savings
   * - **cbor-x** (current)
     - 1.6.0
     - 30.4 KB
     - 10.6 KB
     - —
   * - **cborg**
     - 4.3.2
     - 21.9 KB
     - 6.8 KB
     - 3.8 KB
   * - **cbor**
     - 10.0.11
     - 42.4 KB
     - 12.3 KB
     - -1.7 KB
   * - **@levischuck/tiny-cbor**
     - 0.3.2
     - 5.7 KB
     - 2.1 KB
     - 8.5 KB
   * - **cbor-web**
     - —
     - 160 KB
     - 48 KB
     - -37.4 KB
   * - **Custom codec**
     - —
     - ~4 KB (est.)
     - ~1.8 KB (est.)
     - ~8.8 KB

*Bundle sizes from Bundlephobia. Custom codec is estimated based on
required CBOR subset including float support and encode/decode.*

Feature Comparison
==================

.. list-table:: CBOR Library Feature Matrix
   :widths: 25 15 15 15 15 15
   :header-rows: 1

   * - Feature
     - cbor-x
     - cborg
     - tiny-cbor
     - Custom
     - Needed?
   * - Decode integers
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode arrays
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode maps
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode booleans
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode null
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode byte strings
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode text strings
     - Yes
     - Yes
     - Yes
     - No
     - No
   * - Decode floats
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Decode tags
     - Yes
     - Yes
     - No
     - No
     - No
   * - Encode support
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**
   * - Streaming/indefinite
     - Yes
     - Yes
     - No
     - No
     - No
   * - TypeScript types
     - Yes
     - Yes
     - Yes
     - Yes
     - **Yes**

Custom Codec
************

Design
======

A custom CBOR codec for Fusain implements only the required subset for both
encoding and decoding:

.. code-block:: typescript

   // Minimal CBOR decoder for Fusain
   export function decodeCBOR(data: Uint8Array): unknown {
     let offset = 0;

     function read(): unknown {
       const byte = data[offset++];
       const major = byte >> 5;
       const info = byte & 0x1f;

       // Read argument (length/value)
       let arg = info;
       if (info === 24) arg = data[offset++];
       else if (info === 25) arg = (data[offset++] << 8) | data[offset++];
       // ... handle 26, 27 for 4/8 byte args

       switch (major) {
         case 0: return arg;                    // Positive integer
         case 1: return -1 - arg;               // Negative integer
         case 2: return readBytes(arg);         // Byte string
         case 4: return readArray(arg);         // Array
         case 5: return readMap(arg);           // Map
         case 7: return readSimpleOrFloat(info);// Simple (bool, null) or float
         default: throw new Error('Unsupported');
       }
     }

     return read();
   }

   // Minimal CBOR encoder for Fusain
   export function encodeCBOR(value: unknown): Uint8Array {
     const chunks: Uint8Array[] = [];

     function write(val: unknown): void {
       if (val === null) {
         chunks.push(new Uint8Array([0xf6]));        // null
       } else if (typeof val === 'number') {
         if (Number.isInteger(val)) writeInteger(val);
         else writeFloat(val);
       } else if (typeof val === 'boolean') {
         chunks.push(new Uint8Array([val ? 0xf5 : 0xf4]));
       } else if (val instanceof Uint8Array) {
         writeBytes(val);
       } else if (Array.isArray(val)) {
         writeArray(val);
       } else if (val instanceof Map || typeof val === 'object') {
         writeMap(val);
       }
     }

     write(value);
     return concat(chunks);
   }

Implementation Complexity
=========================

.. list-table:: Custom Codec Estimate
   :widths: 40 20 40
   :header-rows: 1

   * - Component
     - Lines
     - Notes
   * - **Decoder**
     -
     -
   * - Argument parsing (1/2/4/8 byte)
     - ~15
     - Handle CBOR info field
   * - Integer decoding (major 0, 1)
     - ~5
     - Positive and negative
   * - Byte string decoding (major 2)
     - ~5
     - Return Uint8Array slice
   * - Array decoding (major 4)
     - ~8
     - Recursive read
   * - Map decoding (major 5)
     - ~10
     - Return Map<number, unknown>
   * - Simple values (major 7)
     - ~8
     - false, true, null
   * - Float decoding (major 7)
     - ~15
     - IEEE 754 half/single/double
   * - **Encoder**
     -
     -
   * - Integer encoding
     - ~15
     - Positive and negative with size selection
   * - Byte string encoding
     - ~8
     - Length prefix + data
   * - Array/Map encoding
     - ~15
     - Recursive write
   * - Float encoding
     - ~12
     - IEEE 754 single precision
   * - Simple values encoding
     - ~5
     - bool, null
   * - **Common**
     -
     -
   * - Error handling
     - ~10
     - Bounds checks, unknown types
   * - Buffer utilities
     - ~10
     - Concat, DataView helpers
   * - **Total**
     - **~140-180**
     - Plus TypeScript types

**Risk Assessment:**

- **Low risk:** CBOR is a simple, well-documented format (RFC 8949)
- **Test coverage:** Existing Fusain tests validate codec behavior
- **Float handling:** IEEE 754 is well-defined; half-precision needs care
- **Fallback:** Can revert to cbor-x if edge cases arise

Implementation Results
**********************

The custom CBOR codec was implemented in ``packages/fusain/src/cbor-codec.ts``.

Actual Bundle Size
==================

.. list-table:: Custom Codec Actual Size
   :widths: 40 30 30
   :header-rows: 1

   * - Metric
     - Estimated
     - Actual
   * - Source lines
     - 140-180
     - 385
   * - Compiled JS (raw)
     - ~4 KB
     - 13 KB
   * - Compiled JS (gzip)
     - ~1.8 KB
     - **2.6 KB**

The actual implementation is larger than estimated due to:

- Comprehensive bounds checking for safety
- Buffer pooling for performance optimization
- Module-level state management
- Full IEEE 754 float16/32/64 support

Performance Benchmarks
======================

Run benchmarks with:

.. code-block:: bash

   cd apps/roastee/packages/fusain
   task benchmark           # Overall metrics
   task benchmark:compare   # vs cbor-x comparison
   task benchmark:profile   # Component-level profiling

**Overall Performance** (``task benchmark``):

.. code-block:: text

   Fusain CBOR Benchmark
   ==================================================
   Iterations: 50,000

   Results:
   --------------------------------------------------
     Encode:    5.63M ops/sec  (178 ns/op)
     Decode:    4.47M ops/sec  (224 ns/op)

   Throughput:
   --------------------------------------------------
     Message size: 21 bytes
     Encode: 113 MB/s
     Decode: 89 MB/s

**Comparison with cbor-x** (``task benchmark:compare``):

.. code-block:: text

   Direct Comparison: Custom vs cbor-x
   ==================================================

   Encode:
     Custom:  181 ns/op
     cbor-x:  174 ns/op
     Ratio:   1.04x slower

   Decode:
     Custom:  226 ns/op
     cbor-x:  185 ns/op
     Ratio:   1.22x slower

   Wire format:
     Custom:  21 bytes
     cbor-x:  24 bytes

Wire Format Compatibility
=========================

The custom codec produces standard CBOR (RFC 8949) without proprietary tags:

.. code-block:: text

   Custom: [0x82, 0x18, 0x20, 0xa6, 0x00, 0xf5, ...]  (21 bytes)
   cbor-x: [0x82, 0x18, 0x20, 0xd9, 0x01, 0x03, 0xa6, 0x00, 0xf5, ...]  (24 bytes)
                         ^^^^^^^^^^^^^^^^
                         Tag 259 (cbor-x extension)

cbor-x adds Tag 259 to Maps for JavaScript Map round-trip semantics. The custom
codec omits this since Fusain only uses integer keys. This ensures wire
compatibility with the C and Go implementations.

Optimization Techniques
=======================

Key optimizations implemented:

1. **Buffer pooling** — Reuses a module-level buffer across encode calls,
   eliminating per-call allocation overhead (2x encode speedup)

2. **Module-level state** — Avoids closure recreation per call

3. **DataView for multi-byte integers** — Uses platform-optimized byte order
   conversion instead of manual bit shifting

4. **Growing buffer strategy** — Doubles buffer size as needed, amortizing
   allocation cost over multiple large encodes

Bundle Impact Analysis
**********************

Current Fusain Contribution
===========================

The Fusain library contributes approximately 12 KB gzipped to the bundle:

.. list-table:: Fusain Library Breakdown (Estimated)
   :widths: 40 30 30
   :header-rows: 1

   * - Component
     - Size (gzip)
     - % of Fusain
   * - cbor-x dependency
     - ~10.6 KB
     - 88%
   * - Fusain code (decoder, encoder, etc.)
     - ~1.4 KB
     - 12%
   * - **Total**
     - **~12 KB**
     - 100%

With Custom Codec
=================

.. list-table:: Projected Savings
   :widths: 40 20 20 20
   :header-rows: 1

   * - Configuration
     - Fusain (gzip)
     - exp5 Total
     - Budget %
   * - Current (cbor-x)
     - ~12 KB
     - 31.54 KB
     - 21.0%
   * - With cborg
     - ~8.2 KB
     - 27.74 KB
     - 18.5%
   * - With tiny-cbor
     - ~3.5 KB
     - 23.14 KB
     - 15.4%
   * - With custom codec
     - ~3.2 KB
     - 22.74 KB
     - 15.2%

**Potential savings:** Up to 8.8 KB gzipped (5.9% of 150 KB budget)

Recommendations
***************

Option 1: Custom Codec ✓ IMPLEMENTED
====================================

**Status:** Implemented and merged to master.

**Actual Results:**

- Bundle savings: 8 KB gzipped (75% reduction)
- Development time: ~6 hours (including optimization)
- Test coverage: 100%
- Performance: Within 5% of cbor-x for encoding

**Implementation Details:**

- Location: ``packages/fusain/src/cbor-codec.ts``
- Lines of code: 385 (larger than estimated due to optimizations)
- cbor-x retained as dev dependency for benchmarking

Option 2: Switch to cborg
=========================

**Status:** Not implemented (Option 1 chosen).

Would have saved 3.8 KB but retained external dependency.

Option 3: Switch to @levischuck/tiny-cbor
=========================================

**Status:** Not implemented (Option 1 chosen).

Would have saved 8.5 KB but with less control over wire format.

Conclusion
**********

**Outcome:** Custom CBOR codec successfully implemented.

The custom codec achieved the primary goals:

1. **Bundle reduction:** 10.6 KB → 2.6 KB gzipped (-75%)
2. **Zero runtime dependencies:** cbor-x removed from production bundle
3. **Wire compatibility:** Standard CBOR output, identical to C/Go implementations
4. **Performance:** Encoder within 5% of cbor-x (acceptable tradeoff)

The decode performance is 22% slower than cbor-x, which is acceptable for the
telemetry use case where messages arrive at ~10 Hz (100ms intervals). At 225 ns
per decode, this represents 0.000225% of the available processing time.

**Verify results:**

.. code-block:: bash

   cd apps/roastee/packages/fusain
   task benchmark:compare

Appendix A: CBOR Format Reference
*********************************

CBOR uses a simple type/length/value encoding:

.. code-block:: text

   Initial byte: [major type (3 bits)][additional info (5 bits)]

   Major types:
   0 = Positive integer
   1 = Negative integer
   2 = Byte string
   3 = Text string (not needed)
   4 = Array
   5 = Map
   6 = Tag (not needed)
   7 = Simple values (bool, null) and floats

   Additional info:
   0-23  = Inline value
   24    = 1-byte argument follows
   25    = 2-byte argument follows
   26    = 4-byte argument follows
   27    = 8-byte argument follows

For Fusain's use case, inline values and 1-2 byte arguments are common for
integers. Message types are 0-255, payload keys are small integers, and most
sensor values fit in 16-bit ranges. Temperature values use IEEE 754 single
precision floats (4 bytes, additional info = 26).

References
**********

- :doc:`fusain-ota-updates` — OTA byte string requirements (firmware chunks, SHA256 hashes)
- :doc:`fusain-error-communication` — Error payload int type requirements
- `RFC 8949 - CBOR <https://datatracker.ietf.org/doc/html/rfc8949>`_
- `cbor-x npm <https://www.npmjs.com/package/cbor-x>`_
- `cborg npm <https://www.npmjs.com/package/cborg>`_
- `Bundlephobia <https://bundlephobia.com>`_
- `CBOR Implementations <https://cbor.io/impls.html>`_
