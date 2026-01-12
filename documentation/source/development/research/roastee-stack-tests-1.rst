Roastee Stack Tests #1
######################

:Date: 2026-01-12
:Author: Thermoquad
:Status: Completed
:Related: :doc:`roastee-web-stack`

.. note::

   **Experiment Results (Updated with Fusain Protocol Library)**

   This document presents the results of web stack experiments proposed in
   :doc:`roastee-web-stack`. All measurements now **include the Fusain
   protocol library** (~12 KB gzipped), ensuring realistic bundle sizes
   for production deployment.

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

Six experiments were conducted to evaluate web stack choices for the Roastee PWA.
The results challenge several assumptions from the initial research phase and
provide concrete data for technology selection.

**Key Findings (with Fusain protocol library included):**

- **SolidJS outperforms Svelte 5** for bundle size (16.54 KB vs 23.03 KB gzipped)
- **Svelte 5 runes add unexpected overhead** compared to Svelte 4
- **UnoCSS beats Tailwind v4** (19.61 KB vs 20.07 KB gzipped)
- **Manual service worker saves 11 KB** over Workbox
- **Full stack prototype: 31.54 KB** (21.0% of 150 KB budget)

**Revised Recommendations:**

1. Use **SolidJS** as the component framework (smallest runtime + compiled output)
2. Use **UnoCSS** with Tailwind preset for atomic CSS
3. Use **minimal i18n** with TypeScript ``as const`` (no library needed)
4. Use **Nanostores** for state management
5. Use **manual service worker** (avoid Workbox bloat)
6. Use **@solidjs/router** for production-ready routing

These findings contradict the original recommendation of Svelte 5, based on
measured data from identical prototype implementations.

Experiment Results
******************

Experiment 1: Framework Bundle Comparison
=========================================

**Objective:** Verify framework runtime sizes with identical functionality.

**Prototype:** Device dashboard with connection status, 5 telemetry values,
and heater toggle button with reactive 100ms updates.

**Results:**

.. list-table:: Framework Bundle Sizes (with Fusain, gzipped)
   :widths: 25 20 20 20 15
   :header-rows: 1

   * - Framework
     - JS (min)
     - JS (gzip)
     - CSS (gzip)
     - Total
   * - **SolidJS**
     - 44.85 KB
     - 16.18 KB
     - 0.36 KB
     - **16.54 KB**
   * - Preact
     - 50.42 KB
     - 18.56 KB
     - 0.36 KB
     - **18.93 KB**
   * - Svelte 5
     - 61.49 KB
     - 22.60 KB
     - 0.43 KB
     - **23.03 KB**

**Analysis:**

This result contradicts the initial research. Svelte 5 produced the **largest**
bundle, not the smallest. The reasons:

1. **Svelte 5 runes add runtime overhead** not present in Svelte 4
2. **SolidJS has a smaller runtime** (~4 KB) with efficient compiled output
3. **For small apps (<10 components), SolidJS wins on size**

The original research cited Svelte's ~1.8-2.6 KB runtime, but this applies to
Svelte 4. Svelte 5's new runes system adds signals-based reactivity that
increases the compiled output size.

**Winner:** SolidJS (16.54 KB with Fusain)

Experiment 2: CSS Framework Output
==================================

**Objective:** Compare CSS output sizes with realistic utility usage.

**Prototype:** Dashboard styling with responsive layout, dark/light theme,
cards, buttons, and typography utilities (~50 unique classes).

**Results:**

.. list-table:: CSS Framework Sizes (with Fusain, gzipped)
   :widths: 25 20 20 20 15
   :header-rows: 1

   * - Framework
     - JS (gzip)
     - CSS (gzip)
     - Total
     - Notes
   * - **Vanilla**
     - 16.67 KB
     - 0.72 KB
     - **17.39 KB**
     - Manual CSS
   * - **UnoCSS**
     - 17.31 KB
     - 2.30 KB
     - **19.61 KB**
     - On-demand
   * - Tailwind v4
     - 17.30 KB
     - 2.77 KB
     - **20.07 KB**
     - Purge-based

**Analysis:**

- **Vanilla CSS is smallest** but requires manual authoring
- **UnoCSS beats Tailwind** by 0.46 KB gzipped
- **CSS framework overhead is 2-2.5 KB** (acceptable for 150 KB budget)
- **JS overhead is minimal** (~0.64 KB difference)

The original research cited an 8x difference (32 KB vs 4 KB), but this was
comparing older Tailwind with JIT to UnoCSS. Tailwind v4's new architecture
significantly reduces output, though UnoCSS still wins.

**Winner:** UnoCSS (19.61 KB total with Fusain, best DX/size balance)

Experiment 3: i18n Integration
==============================

**Objective:** Validate i18n library bundle impact.

**Prototype:** Dashboard with language switcher (EN/ES/FR), ~25 translation
keys covering UI labels, units, and controls.

**Results:**

.. list-table:: i18n Library Sizes (with Fusain, gzipped)
   :widths: 30 20 20 30
   :header-rows: 1

   * - Solution
     - JS (gzip)
     - CSS (gzip)
     - Total
   * - **Minimal (as const)**
     - 17.70 KB
     - 2.19 KB
     - **19.89 KB**
   * - typesafe-i18n
     - 18.64 KB
     - 2.19 KB
     - **20.83 KB**

**Analysis:**

- **typesafe-i18n adds ~1 KB** to the JS bundle
- **Both provide full type safety** via TypeScript
- **Minimal is simpler** - just a lookup function and translations object

The minimal approach uses TypeScript's ``as const`` assertion for type inference:

.. code-block:: typescript

   export const translations = {
     en: {
       'dashboard.title': 'Device Dashboard',
       'controls.heater': 'Heater',
       // ...
     },
     es: { /* ... */ },
     fr: { /* ... */ },
   } as const;

   export type TranslationKey = keyof typeof translations.en;
   export const t = (locale: Locale, key: TranslationKey) => translations[locale][key];

This provides compile-time key validation without library overhead.

**Winner:** Minimal i18n (19.89 KB total with Fusain)

Experiment 4: PWA Service Worker
================================

**Objective:** Compare Workbox vs manual service worker for embedded PWA.

**Prototype:** Offline caching with cache-first for assets, network-first for
navigation, and basic update handling.

**Results:**

.. list-table:: Service Worker Sizes (with Fusain, gzipped)
   :widths: 25 20 20 20 15
   :header-rows: 1

   * - Approach
     - App JS
     - SW
     - CSS
     - Total
   * - **Manual**
     - 15.52 KB
     - 0.63 KB
     - 1.92 KB
     - **18.07 KB**
   * - Workbox
     - 18.72 KB*
     - 8.17 KB
     - 1.92 KB
     - **28.81 KB**

\*Workbox app JS includes workbox-window registration module.

**Analysis:**

- **Workbox SW is 13x larger** (8.17 KB vs 0.63 KB gzipped)
- **Manual SW is ~50 lines** with basic cache-first strategy
- **Workbox adds 11 KB total overhead**

The manual service worker provides all functionality Roastee needs:

.. code-block:: javascript

   // ~50 lines for cache-first + network-first hybrid
   const CACHE_NAME = 'roastee-v1';

   self.addEventListener('fetch', (event) => {
     if (event.request.mode === 'navigate') {
       // Network-first for HTML
       event.respondWith(fetch(event.request).catch(() => caches.match(event.request)));
     } else {
       // Cache-first for assets
       event.respondWith(caches.match(event.request).then(c => c || fetch(event.request)));
     }
   });

Workbox's advanced features (background sync, retry queues, workbox-window)
are unnecessary for Roastee's simple caching needs.

**Winner:** Manual service worker (18.07 KB total with Fusain)

Experiment 5: Full Stack Integration
====================================

**Objective:** Combine all winners and measure complete prototype.

**Stack:**

- SolidJS (Experiment 1 winner)
- UnoCSS (Experiment 2 winner)
- Minimal i18n (Experiment 3 winner)
- Manual service worker (Experiment 4 winner)
- Nanostores for state management
- @solidjs/router for routing

**Features:**

- Dashboard with live telemetry display
- Device controls (heater, pump)
- Settings page (language, theme)
- Full i18n (EN/ES/FR)
- PWA offline support

**Results:**

.. list-table:: Full Stack Bundle Sizes (with Fusain and @solidjs/router, gzipped)
   :widths: 40 30 30
   :header-rows: 1

   * - Component
     - Size (min)
     - Size (gzip)
   * - App JS
     - 78.42 KB
     - 28.69 KB
   * - CSS
     - 8.12 KB
     - 2.33 KB
   * - Service Worker
     - —
     - 0.52 KB
   * - **Total**
     - 86.54 KB
     - **31.54 KB**

**Budget Analysis:**

.. list-table:: Budget Comparison
   :widths: 30 25 25 20
   :header-rows: 1

   * - Threshold
     - Budget
     - Actual
     - Status
   * - Target (75%)
     - 150 KB
     - 31.54 KB
     - ✅ 21.0%
   * - Maximum
     - 200 KB
     - 31.54 KB
     - ✅ 15.8%

**Projection:**

- Current prototype (2 pages, ~400 LoC): 31.54 KB
- Estimated full app (~10 pages): ~50-60 KB
- With icons/assets (10 KB): ~60-70 KB
- **Projected total: ~70 KB** (47% of 150 KB budget)

**Winner:** SolidJS + UnoCSS + Minimal i18n + @solidjs/router + Fusain (31.54 KB)

Experiment 6: Alternative Stack Comparison
==========================================

**Objective:** Compare winning stack against original research hypothesis.

**Stacks Compared:**

.. list-table:: Stack Comparison
   :widths: 20 40 40
   :header-rows: 1

   * - Aspect
     - Stack A (Winner)
     - Stack B (Hypothesis)
   * - Framework
     - SolidJS
     - Svelte 5
   * - CSS
     - UnoCSS
     - Tailwind v4
   * - State
     - Nanostores
     - Svelte runes

Both stacks implemented identical features: Dashboard, Settings, i18n, routing.

**Results:**

.. list-table:: Stack Comparison Results (with Fusain, gzipped)
   :widths: 30 20 20 20 10
   :header-rows: 1

   * - Stack
     - JS
     - CSS
     - Total
     - Delta
   * - **SolidJS + UnoCSS**
     - 20.90 KB
     - 2.35 KB
     - **23.25 KB**
     - —
   * - Svelte + Tailwind
     - 27.08 KB
     - 3.07 KB
     - **30.15 KB**
     - +30%

**Analysis:**

SolidJS stack is **23% smaller** than Svelte stack (6.90 KB difference).

This contradicts the original research hypothesis that Svelte would produce
smaller bundles. The reasons:

1. **Svelte 5 runes compile to larger output** than expected
2. **SolidJS runtime is more efficient** than Svelte 5
3. **Crossover point requires 100+ components** (unlikely for Roastee)

The original research cited Svelte's 0.493 bytes/byte growth rate vs React's
0.153 bytes/byte, suggesting Svelte would win at larger app sizes. However:

- SolidJS growth characteristics are similar to Svelte
- The runtime difference (~7 KB) dominates at Roastee's scale
- Svelte 5's runes add overhead not present in Svelte 4

**Winner:** SolidJS + UnoCSS (23.75 KB with Fusain)

Conclusions
***********

Research Hypothesis vs Reality
==============================

.. list-table:: Hypothesis Validation
   :widths: 35 30 35
   :header-rows: 1

   * - Hypothesis
     - Expected
     - Actual
   * - Svelte 5 smallest framework
     - ~2 KB runtime
     - **23.03 KB** (largest tested)
   * - UnoCSS 8x smaller than Tailwind
     - ~4 KB vs ~32 KB
     - **2.30 KB vs 2.77 KB** (1.2x)
   * - typesafe-i18n optimal
     - ~1 KB overhead
     - **1 KB overhead** (confirmed, but unnecessary)
   * - Manual SW saves space
     - Significant savings
     - **11 KB savings** (confirmed)

Key insights:

1. **Svelte 5 is not Svelte 4** — The runes system adds significant overhead
2. **Tailwind v4 improved significantly** — But UnoCSS still wins
3. **Libraries aren't always needed** — Minimal solutions often suffice
4. **Measure, don't assume** — Published benchmarks may not reflect your use case

Final Recommended Stack
=======================

Based on measured results:

.. list-table:: Final Stack
   :widths: 25 25 50
   :header-rows: 1

   * - Component
     - Choice
     - Rationale
   * - Framework
     - **SolidJS**
     - Smallest bundle, efficient reactivity
   * - CSS
     - **UnoCSS**
     - On-demand generation, Tailwind syntax
   * - i18n
     - **Minimal (as const)**
     - Type-safe without library overhead
   * - State
     - **Nanostores**
     - Tiny, framework-agnostic
   * - PWA
     - **Manual SW**
     - 10 KB savings over Workbox
   * - Router
     - **@solidjs/router**
     - Production-ready routing with lazy loading support

**Total prototype size:** 31.54 KB (21.0% of 150 KB budget, includes Fusain protocol
library and @solidjs/router)

On-Device Testing: Bucket
*************************

The experiments were validated on real embedded hardware using **Bucket**, a
minimal Zephyr HTTP server running on Raspberry Pi Pico 2W.

Bucket Overview
===============

Bucket serves experiment builds over WiFi with gzip compression, providing
realistic testing of embedded deployment. Key features:

- **Gzip-compressed static assets** served from flash
- **WebSocket endpoint** (``/ws/fusain``) with simulated 60-second burn cycle
- **WiFi connectivity** via shell commands (AP mode or client connection)
- **Fusain protocol integration** for telemetry simulation

**Verification Tasks:**

The following Taskfile commands in ``packages/experiments/bucket/`` can be used
to reproduce and verify results:

.. code-block:: bash

   # Build firmware images for all experiments
   task images-all

   # Display firmware flash/RAM usage per experiment
   task images-stats

   # Display asset compression statistics
   task assets-stats

   # Deploy and test a specific experiment
   task deploy-exp5-fullstack
   task rebuild-firmware

Embedded Firmware Results
=========================

.. list-table:: Firmware Size by Experiment
   :widths: 30 25 25 20
   :header-rows: 1

   * - Experiment
     - Flash (bytes)
     - RAM (bytes)
     - Notes
   * - exp1-solid
     - 610,936
     - 163,205
     - Smallest
   * - exp1-preact
     - 613,272
     - 165,541
     -
   * - exp2-unocss
     - 613,988
     - 166,257
     -
   * - exp1-svelte
     - 617,392
     - 169,661
     -
   * - exp5-fullstack
     - 617,700
     - 169,969
     - Full prototype

These results confirm the bundle size measurements—SolidJS produces the
smallest firmware image, and Svelte 5 produces the largest among tested
frameworks.

Asset Compression Analysis
==========================

Bucket embeds web assets with gzip compression. The ``assets-stats`` task
reports compression ratios across all experiments:

.. list-table:: Compression Statistics Summary
   :widths: 30 25 25 20
   :header-rows: 1

   * - Metric
     - Plain (bytes)
     - Gzip (bytes)
     - Saving
   * - Total (all exp)
     - 282,751
     - 102,864
     - 63.6%
   * - exp1-solid
     - 45,965
     - 16,845
     - 63.4%
   * - exp5-fullstack
     - 67,019
     - 23,588
     - 64.8%

**Gzip compression achieves ~64% average reduction**, consistent with typical
minified JavaScript compression ratios.

Brotli Investigation
====================

.. note::

   The data in this section was collected before @solidjs/router was added to
   exp5-full-stack. The relative savings (~10%) remain valid, but absolute
   sizes have increased in the final prototype.

Brotli compression was investigated as it typically provides ~10% smaller
output than gzip. A local ``file2hex.py`` with brotli support was implemented
and tested:

.. list-table:: Gzip vs Brotli (exp5-fullstack)
   :widths: 25 25 25 25
   :header-rows: 1

   * - File
     - Gzip
     - Brotli
     - Savings
   * - app.js
     - 20,903 B
     - 18,850 B
     - 9.8%
   * - app.css
     - 2,348 B
     - 2,022 B
     - 13.9%
   * - index.html
     - 337 B
     - 221 B
     - 34.4%

**Finding:** Brotli provides ~10% additional savings over gzip.

**Limitation:** Browsers only support brotli (``Content-Encoding: br``) over
**HTTPS connections**. Over plain HTTP, browsers send ``Accept-Encoding: gzip``
only. Since Bucket serves over HTTP for development, brotli cannot be used
until TLS is implemented.

**Recommendation:** Use gzip for HTTP testing. Implement HTTPS/TLS for
production deployments to enable brotli compression.

WebSocket Fusain Integration
============================

Bucket includes a WebSocket endpoint at ``/ws/fusain`` that simulates a
60-second burn cycle with realistic telemetry:

- Temperature ramp from 25°C to 200°C during heating
- RPM control matching Helios behavior
- Proper Fusain packet framing with CBOR payloads

This validated the Fusain TypeScript client implementation and identified a
bug where wrong property names were used (``messageType`` vs ``type``,
``payload`` vs ``payloadMap``). The fix was applied to all experiment variants.

Routing Analysis
================

This section documents the analysis that led to adopting ``@solidjs/router``
for the production prototype. The original experiments used a custom hash-based
router (~0.3 KB), but analysis showed the router library cost is justified.

Router Bundle Impact
--------------------

Measured by adding ``@solidjs/router`` to exp5-full-stack:

.. list-table:: Router Bundle Cost
   :widths: 40 25 25 10
   :header-rows: 1

   * - Configuration
     - JS (min)
     - JS (gzip)
     - Delta
   * - Custom hash router
     - 58.39 KB
     - 21.04 KB
     - —
   * - With @solidjs/router
     - 80.65 KB
     - 29.18 KB
     - **+8.14 KB**

**@solidjs/router adds 8.14 KB gzipped** to the bundle.

Feature Comparison
------------------

.. list-table:: Router Feature Matrix
   :widths: 40 20 20 20
   :header-rows: 1

   * - Feature
     - Custom
     - @solidjs/router
     - Impact
   * - Basic navigation
     - Yes
     - Yes
     - —
   * - Hash-based URLs
     - Yes
     - Yes
     - —
   * - Route parameters (``/device/:id``)
     - No
     - Yes
     - High
   * - Nested routes
     - No
     - Yes
     - Medium
   * - Lazy loading routes
     - No
     - Yes
     - High
   * - Route guards/preload
     - No
     - Yes
     - Medium
   * - Catch-all routes
     - No
     - Yes
     - Low
   * - Type-safe params
     - No
     - Yes
     - Medium

Scaling Recommendation
----------------------

The original research assumed "3-5 views" based on an informal list (dashboard,
settings, setup). This assumption was not validated against actual requirements.

**For applications with 10+ routes, @solidjs/router is recommended:**

1. **Route parameters** are essential for device/profile detail views
2. **Lazy loading** can save more than the 8 KB router cost at scale
3. **Maintenance burden** of manual routing becomes significant
4. **Type safety** prevents routing bugs as route count grows

**Break-even analysis:** If lazy loading defers 3+ pages averaging 3 KB each,
the router pays for itself in reduced initial bundle size.

.. list-table:: Budget Impact
   :widths: 40 20 20 20
   :header-rows: 1

   * - Configuration
     - Bundle (gzip)
     - % of 150 KB
     - Status
   * - Without router library
     - 23.75 KB
     - 15.8%
     - Historical baseline
   * - With @solidjs/router
     - 31.54 KB
     - 21.0%
     - **Current prototype**
   * - With lazy loading (est.)
     - ~25-28 KB
     - ~17-19%
     - Optimal at scale

**Conclusion:** The 8 KB router cost is acceptable given the 150 KB budget
and is likely offset by lazy loading benefits in a production application
with 10+ routes. Based on this analysis, **exp5-full-stack was updated to use
@solidjs/router**, bringing the prototype to 31.54 KB (21.0% of budget).

Proposed Further Experiments
****************************

Experiment 7: Accessibility Audit
=================================

**Objective:** Validate WCAG 2.1 AA compliance.

The prototypes focused on bundle size, not accessibility. A dedicated
experiment should:

1. Run Lighthouse accessibility audit
2. Test with screen readers (VoiceOver, TalkBack)
3. Verify keyboard navigation
4. Check color contrast in both themes
5. Test live regions for telemetry updates

**Success criteria:** Lighthouse accessibility score >90.

**Priority:** High — Required for consumer product.

Experiment 8: Production Build Optimization
===========================================

**Objective:** Optimize production build for embedded deployment with TLS.

Additional optimizations to test once HTTPS is available:

1. **Brotli compression** (~10% smaller than gzip, requires HTTPS)
2. Code splitting for async-loaded features
3. Preload hints for critical assets
4. Service worker precaching strategies

**Measurements:**

- Compressed bundle size (gzip vs brotli)
- First load time on 3G connection
- Cache hit rate after initial load

**Priority:** Medium — After TLS implementation.

Appendix A: Experiment Source Code
**********************************

All experiment source code is available in the Roastee repository:

.. code-block:: none

   roastee/
   └── packages/
       └── experiments/
           ├── README.md              # Summary of all experiments
           ├── bucket/                # Embedded HTTP server for testing
           │   ├── src/main.c         # Zephyr HTTP server
           │   ├── src/ws_fusain.c    # WebSocket Fusain endpoint
           │   ├── Taskfile.dist.yml  # Build and deploy tasks
           │   └── images/            # Built firmware images
           ├── exp1-framework/        # Svelte, SolidJS, Preact comparison
           │   ├── svelte/
           │   ├── solid/
           │   └── preact/
           ├── exp2-css/              # UnoCSS, Tailwind, Vanilla comparison
           │   ├── unocss/
           │   ├── tailwind/
           │   └── vanilla/
           ├── exp3-i18n/             # typesafe-i18n vs minimal
           │   ├── typesafe-i18n/
           │   └── minimal/
           ├── exp4-pwa/              # Manual SW vs Workbox
           │   ├── manual/
           │   └── workbox/
           ├── exp5-full-stack/       # Combined winning stack
           └── exp6-alt-stack/        # Svelte + Tailwind comparison
               └── svelte-tailwind/

**Tag:** ``archive/stack-experiments-v1-2026-01-12``

Each experiment is a standalone Vite project that can be built and measured
independently.

Appendix B: Measurement Methodology
***********************************

All measurements used consistent methodology:

**Build Configuration:**

- Vite with esbuild minification
- ``target: 'esnext'`` for modern browsers
- Production mode (``vite build``)

**Size Measurement:**

.. code-block:: bash

   # Minified size from Vite output
   vite build  # Reports sizes in build output

   # Gzipped size verification
   gzip -c dist/assets/*.js | wc -c
   gzip -c dist/assets/*.css | wc -c

**Environment:**

- Node.js 25.x
- pnpm 10.x
- Linux (Arch Linux 6.17.9)

All experiments ran on the same machine with identical configurations to
ensure comparable results.
