Roastee Web Stack
#################

:Date: 2026-01-12
:Author: Thermoquad
:Status: Research Complete (Revised)

.. warning::

   **Recommendations Revised Based on Experiments**

   The original recommendations in this document (Svelte 5, typesafe-i18n) have
   been **superseded** by empirical testing. See :doc:`roastee-stack-tests-1`
   for the authoritative results.

   **Key Changes:**

   - **Framework:** SolidJS (not Svelte 5) — Svelte 5 runes add unexpected overhead
   - **i18n:** Minimal ``as const`` approach (not typesafe-i18n) — No library needed
   - **Compression:** Brotli requires HTTPS; use gzip for HTTP deployments

   The recommendations below represent the **original research hypothesis**.
   Actual measurements contradicted several assumptions.

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

This research investigates web technology choices for Roastee, the Thermoquad
PWA. The primary constraint is **bundle size**, as Roastee must be served from
embedded firmware with limited flash storage (~200-500 KB compressed budget).

**Key Findings:**

- **Svelte 5** offers the smallest runtime (~1.8 KB) with excellent performance
- **UnoCSS** produces 8x smaller CSS than Tailwind in real-world comparisons
- **typesafe-i18n** provides type-safe localization at ~1 KB
- **Nanostores** offers framework-agnostic state management at ~300-800 bytes
- **Vite** with manual service worker provides optimal PWA bundle control
- TypeScript adds negligible runtime overhead (compile-time only)
- Modular architecture is achievable through Vite's tree-shaking and code splitting

**Recommendations:**

1. Use **Svelte 5** as the component framework (smallest runtime, best DX)
2. Use **UnoCSS** with Tailwind preset for atomic CSS (on-demand generation)
3. Use **typesafe-i18n** for internationalization (type-safe, ~1 KB)
4. Use **Nanostores** for state management (framework-agnostic, tiny)
5. Use **Vite** as build tool with manual service worker (avoid Workbox bloat)
6. Use **TypeScript** throughout (zero runtime cost)
7. Avoid SvelteKit—use plain Vite + Svelte for minimal overhead

Constraints
***********

Flash Storage Budget
====================

From the :doc:`user-onboarding` research, the Roastee PWA must fit within the
Slate firmware's flash allocation:

.. list-table:: Flash Budget
   :widths: 40 30 30
   :header-rows: 1

   * - Component
     - Estimated Size
     - Notes
   * - Base Slate firmware
     - ~400 KB
     - Current size
   * - WiFi/BLE stacks
     - ~100 KB
     - May already be included
   * - HTTP server
     - ~20 KB
     - Zephyr HTTP library
   * - **Roastee PWA (gzip)**
     - **200-500 KB**
     - Target budget
   * - **Total**
     - ~720 KB - 1 MB
     - Must fit in 4 MB flash

**Budget:**

.. list-table:: PWA Size Budget
   :widths: 30 25 45
   :header-rows: 1

   * - Threshold
     - Size
     - Notes
   * - **Maximum**
     - 200 KB
     - Hard limit for embedded deployment
   * - **Target (75%)**
     - 150 KB
     - Acceptable production size
   * - **Ideal**
     - <100 KB
     - Leaves room for future features

The 75% target (150 KB) provides headroom for:

- Future feature additions
- Translation string growth
- Third-party library updates
- Safety margin for estimation errors

For reference, a Vite + SvelteKit PWA can achieve ~45 KB gzipped with proper
optimization. [1]_

Modularity Requirements
=======================

Roastee may be built in different configurations:

.. list-table:: Build Configurations
   :widths: 25 75
   :header-rows: 1

   * - Configuration
     - Description
   * - Full
     - Complete PWA with all features (SSO, cloud sync, analytics)
   * - Embedded
     - Minimal PWA served from Slate firmware (no SSO, no cloud)
   * - Standalone
     - Web-hosted version with full features

The web stack must support **tree-shaking** and **code splitting** to enable
these configurations without manual code removal.

Performance Requirements
========================

- **Mobile-first:** Must perform well on mid-range Android devices
- **Offline-first:** Full functionality without network after initial load
- **Fast TTI:** Time to Interactive < 200ms after cache hit
- **Low memory:** Minimize runtime memory footprint

Framework Comparison
********************

Bundle Size Analysis
====================

.. list-table:: Framework Runtime Sizes (min+gzip)
   :widths: 30 25 45
   :header-rows: 1

   * - Framework
     - Bundle Size
     - Notes
   * - **Svelte 5**
     - ~1.8-2.6 KB
     - Compiles to vanilla JS, no runtime
   * - SolidJS
     - ~10 KB
     - Fine-grained reactivity, fastest benchmarks
   * - Preact
     - ~13.7 KB
     - React-compatible, small footprint
   * - Vue 3
     - ~33 KB
     - Larger ecosystem, more features
   * - React 18
     - ~44.5 KB
     - Largest runtime, requires ReactDOM

**Winner: Svelte 5** — Smallest runtime with excellent developer experience.

Svelte compiles components to vanilla JavaScript at build time, eliminating
runtime framework overhead. This is ideal for embedded PWA deployment. [2]_

Growth Characteristics
======================

An important consideration is how bundle size scales with application complexity:

.. list-table:: Bundle Growth Rate
   :widths: 30 30 40
   :header-rows: 1

   * - Framework
     - Growth Factor
     - Implication
   * - Svelte
     - 0.493 bytes/byte
     - Grows faster (compiled output)
   * - React
     - 0.153 bytes/byte
     - Slower growth (shared runtime)

Svelte's compiled output grows faster per line of source code, but starts from
a much smaller baseline. For a PWA under 500 KB budget, Svelte remains smaller
until approximately 80 "TodoMVC-sized" components. [3]_

**Conclusion:** For Roastee's scope (~20-30 components), Svelte remains optimal.

Svelte 5 Runes
==============

Svelte 5 introduces **runes**, a new reactivity system using signals under the
hood. This provides:

- Fine-grained DOM updates (only affected nodes re-render)
- Explicit reactivity with ``$state``, ``$derived``, ``$effect``
- Better performance than Svelte 4's implicit reactivity
- Familiar patterns for developers coming from SolidJS or React hooks

.. code-block:: html

   <script>
     // Svelte 5 runes
     let count = $state(0);
     let doubled = $derived(count * 2);

     function increment() {
       count++;
     }
   </script>

   <button onclick={increment}>
     Count: {count}, Doubled: {doubled}
   </button>

Svelte 5's performance now matches SolidJS in most benchmarks while maintaining
Svelte's superior developer experience. [4]_

CSS Framework Comparison
************************

Bundle Size Analysis
====================

.. list-table:: CSS Framework Output Sizes
   :widths: 30 30 40
   :header-rows: 1

   * - Framework
     - Production Size
     - Generation Method
   * - **UnoCSS**
     - ~4.2 KB
     - On-demand (generates only used)
   * - Tailwind CSS
     - ~32 KB
     - Purge-based (generates all, removes unused)
   * - Bootstrap
     - ~150+ KB
     - Component-based (large baseline)

In a real-world comparison on a React onboarding platform, Tailwind's JIT
output was ~32 KB even after purging, while UnoCSS produced ~4.2 KB for the
same functionality—an **8x reduction**. [5]_

UnoCSS Architecture
===================

UnoCSS differs fundamentally from Tailwind:

- **On-demand generation:** Only generates CSS for classes actually used
- **No purging step:** Avoids the complexity of tree-shaking CSS
- **Faster builds:** Custom parser and AST, faster than PostCSS-based tools
- **Tailwind compatibility:** Full preset available for familiar class names

.. code-block:: typescript

   // vite.config.ts
   import UnoCSS from 'unocss/vite';
   import { presetWind } from 'unocss';

   export default {
     plugins: [
       UnoCSS({
         presets: [presetWind()], // Tailwind-compatible classes
       }),
     ],
   };

**Recommendation:** Use **UnoCSS with presetWind** for Tailwind-compatible
syntax with significantly smaller output.

Dark Mode and Theming
=====================

UnoCSS supports dark mode and custom theming out of the box:

.. code-block:: html

   <!-- Automatic dark mode via class or media query -->
   <div class="bg-white dark:bg-gray-900 text-black dark:text-white">
     Themed content
   </div>

Custom theme colors can be defined in the UnoCSS configuration, enabling
Roastee's brand colors while maintaining small bundle size.

Internationalization
********************

Library Comparison
==================

.. list-table:: i18n Library Sizes
   :widths: 30 25 45
   :header-rows: 1

   * - Library
     - Bundle Size
     - Notes
   * - **typesafe-i18n**
     - ~1 KB
     - Full type-safety, code generation
   * - typed-locale
     - ~1 KB
     - Framework-agnostic
   * - i18next
     - ~14.8 KB
     - Feature-rich, larger
   * - FormatJS (react-intl)
     - ~17.8 KB
     - Enterprise-focused

**Winner: typesafe-i18n** — Smallest with full TypeScript integration.

typesafe-i18n Features
======================

typesafe-i18n provides compile-time type safety for translations:

- **Zero dependencies**
- **Code generation:** TypeScript types generated from translation files
- **Plural rules:** Built-in pluralization support
- **Formatters:** Date, number, and currency formatting
- **Svelte integration:** First-class Svelte support
- **Async loading:** Load locales on demand

.. code-block:: typescript

   // translations/en/index.ts
   export default {
     greeting: 'Hello {name}!',
     items: '{count} item{{s}}',  // Pluralization
   } as const;

Usage in Svelte:

.. code-block:: html

   <script>
     import LL from '$i18n/i18n-svelte';
   </script>

   <p>{$LL.greeting({ name: 'User' })}</p>
   <p>{$LL.items({ count: 5 })}</p>  <!-- "5 items" -->

Locale Detection
================

typesafe-i18n supports automatic locale detection:

.. code-block:: typescript

   import { detectLocale } from 'typesafe-i18n/detectors';

   // Detect from browser settings
   const locale = detectLocale(
     navigator.languages,
     ['en', 'de', 'fr', 'es'],  // Supported locales
     'en'                        // Fallback
   );

For Roastee, locale detection should follow this priority:

1. User preference (stored in device settings via Fusain)
2. Browser ``navigator.languages``
3. Fallback to English

Accessibility
*************

Accessibility (a11y) is essential for a consumer product. Roastee must be usable
by people with disabilities, including those using screen readers, keyboard
navigation, or high-contrast modes.

Svelte Accessibility Features
=============================

Svelte 5 includes built-in accessibility warnings during compilation:

- Missing ``alt`` attributes on images
- Missing ``aria-label`` on interactive elements
- Invalid ``role`` attributes
- Keyboard accessibility issues

These warnings appear in the terminal during development, catching issues early.

.. code-block:: html

   <!-- Svelte warns: A11y: <button> element should have accessible text -->
   <button on:click={toggle}>
     <Icon name="settings" />
   </button>

   <!-- Fixed with aria-label -->
   <button on:click={toggle} aria-label="Open settings">
     <Icon name="settings" />
   </button>

Accessibility Requirements
==========================

Roastee should meet **WCAG 2.1 Level AA** guidelines:

.. list-table:: Key Accessibility Requirements
   :widths: 30 70
   :header-rows: 1

   * - Requirement
     - Implementation
   * - Keyboard navigation
     - All controls focusable and operable via keyboard
   * - Screen reader support
     - Semantic HTML, ARIA labels, live regions for updates
   * - Color contrast
     - 4.5:1 ratio for text, 3:1 for large text/icons
   * - Focus indicators
     - Visible focus ring on interactive elements
   * - Touch targets
     - Minimum 44x44px for mobile touch targets
   * - Motion sensitivity
     - Respect ``prefers-reduced-motion`` media query

Dark Mode and Contrast
======================

UnoCSS supports the ``prefers-color-scheme`` media query and high-contrast modes:

.. code-block:: html

   <!-- Automatic dark mode -->
   <div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
     Content with sufficient contrast in both modes
   </div>

Theme colors should be tested for WCAG contrast compliance in both light and
dark modes.

Live Regions for Telemetry
==========================

Telemetry updates should be announced to screen readers without stealing focus:

.. code-block:: html

   <div aria-live="polite" aria-atomic="true" class="sr-only">
     Temperature: {$telemetry.temperature}°C
   </div>

Use ``aria-live="polite"`` for non-critical updates (telemetry) and
``aria-live="assertive"`` for critical alerts (errors, safety warnings).

State Management
****************

Library Comparison
==================

.. list-table:: State Management Library Sizes
   :widths: 30 25 45
   :header-rows: 1

   * - Library
     - Bundle Size
     - Notes
   * - **Nanostores**
     - 286-818 bytes
     - Framework-agnostic, atomic stores
   * - Svelte stores
     - 0 bytes
     - Built into Svelte
   * - Zustand
     - ~1-3 KB
     - React-focused, simple API

Recommended Approach
====================

For Roastee, use a **hybrid approach**:

1. **Svelte 5 runes** for component-local state
2. **Nanostores** for cross-component shared state
3. **No additional library** for simple cases

Nanostores integrates seamlessly with Svelte:

.. code-block:: typescript

   // stores/device.ts
   import { atom, computed } from 'nanostores';

   export const $deviceAddress = atom<bigint | null>(null);
   export const $telemetry = atom<TelemetryData | null>(null);

   export const $isConnected = computed($deviceAddress, (addr) => addr !== null);

.. code-block:: html

   <!-- Component.svelte -->
   <script>
     import { $deviceAddress, $isConnected } from '../stores/device';
   </script>

   {#if $isConnected}
     <p>Connected to device {$deviceAddress}</p>
   {/if}

Why Not Just Svelte Stores?
===========================

Nanostores offers advantages for Roastee:

- **Framework-agnostic:** Could share stores with future React Native app
- **Tree-shakeable:** Only include stores actually used
- **Atomic updates:** Fine-grained reactivity matches Svelte 5 runes
- **Tiny footprint:** Smaller than adding external state library later

Build Tooling
*************

Vite Configuration
==================

Vite provides optimal build performance for Svelte:

- **Fast HMR:** Instant updates during development
- **ES modules:** Native browser module loading
- **Rollup bundling:** Efficient tree-shaking for production
- **Plugin ecosystem:** UnoCSS, PWA, and Svelte plugins

.. code-block:: typescript

   // vite.config.ts
   import { defineConfig } from 'vite';
   import { svelte } from '@sveltejs/vite-plugin-svelte';
   import UnoCSS from 'unocss/vite';

   export default defineConfig({
     plugins: [
       UnoCSS(),
       svelte(),
     ],
     build: {
       target: 'es2022',
       minify: 'esbuild',
       rollupOptions: {
         output: {
           manualChunks: {
             // Code splitting for modularity
             fusain: ['fusain'],
           },
         },
       },
     },
   });

Why Not SvelteKit?
==================

SvelteKit adds overhead not needed for Roastee:

.. list-table:: Vite+Svelte vs SvelteKit
   :widths: 25 35 40
   :header-rows: 1

   * - Aspect
     - Vite + Svelte
     - SvelteKit
   * - Bundle size
     - Smaller (no router)
     - Larger (+20-30 KB)
   * - SSR
     - Not needed
     - Adds complexity
   * - Routing
     - Simple (few pages)
     - Full router overhead
   * - PWA control
     - Full control
     - Service worker quirks [6]_

Roastee has only a few views (dashboard, settings, setup) and doesn't need
server-side rendering. Plain Vite + Svelte provides a leaner foundation.

Routing Strategy
================

Without SvelteKit, Roastee needs a lightweight routing solution:

.. list-table:: Routing Options
   :widths: 30 20 50
   :header-rows: 1

   * - Option
     - Size
     - Notes
   * - Conditional rendering
     - 0 KB
     - Simplest, state-based view switching
   * - svelte-spa-router
     - ~2 KB
     - Hash-based, no server config needed
   * - tinro
     - ~3 KB
     - Declarative, similar to SvelteKit

**Recommendation:** Use **hash-based routing** for embedded deployment.

Hash routing (``#/dashboard``, ``#/settings``) works without server configuration,
which is essential when serving from Slate's embedded HTTP server. The server
doesn't need to handle client-side routes—all paths resolve to ``index.html``.

.. code-block:: typescript

   // Simple hash-based routing (~500 bytes)
   import { writable, derived } from 'svelte/store';

   export const hash = writable(window.location.hash.slice(1) || '/');

   window.addEventListener('hashchange', () => {
     hash.set(window.location.hash.slice(1) || '/');
   });

   export function navigate(path: string) {
     window.location.hash = path;
   }

.. code-block:: html

   <!-- App.svelte -->
   <script>
     import { hash } from './router';
     import Dashboard from './routes/Dashboard.svelte';
     import Settings from './routes/Settings.svelte';
     import Setup from './routes/Setup.svelte';
   </script>

   {#if $hash === '/' || $hash === '/dashboard'}
     <Dashboard />
   {:else if $hash === '/settings'}
     <Settings />
   {:else if $hash === '/setup'}
     <Setup />
   {/if}

For Roastee's ~3-5 views, this minimal approach avoids library overhead entirely.
If routing complexity grows, ``svelte-spa-router`` can be added later.

PWA Implementation
==================

Avoid Workbox for embedded PWA—it adds significant overhead:

.. code-block:: none

   Workbox output example:
   - sw.js
   - workbox-*.js (multiple chunks)
   Total: ~80+ KB

Instead, use a **manual service worker** for precise control:

.. code-block:: typescript

   // sw.ts - Manual service worker (~2 KB)
   const CACHE_NAME = 'roastee-v1';
   const ASSETS = [
     '/',
     '/index.html',
     '/app.js',
     '/app.css',
   ];

   self.addEventListener('install', (event) => {
     event.waitUntil(
       caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS))
     );
   });

   self.addEventListener('fetch', (event) => {
     event.respondWith(
       caches.match(event.request).then((cached) => cached || fetch(event.request))
     );
   });

This approach provides:

- **Smaller bundle:** ~2 KB vs ~80 KB
- **Full control:** Exact caching behavior
- **No dependencies:** No Workbox runtime

Modular Architecture
********************

Feature Flags
=============

Use compile-time feature flags for build variants:

.. code-block:: typescript

   // vite.config.ts
   export default defineConfig({
     define: {
       __FEATURE_SSO__: JSON.stringify(process.env.FEATURE_SSO === 'true'),
       __FEATURE_CLOUD__: JSON.stringify(process.env.FEATURE_CLOUD === 'true'),
       __FEATURE_ANALYTICS__: JSON.stringify(process.env.FEATURE_ANALYTICS === 'true'),
     },
   });

.. code-block:: html

   <!-- LoginPage.svelte -->
   <script>
     // SSO button only included in full build
     const showSSO = __FEATURE_SSO__;
   </script>

   {#if showSSO}
     <SSOLoginButton />
   {/if}

Vite's dead code elimination removes unreachable branches, ensuring embedded
builds don't include unused features.

Recommended Package Structure
=============================

Building on the existing Roastee monorepo structure:

.. code-block:: none

   roastee/
   ├── packages/
   │   ├── fusain/              # ✅ Exists - Fusain protocol
   │   ├── ui/                  # Svelte components (future)
   │   │   ├── src/
   │   │   │   ├── components/  # Reusable UI components
   │   │   │   ├── stores/      # Nanostores
   │   │   │   └── i18n/        # typesafe-i18n translations
   │   │   └── package.json
   │   └── pwa/                 # PWA application (future)
   │       ├── src/
   │       │   ├── routes/      # Page components
   │       │   ├── App.svelte   # Root component
   │       │   └── main.ts      # Entry point
   │       ├── public/
   │       │   └── manifest.json
   │       └── vite.config.ts
   ├── pnpm-workspace.yaml
   └── package.json

This structure enables:

- **Shared UI components** between PWA and future apps
- **Independent versioning** of packages
- **Selective inclusion** via pnpm workspace dependencies

Authentication State
********************

Roastee must manage authentication with Slate devices. See :doc:`user-onboarding`
for the full authentication flow.

Token Storage
=============

Authentication tokens from Slate should be persisted client-side for session
continuity:

.. list-table:: Storage Options
   :widths: 25 25 50
   :header-rows: 1

   * - Storage
     - Capacity
     - Notes
   * - localStorage
     - ~5 MB
     - Simple, synchronous, persists across sessions
   * - sessionStorage
     - ~5 MB
     - Cleared when tab closes
   * - IndexedDB
     - Large
     - Async, better for complex data

**Recommendation:** Use **localStorage** for auth tokens and device preferences.
It's simple, persistent, and sufficient for Roastee's needs.

.. code-block:: typescript

   // stores/auth.ts
   import { atom, onMount } from 'nanostores';

   export const $authToken = atom<string | null>(null);
   export const $deviceAddress = atom<string | null>(null);

   // Load from localStorage on init
   onMount($authToken, () => {
     const token = localStorage.getItem('roastee:authToken');
     if (token) $authToken.set(token);
   });

   // Persist on change
   $authToken.subscribe((token) => {
     if (token) {
       localStorage.setItem('roastee:authToken', token);
     } else {
       localStorage.removeItem('roastee:authToken');
     }
   });

Multi-Device Support
====================

Users may connect to multiple Thermoquad devices. Store credentials per device:

.. code-block:: typescript

   interface DeviceCredentials {
     address: string;
     name: string;
     token: string;
     lastConnected: number;
   }

   // Store array of known devices
   const devices = JSON.parse(
     localStorage.getItem('roastee:devices') || '[]'
   ) as DeviceCredentials[];

Token Refresh
=============

If Slate implements token expiration, Roastee should handle refresh:

1. Detect 401 Unauthorized response
2. Prompt user to re-authenticate
3. Update stored token on success

For embedded deployment (no internet), tokens can be long-lived since the
threat model is local network only.

Error Handling and Logging
**************************

Debugging issues in an embedded PWA without network connectivity requires
local error tracking.

Error Capture Strategy
======================

.. code-block:: typescript

   // stores/errors.ts
   import { atom } from 'nanostores';

   interface ErrorEntry {
     timestamp: number;
     type: 'error' | 'warning' | 'info';
     message: string;
     stack?: string;
     context?: Record<string, unknown>;
   }

   const MAX_ERRORS = 100;
   export const $errorLog = atom<ErrorEntry[]>([]);

   export function logError(error: Error, context?: Record<string, unknown>) {
     const entry: ErrorEntry = {
       timestamp: Date.now(),
       type: 'error',
       message: error.message,
       stack: error.stack,
       context,
     };

     $errorLog.set([entry, ...$errorLog.get()].slice(0, MAX_ERRORS));

     // Also persist to localStorage for crash recovery
     persistErrors();
   }

   // Global error handler
   window.addEventListener('error', (event) => {
     logError(event.error || new Error(event.message));
   });

   window.addEventListener('unhandledrejection', (event) => {
     logError(event.reason instanceof Error
       ? event.reason
       : new Error(String(event.reason)));
   });

Error Display
=============

Provide a debug view accessible via settings:

.. code-block:: text

   <!-- DebugLog.svelte -->
   <script>
     import { $errorLog } from '../stores/errors';
   </script>

   <div class="font-mono text-sm">
     {#each $errorLog as entry}
       <div class="border-b py-2">
         <span class="text-gray-500">
           {new Date(entry.timestamp).toLocaleTimeString()}
         </span>
         <span class="text-red-500">{entry.message}</span>
       </div>
     {/each}
   </div>

   <button on:click={() => $errorLog.set([])}>
     Clear Log
   </button>

Export for Support
==================

Allow users to export error logs for support:

.. code-block:: typescript

   function exportErrorLog(): string {
     const log = $errorLog.get();
     return JSON.stringify({
       exported: new Date().toISOString(),
       device: $deviceAddress.get(),
       errors: log,
     }, null, 2);
   }

   function downloadErrorLog() {
     const blob = new Blob([exportErrorLog()], { type: 'application/json' });
     const url = URL.createObjectURL(blob);
     const a = document.createElement('a');
     a.href = url;
     a.download = `roastee-log-${Date.now()}.json`;
     a.click();
   }

This enables users to share debug information without requiring network access
from the embedded device.

Bundle Size Estimates
*********************

Estimated Production Bundle
===========================

.. list-table:: Estimated Bundle Breakdown (gzipped)
   :widths: 35 20 20 25
   :header-rows: 1

   * - Component
     - Typical
     - Max
     - Notes
   * - Svelte runtime
     - ~2 KB
     - 3 KB
     - Svelte 5, compiled
   * - UnoCSS output
     - ~5 KB
     - 10 KB
     - Depends on utility usage
   * - typesafe-i18n + strings
     - ~3 KB
     - 8 KB
     - Grows with translations
   * - Nanostores
     - ~0.5 KB
     - 1 KB
     - Atomic stores
   * - Fusain + cbor-x
     - ~10 KB
     - 12 KB
     - Protocol + CBOR library
   * - Application code
     - ~25 KB
     - 50 KB
     - Components, logic, routing
   * - Service worker
     - ~2 KB
     - 3 KB
     - Manual implementation
   * - SVG icons
     - ~3 KB
     - 8 KB
     - Inline SVG, tree-shaken
   * - Web fonts (optional)
     - ~0 KB
     - 20 KB
     - Icon font if needed

**Build Totals:**

.. list-table:: Build Size Summary
   :widths: 25 20 20 35
   :header-rows: 1

   * - Configuration
     - Typical
     - Max
     - Status vs 150 KB Target
   * - **Embedded (no fonts)**
     - ~50 KB
     - 95 KB
     - ✅ Well under target
   * - **Embedded (with fonts)**
     - ~65 KB
     - 115 KB
     - ✅ Under target
   * - **Full (all features)**
     - ~90 KB
     - 150 KB
     - ⚠️ At target limit

**Notes:**

- **cbor-x:** Required for Fusain protocol. ~8 KB but smaller than alternatives.
- **Icons:** Inline SVGs from Thermoquad design system, tree-shaken per import.
- **Fonts:** Optional. Consider subset icon font only if SVG count exceeds ~20.
- **Max estimates:** Assume worst-case growth; typical builds should be 50-70% of max.

**Conclusion:** The recommended stack achieves **~50-65 KB typical** for embedded
builds, well under the 150 KB target. Maximum estimates (~95-115 KB) still leave
headroom below the 200 KB hard limit.

Comparison with Requirements
============================

.. list-table:: Stack vs Requirements
   :widths: 35 35 30
   :header-rows: 1

   * - Requirement
     - Solution
     - Status
   * - Small bundle size
     - Svelte 5 + UnoCSS
     - ✅ ~50-65 KB typical (115 KB max)
   * - Mobile performance
     - Compiled output, no virtual DOM
     - ✅ <200ms TTI
   * - Modular architecture
     - Vite feature flags, tree-shaking
     - ✅ Compile-time removal
   * - TypeScript
     - Zero runtime cost
     - ✅ Compile-time only
   * - Tailwind-like CSS
     - UnoCSS presetWind
     - ✅ Compatible syntax
   * - Localization
     - typesafe-i18n
     - ✅ Type-safe, ~1 KB
   * - Component architecture
     - Svelte 5
     - ✅ Best-in-class DX
   * - Framework adoption
     - Svelte: growing rapidly
     - ✅ Active community

Alternatives Considered
***********************

SolidJS
=======

**Pros:**

- Fastest benchmarks
- Fine-grained reactivity (signals)
- Smaller ecosystem overhead

**Cons:**

- Larger runtime (~10 KB vs ~2 KB)
- JSX syntax (less concise than Svelte)
- Smaller ecosystem than Svelte

**Verdict:** Excellent option, but Svelte 5's compiler-based approach produces
smaller bundles with comparable performance.

Preact
======

**Pros:**

- React-compatible API
- Smaller than React (~13.7 KB)
- Large ecosystem via React compatibility

**Cons:**

- Still larger than Svelte
- Virtual DOM overhead
- Less optimized than compile-time approach

**Verdict:** Good for React migration, but not optimal for greenfield PWA.

Tailwind CSS
============

**Pros:**

- Excellent documentation
- Huge ecosystem
- Well-known syntax

**Cons:**

- Purge-based approach less efficient
- ~32 KB vs ~4 KB in real comparisons
- PostCSS dependency adds build complexity

**Verdict:** UnoCSS with presetWind provides same syntax with better output.

Proposed Experiments
********************

Before committing to the recommended stack, build minimal prototypes to validate
bundle size claims and developer experience. Each experiment should produce
measurable results.

Experiment 1: Framework Bundle Comparison
=========================================

**Objective:** Verify framework runtime sizes with identical functionality.

**Prototype:** Build a minimal "device dashboard" in each framework:

- Display device connection status
- Show 5 telemetry values (temperature, RPM, etc.)
- Toggle button for heater on/off
- Simple reactive updates every 100ms

**Frameworks to test:**

1. Svelte 5 (recommended)
2. SolidJS (alternative)
3. Preact (React-compatible baseline)

**Measurements:**

.. list-table::
   :widths: 40 60
   :header-rows: 1

   * - Metric
     - Method
   * - Bundle size (min+gzip)
     - ``vite build && gzip -c dist/assets/*.js | wc -c``
   * - Initial load time
     - Lighthouse performance audit
   * - Memory usage
     - Chrome DevTools Memory panel
   * - TTI (Time to Interactive)
     - Lighthouse metric

**Success criteria:** Svelte 5 produces smallest bundle with comparable performance.

Experiment 2: CSS Framework Output
==================================

**Objective:** Compare CSS output sizes with realistic utility usage.

**Prototype:** Style the device dashboard with:

- Responsive layout (mobile-first)
- Dark/light theme toggle
- ~50 unique utility classes (typical component)
- Button, card, input, badge components

**Frameworks to test:**

1. UnoCSS with presetWind (recommended)
2. Tailwind CSS with JIT
3. Vanilla CSS (baseline)

**Measurements:**

.. list-table::
   :widths: 40 60
   :header-rows: 1

   * - Metric
     - Method
   * - CSS output size (min+gzip)
     - ``gzip -c dist/assets/*.css | wc -c``
   * - Build time
     - ``time vite build``
   * - HMR speed
     - Subjective (instant vs noticeable delay)

**Success criteria:** UnoCSS produces <10 KB CSS, significantly smaller than Tailwind.

Experiment 3: i18n Integration
==============================

**Objective:** Validate typesafe-i18n bundle impact and DX.

**Prototype:** Add localization to the dashboard:

- 3 locales: English, German, Spanish
- ~50 translation keys
- Pluralization (e.g., "1 error" vs "5 errors")
- Number/date formatting
- Runtime locale switching

**Measurements:**

.. list-table::
   :widths: 40 60
   :header-rows: 1

   * - Metric
     - Method
   * - Library overhead (gzip)
     - Compare bundle with/without i18n
   * - Per-locale size
     - Size of each locale JSON/TS
   * - Type safety
     - Verify compile-time errors for missing keys
   * - Async loading
     - Network waterfall for lazy-loaded locales

**Success criteria:** <2 KB library overhead, type errors for invalid keys.

Experiment 4: PWA Service Worker Approaches
===========================================

**Objective:** Compare Workbox vs manual service worker for embedded PWA.

**Prototype:** Implement offline caching for the dashboard:

- Cache app shell (HTML, CSS, JS)
- Cache-first strategy for static assets
- Network-first for API calls (simulated)
- Update notification when new version available

**Approaches to test:**

1. Manual service worker (recommended)
2. Vite PWA plugin with generateSW
3. Vite PWA plugin with injectManifest

**Measurements:**

.. list-table::
   :widths: 40 60
   :header-rows: 1

   * - Metric
     - Method
   * - SW + runtime size (gzip)
     - Total size of sw.js and workbox chunks
   * - Offline functionality
     - Manual testing with DevTools offline mode
   * - Update flow
     - Verify skipWaiting and clients.claim behavior
   * - Complexity
     - Lines of code, configuration required

**Success criteria:** Manual SW achieves same functionality at <5 KB vs ~80 KB.

Experiment 5: Full Stack Integration
====================================

**Objective:** Build complete prototype with recommended stack and measure total.

**Prototype:** Minimal Roastee with:

- Svelte 5 + UnoCSS + typesafe-i18n + Nanostores
- WebSocket connection to mock server
- Fusain packet encoding/decoding (use existing package)
- Dashboard, settings, and setup views
- Manual service worker
- Feature flags for embedded vs full build

**Measurements:**

.. list-table::
   :widths: 40 60
   :header-rows: 1

   * - Metric
     - Method
   * - Embedded build size
     - ``FEATURE_SSO=false vite build``
   * - Full build size
     - ``FEATURE_SSO=true vite build``
   * - Tree-shaking effectiveness
     - Compare builds with/without features
   * - Lighthouse PWA score
     - Chrome Lighthouse audit
   * - Mobile performance
     - Test on mid-range Android device

**Success criteria:**

- Embedded build: <95 KB typical, <115 KB max (target: 75% of 150 KB = 112 KB)
- Full build: <115 KB typical, <150 KB max
- Lighthouse PWA score: >90
- Mobile TTI: <200ms (cached)

Experiment 6: Alternative Stack Comparison
==========================================

**Objective:** Validate recommendation by testing leading alternative.

**Prototype:** Build same full integration with alternative stack:

- SolidJS + Tailwind CSS + i18next + Zustand
- Same functionality as Experiment 5

**Measurements:** Same as Experiment 5.

**Purpose:** Ensure recommendation is based on measured data, not assumptions.
If alternative stack performs comparably, document trade-offs for future reference.

Experiment Execution Plan
=========================

.. list-table::
   :widths: 15 45 40
   :header-rows: 1

   * - Order
     - Experiment
     - Blocking Dependencies
   * - 1
     - Framework Bundle Comparison
     - None (can start immediately)
   * - 2
     - CSS Framework Output
     - None (parallel with #1)
   * - 3
     - i18n Integration
     - After #1 (needs framework choice)
   * - 4
     - PWA Service Worker
     - After #1 (needs framework choice)
   * - 5
     - Full Stack Integration
     - After #1-4 (combines all)
   * - 6
     - Alternative Stack
     - After #5 (comparison baseline)

Experiments 1-2 can run in parallel. Results inform whether to proceed with
recommended stack or pivot to alternatives.

Prototype Location
==================

Prototypes should be created in the Roastee monorepo:

.. code-block:: none

   roastee/
   └── packages/
       ├── fusain/                    # ✅ Exists
       └── experiments/               # New - prototype workspace
           ├── exp1-framework/        # Framework comparison
           ├── exp2-css/              # CSS framework comparison
           ├── exp3-i18n/             # i18n integration
           ├── exp4-pwa/              # Service worker approaches
           ├── exp5-full-stack/       # Recommended stack
           └── exp6-alternative/      # Alternative stack

Each experiment is a standalone Vite project with its own ``package.json``.
After validation, successful patterns migrate to ``packages/ui`` and
``packages/pwa``.

Implementation Roadmap
**********************

Phase 1: UI Package Foundation
==============================

1. Create ``packages/ui`` with Svelte 5 + TypeScript
2. Configure UnoCSS with presetWind
3. Set up typesafe-i18n with initial locales (en, de)
4. Create base components (Button, Input, Card)
5. Configure Nanostores for device state

Phase 2: PWA Application
========================

1. Create ``packages/pwa`` with Vite configuration
2. Integrate fusain package for protocol communication
3. Implement core views (Dashboard, Settings)
4. Add WebSocket/Web Bluetooth transport abstraction
5. Implement manual service worker

Phase 3: Build Variants
=======================

1. Configure feature flags for embedded vs full builds
2. Test tree-shaking effectiveness
3. Optimize bundle for embedded deployment
4. Verify gzip output meets <200 KB target

Phase 4: Integration Testing
============================

1. Test PWA serving from Slate firmware
2. Verify offline functionality
3. Performance testing on target mobile devices
4. Localization testing with RTL languages

References
**********

.. [1] Building PWAs with Vite and SvelteKit
   https://johal.in/building-pwas-with-vite-and-sveltekit-offline-first-strategies-using-python-apis-in-2025/

.. [2] Why Svelte 5 is Redefining Frontend Performance in 2025
   https://dev.to/krish_kakadiya_5f0eaf6342/why-svelte-5-is-redefining-frontend-performance-in-2025-a-deep-dive-into-reactivity-and-bundle-5200

.. [3] Size Comparison: Vue vs Svelte vs Solid
   https://gist.github.com/ryansolid/71e2b160df4db33fcca2862355377983

.. [4] Next-Gen Reactivity: Preact SolidJS Signals vs Svelte 5 Runes
   https://leapcell.io/blog/next-gen-reactivity-rethink-preact-solidjs-signals-vs-svelte-5-runes

.. [5] Why UnoCSS?
   https://unocss.dev/guide/why

.. [6] SvelteKit Service Workers Documentation
   https://kit.svelte.dev/docs/service-workers

**Additional Resources:**

- typesafe-i18n Documentation: https://typesafe-i18n.pages.dev/
- Nanostores GitHub: https://github.com/nanostores/nanostores
- Vite PWA Plugin Documentation: https://vite-pwa-org.netlify.app/
- Frontend Framework Performance Benchmark 2025-2026: https://www.frontendtools.tech/blog/best-frontend-frameworks-2025-comparison

Appendix A: Minimum Viable Stack
********************************

For the absolute smallest bundle, this stack can be reduced to:

.. code-block:: none

   Component                 Typical    Max
   ─────────────────────────────────────────
   Svelte 5 (compiled)       ~2 KB      3 KB
   UnoCSS                    ~4 KB      6 KB
   Fusain + cbor-x           ~10 KB     12 KB
   Manual service worker     ~2 KB      3 KB
   SVG icons                 ~2 KB      5 KB
   Application code          ~10 KB     20 KB
   ─────────────────────────────────────────
   Total                     ~30 KB     49 KB

This excludes i18n (hardcoded strings), external state management (Svelte
stores only), and web fonts. Suitable for prototype or single-language
deployments.

**Target:** 75% of max = ~37 KB for minimum viable builds.

Appendix B: TypeScript Consideration
************************************

TypeScript adds **zero runtime overhead**. All type checking happens at compile
time, and types are stripped from the production bundle.

The only consideration is build tooling:

.. list-table:: TypeScript Impact
   :widths: 30 70
   :header-rows: 1

   * - Aspect
     - Impact
   * - Runtime size
     - None (types stripped)
   * - Build time
     - Slightly longer (type checking)
   * - Bundle size
     - None
   * - DX
     - Significantly better (autocomplete, errors)

**Recommendation:** Always use TypeScript. The benefits far outweigh the
minimal build time increase.

Appendix C: Browser Support
***************************

Target browsers for Roastee PWA:

.. list-table:: Browser Support Matrix
   :widths: 25 25 50
   :header-rows: 1

   * - Browser
     - Minimum Version
     - Notes
   * - Chrome (Android)
     - 90+
     - Primary target, Web Bluetooth
   * - Safari (iOS)
     - 15+
     - No Web Bluetooth, WiFi only
   * - Chrome (Desktop)
     - 90+
     - Development and secondary use
   * - Firefox
     - 100+
     - Limited Web Bluetooth (flag)
   * - Edge
     - 90+
     - Chromium-based, full support

Build target: ES2022 (supported by all target browsers).
