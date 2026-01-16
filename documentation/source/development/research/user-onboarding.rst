User Onboarding
###############

:Date: 2026-01-11
:Author: Thermoquad
:Status: Research Complete

.. note::

   **Development Status: Research Phase**

   This document captures research findings and recommendations. Implementation
   has not yet begun. The findings here will guide future development of
   user onboarding for Thermoquad heaters.

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

This research investigates user onboarding for Thermoquad heaters, with the
paramount goal: **A user must be able to fully use a Thermoquad heater from
their mobile phone or computer without an internet connection.**

**Key Findings:**

- WiFi AP mode with captive portal is a viable onboarding method for Block (Slate)
- Zephyr provides WiFi AP mode, DHCP server, mDNS, and HTTP server capabilities
- Open networks during setup are acceptable if a device password is required before operation
- Web Bluetooth API enables PWA-to-BLE communication on Android/Chrome (not iOS Safari)
- Roastee PWA can be served directly from Block and installed for offline use
- BLE pairing should use LESC with application-layer authentication for security

**Recommendations:**

1. Use WiFi AP mode with captive portal for initial device setup
2. Require device password before any heater operations
3. Serve Roastee PWA directly from Block with service worker for offline support
4. Support optional BLE mode with time-limited pairing window
5. Use mDNS (``thermoquad-XXXX.local``) for device discovery
6. OTA updates via Roastee (works with WiFi station mode, AP mode, or BLE)

System Architecture
*******************

Hardware Components
===================

This research assumes the following hardware configuration:

.. list-table:: Hardware Configuration
   :widths: 20 30 50
   :header-rows: 1

   * - Device
     - Hardware
     - Role
   * - Helios
     - Hades (RP2350-based ICU)
     - Burner control, Fusain slave
   * - Slate
     - Block (RP2350 + RM2 module)
     - WiFi/BT bridge, Fusain master
   * - Roastee
     - User's phone/computer
     - PWA client interface

**Block Hardware Features:**

- Raspberry Pi Radio Module 2 (RM2) with CYW43439 [1]_
- WiFi 4 (802.11n, 2.4 GHz single-band)
- Bluetooth 5.2 (Classic + BLE)
- SoftAP support (up to 4 clients)
- 4 RGB status LEDs for user feedback
- IP68 rated enclosure

**RM2 Module Specifications:**

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - Parameter
     - Value
   * - WiFi Standard
     - 802.11b/g/n (WiFi 4)
   * - Frequency
     - 2.4 GHz single-band
   * - Max Data Rate
     - 96 Mbps (PHY rate)
   * - Bluetooth
     - 5.2 (Classic + LE)
   * - Power (Active RX)
     - ~43 mA at -50 dBm
   * - Power (Active TX)
     - ~271 mA at 16 dBm
   * - Power (Sleep)
     - ~1.19 mA (PM1 DTIM1)
   * - Dimensions
     - 14.5 x 16.7 x 2.6 mm

Device Roles
============

In the Fusain protocol architecture:

- **Helios** (ICU): Slave device, receives commands, sends telemetry
- **Slate** (Block): Router/Controller, bridges user interfaces to ICU
- **Roastee** (PWA): Client, user interaction and control

The Block acts as the central integration point, providing:

- WiFi access point for initial setup
- WiFi station mode for network integration
- Bluetooth LE for direct device communication
- WebSocket server for Roastee communication
- Fusain bridge to Helios ICU

Onboarding Flow
***************

Overview
========

.. code-block:: none

   ┌─────────────────────────────────────────────────────────────────┐
   │                      USER ONBOARDING FLOW                       │
   └─────────────────────────────────────────────────────────────────┘

   1. POWER ON
      ├── Helios boots, connects to Slate via Fusain
      ├── Slate detects unconfigured state
      └── Slate activates WiFi AP mode, LEDs indicate setup mode

   2. WIFI CONNECT
      ├── User sees "Thermoquad-XXXX" network (open)
      ├── User connects from phone/laptop
      └── Captive portal auto-opens (or user navigates to portal)

   3. CAPTIVE PORTAL
      ├── Slate serves configuration UI
      ├── User sets device password (REQUIRED)
      └── User configures WiFi (AP password OR join network)

   4. NETWORK TRANSITION
      ├── Slate applies new WiFi settings
      ├── User reconnects to configured network
      └── Slate reachable via mDNS: thermoquad-XXXX.local

   5. PWA INSTALLATION
      ├── User visits http://thermoquad-XXXX.local
      ├── Roastee PWA served from Slate
      ├── User prompted to install PWA
      └── PWA cached for offline use

   6. AUTHENTICATION
      ├── Roastee prompts for device password
      ├── Password persisted in PWA settings
      └── User can now control heater

   7. OPTIONAL: BLE SETUP
      ├── User requests BLE pairing in Roastee settings
      ├── Slate enables BLE advertising for limited time
      ├── Roastee initiates Web Bluetooth pairing
      └── Secure BLE connection established

Step 1: Power On
================

When a new Thermoquad heater is powered on:

1. **Helios** boots and initializes hardware (firmware defaults sufficient)
2. **Slate** boots and establishes Fusain connection with Helios
3. **Slate** checks persistent storage for configuration state
4. If unconfigured, Slate enters **Setup Mode**:
   - Activates WiFi in AP mode
   - SSID: ``Thermoquad-XXXX`` (XXXX from RP2350 unique ID)
   - Open network (no password initially)
   - Starts DHCP server, DNS server, HTTP server
   - RGB LEDs display distinct "setup mode" pattern

**LED Pattern for Setup Mode:** (Example)

- Slow blue pulse: WiFi AP active, awaiting connection
- Fast blue pulse: Client connected, portal active
- Green: Configuration complete
- Red: Error state

Step 2: WiFi Connect
====================

The user connects their phone or computer to the ``Thermoquad-XXXX`` network.

**Why Open Network Initially?**

The initial network is open (no password) because:

- No pre-printed credentials on device (user requirement)
- Eliminates need for default passwords like "admin"
- Simpler onboarding for non-technical users
- Security enforced at application layer (device password required)

**Security Mitigation:**

- Captive portal only allows configuration, not heater operation
- Device password MUST be set before any Fusain commands
- Configuration session has timeout
- LEDs indicate when device is in setup mode (physical awareness)

Step 3: Captive Portal
======================

Modern devices automatically detect captive portals and display them. [2]_

**Detection Mechanism:**

Operating systems probe specific URLs to detect captive portals:

.. list-table:: Captive Portal Detection URLs
   :widths: 20 80
   :header-rows: 1

   * - Platform
     - Detection URLs
   * - iOS/macOS
     - ``captive.apple.com``, ``www.apple.com/library/test/success.html``
   * - Android
     - ``connectivitycheck.gstatic.com``, ``clients3.google.com``
   * - Windows
     - ``www.msftconnecttest.com/connecttest.txt``
   * - Firefox
     - ``detectportal.firefox.com``

**Slate Implementation:**

Slate's DNS server returns its own IP for ALL DNS queries, redirecting all
HTTP traffic to the captive portal. When the device probes its detection URL:

1. DNS resolves to Slate's IP (e.g., 192.168.4.1)
2. HTTP request redirected to configuration page
3. Device displays captive portal UI

**DHCP Option 114 (Modern Standard):**

For iOS 14+ and macOS Big Sur+, Slate can advertise captive portal presence
via DHCP Option 114, providing a better user experience. [3]_

**Configuration Options:**

The captive portal presents two mutually exclusive WiFi configurations:

**Option A: Secure AP Mode**

- Keep SSID as ``Thermoquad-XXXX`` (or allow customization)
- User MUST set a WiFi password (WPA2/WPA3)
- Block continues as access point
- Suitable for: Direct connection, no existing network

**Option B: Station Mode (Join Network)**

- User selects from scanned networks
- User enters network password
- Optional: Set custom hostname (default: ``thermoquad-XXXX``)
- Block connects as client to existing network
- Suitable for: Home/shop integration, multiple devices

**Required for Both Options:**

- **Device Password**: User MUST set a password for Roastee authentication
- This password protects the HTTP/WebSocket API
- Stored securely in Slate's persistent storage (NVS)

Step 4: Network Transition
==========================

After configuration:

1. Slate saves settings to persistent storage (NVS)
2. Slate applies new WiFi configuration
3. If AP mode: Restarts AP with password protection
4. If Station mode: Connects to configured network
5. Slate starts mDNS responder: ``thermoquad-XXXX.local``
6. LEDs indicate "configured" state

**User Reconnection:**

- AP mode: User reconnects with new WiFi password
- Station mode: User's device auto-reconnects to home network

**Device Discovery:**

Slate registers with mDNS, making it discoverable as:

- ``thermoquad-XXXX.local`` (default)
- ``<custom-hostname>.local`` (if user configured)

Step 5: PWA Installation
========================

User navigates to ``http://thermoquad-XXXX.local`` in their browser.

**Roastee PWA Serving:**

Slate's HTTP server serves the Roastee PWA directly:

- HTML, CSS, JavaScript (gzip compressed)
- Web App Manifest (``manifest.json``)
- Service Worker for offline caching
- Static assets embedded in Slate firmware

**PWA Requirements:** [4]_

For a PWA to be installable, it must have:

1. Valid Web App Manifest with required fields
2. Registered Service Worker with offline fallback
3. Served over HTTPS (or localhost/``.local`` for development)

**Note on HTTPS:**

Local network PWAs present a challenge for HTTPS. Options:

1. **Self-signed certificate**: Works but requires user to accept warning
2. **mDNS with ``.local``**: Some browsers allow service workers on ``.local``
3. **HTTP for local**: Chrome allows service workers on ``localhost`` and local IPs

**Recommendation:** Use HTTP for local network operation. The lack of internet
requirement means HTTPS certificate validation is not the primary security
mechanism—the device password is.

**Install Prompt:**

On Android/Chrome, the ``beforeinstallprompt`` event allows custom install UI.
On iOS, users must manually "Add to Home Screen" from Safari share menu. [5]_

**Offline Capability:**

Once installed, Roastee's service worker caches:

- App shell (HTML, CSS, JS)
- Static assets (icons, fonts)
- API responses (where applicable)

The PWA works offline, connecting to Slate when available.

Step 6: Authentication
======================

When Roastee connects to Slate:

1. WebSocket connection established
2. Roastee prompts for device password (first time)
3. Password sent to Slate for verification
4. On success: Auth token returned, stored in PWA settings
5. Subsequent connections use stored token

**Authentication Flow:**

.. code-block:: none

   Roastee                           Slate
      │                                │
      │──── WebSocket Connect ────────>│
      │                                │
      │<─── Auth Required ─────────────│
      │                                │
      │──── Password ─────────────────>│
      │                                │
      │<─── Auth Token ────────────────│
      │                                │
      │──── Commands (with token) ────>│
      │                                │
      │<─── Telemetry ─────────────────│

**Token Management:**

- Tokens are session-based or long-lived (configurable)
- Password can be changed via authenticated API
- Logout invalidates current token

Step 7: Optional BLE Setup
==========================

After WiFi setup, users can optionally enable BLE communication.

**Why BLE as Secondary?**

- WiFi provides better range and throughput
- WiFi works with any browser
- BLE is useful for: Battery-powered Luna, reduced power consumption
- Web Bluetooth API has limited browser support

**BLE Pairing Security:**

BLE pairing MUST be secure. The approach:

1. BLE advertising disabled by default
2. User requests BLE pairing in Roastee (over WiFi)
3. Slate enables BLE advertising for limited time (e.g., 60 seconds)
4. Roastee initiates Web Bluetooth pairing
5. LESC pairing with application-layer authentication
6. After timeout, BLE advertising disabled

**Web Bluetooth API Support:** [6]_

.. list-table:: Web Bluetooth Browser Support
   :widths: 30 70
   :header-rows: 1

   * - Platform
     - Support
   * - Chrome (Android)
     - Full support
   * - Chrome (Windows/macOS)
     - Full support
   * - Edge
     - Full support
   * - Safari (iOS/macOS)
     - **Not supported**
   * - Firefox
     - Experimental (flag required)

**iOS Limitation:**

Web Bluetooth is not available on iOS Safari. For iOS BLE support, options:

1. Use WiFi-only on iOS (recommended)
2. Third-party browser (Bluefy) with Web Bluetooth support
3. Native iOS app (outside PWA scope)

**Recommendation:** Design Roastee to gracefully degrade on iOS, using WiFi
as the primary communication method.

Zephyr Implementation
*********************

WiFi AP Mode
============

Zephyr supports WiFi AP mode (SoftAP) with the ``wifi_mgmt`` API. [7]_

**Kconfig Options:**

.. code-block:: kconfig

   # WiFi support
   CONFIG_WIFI=y
   CONFIG_WIFI_NM=y

   # AP mode
   CONFIG_WIFI_NM_WPA_SUPPLICANT=y
   CONFIG_NET_L2_WIFI_MGMT=y

   # Networking
   CONFIG_NETWORKING=y
   CONFIG_NET_IPV4=y
   CONFIG_NET_TCP=y
   CONFIG_NET_UDP=y

**AP Mode Initialization:**

.. code-block:: c

   #include <zephyr/net/wifi_mgmt.h>

   static struct wifi_connect_req_params ap_params = {
       .ssid = "Thermoquad-XXXX",
       .ssid_length = 16,
       .channel = WIFI_CHANNEL_ANY,
       .security = WIFI_SECURITY_TYPE_NONE,  // Open initially
   };

   int start_ap_mode(void)
   {
       struct net_if *iface = net_if_get_default();
       return net_mgmt(NET_REQUEST_WIFI_AP_ENABLE, iface,
                       &ap_params, sizeof(ap_params));
   }

**Note:** The CYW43439 supports up to 4 simultaneous clients in AP mode.

DHCP Server
===========

Zephyr includes a DHCPv4 server for AP mode. [8]_

**Kconfig Options:**

.. code-block:: kconfig

   CONFIG_NET_DHCPV4_SERVER=y

**Configuration:**

.. code-block:: c

   #include <zephyr/net/dhcpv4_server.h>

   static struct in_addr base_addr = { .s_addr = htonl(0xC0A80402) };  // 192.168.4.2

   int start_dhcp_server(void)
   {
       struct net_if *iface = net_if_get_default();
       return net_dhcpv4_server_start(iface, &base_addr);
   }

**DHCP Option 114 (Captive Portal):**

For modern captive portal detection, implement DHCP Option 114:

.. code-block:: c

   // Option 114: Captive Portal URI
   // Value: "http://192.168.4.1/portal"

This tells iOS 14+ devices about the captive portal URL directly.

DNS Server
==========

For captive portal functionality, Slate needs a DNS server that returns
its own IP for all queries (DNS hijacking).

**Note:** Zephyr does not include a DNS server out-of-box. Implementation
options:

1. **Custom minimal DNS server**: Respond to all A queries with AP IP
2. **Port existing lightweight DNS**: Adapt from lwIP or similar

**Minimal DNS Server (Concept):**

.. code-block:: c

   // Pseudo-code for captive portal DNS
   void handle_dns_query(struct dns_query *query)
   {
       // Respond to ALL queries with our IP
       struct dns_response response = {
           .name = query->name,
           .type = DNS_TYPE_A,
           .class = DNS_CLASS_IN,
           .ttl = 60,
           .ip = CAPTIVE_PORTAL_IP,  // e.g., 192.168.4.1
       };
       send_dns_response(&response);
   }

mDNS Responder
==============

Zephyr includes mDNS responder support. [9]_

**Kconfig Options:**

.. code-block:: kconfig

   CONFIG_MDNS_RESPONDER=y
   CONFIG_NET_HOSTNAME="thermoquad-XXXX"

**mDNS Registration:**

The device automatically responds to ``<hostname>.local`` queries when mDNS
is enabled. Set ``CONFIG_NET_HOSTNAME`` to the desired name.

HTTP Server
===========

Zephyr's HTTP server library supports static files, dynamic content, and
WebSockets. [10]_

**Kconfig Options:**

.. code-block:: kconfig

   CONFIG_HTTP_SERVER=y
   CONFIG_HTTP_SERVER_WEBSOCKET=y
   CONFIG_HTTP_SERVER_RESOURCE_WILDCARD=y

**Serving PWA Static Files:**

Static files can be embedded at compile time:

.. code-block:: c

   #include <zephyr/net/http/server.h>

   // Embedded gzip-compressed files
   extern const uint8_t index_html_gz[];
   extern const size_t index_html_gz_len;

   static struct http_resource_detail_static index_resource = {
       .common = {
           .type = HTTP_RESOURCE_TYPE_STATIC,
           .content_type = "text/html",
           .content_encoding = "gzip",
       },
       .static_data = index_html_gz,
       .static_data_len = index_html_gz_len,
   };

   HTTP_RESOURCE_DEFINE(index, &index_resource, "/");

**WebSocket Support:**

For Roastee communication:

.. code-block:: c

   static int ws_handler(struct http_client_ctx *client,
                         enum http_data_status status,
                         const uint8_t *data, size_t len)
   {
       // Handle WebSocket messages (Fusain protocol)
       return 0;
   }

   static struct http_resource_detail_websocket ws_resource = {
       .common = { .type = HTTP_RESOURCE_TYPE_WEBSOCKET },
       .cb = ws_handler,
   };

   HTTP_RESOURCE_DEFINE(ws, &ws_resource, "/ws");

Bluetooth LE
============

Zephyr provides comprehensive BLE support with security options. [11]_

**Kconfig Options:**

.. code-block:: kconfig

   CONFIG_BT=y
   CONFIG_BT_PERIPHERAL=y
   CONFIG_BT_SMP=y
   CONFIG_BT_SMP_SC_ONLY=y  # LE Secure Connections only
   CONFIG_BT_BONDABLE=y
   CONFIG_BT_SETTINGS=y     # Persist bonding info

**Security Configuration:**

For secure BLE pairing with application-layer auth:

.. code-block:: c

   #include <zephyr/bluetooth/bluetooth.h>
   #include <zephyr/bluetooth/conn.h>

   static struct bt_conn_auth_cb auth_cb = {
       .passkey_display = NULL,      // No display
       .passkey_confirm = NULL,      // No numeric comparison
       .cancel = auth_cancel,
       .pairing_complete = pairing_complete,
   };

   // Use "Just Works" pairing (no MITM) but require app-layer auth
   // BLE connection is encrypted, app password provides authentication

**Time-Limited Advertising:**

.. code-block:: c

   static struct k_timer pairing_timer;

   void enable_ble_pairing(void)
   {
       // Start advertising
       bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), NULL, 0);

       // Start 60-second timeout
       k_timer_start(&pairing_timer, K_SECONDS(60), K_NO_WAIT);
   }

   void pairing_timeout(struct k_timer *timer)
   {
       // Stop advertising
       bt_le_adv_stop();
   }

PWA Implementation
******************

Roastee Architecture
====================

Roastee is a Progressive Web App built with TypeScript.

.. note::

   For detailed technology stack decisions, see :doc:`roastee-web-stack`.
   That document covers framework selection, bundle size optimization, and
   build configuration based on measured experiment results.

**Technology Stack:**

- **Framework:** Svelte 5 (smallest runtime, see :doc:`roastee-web-stack`)
- **Build Tool:** Vite (fast builds, optimal tree-shaking)
- **State Management:** Nanostores (framework-agnostic, ~500 bytes)
- **CSS:** UnoCSS with Tailwind preset (on-demand generation)
- **WebSocket:** Native WebSocket API
- **Web Bluetooth:** Native Web Bluetooth API
- **PWA:** Manual service worker (smaller than Workbox)

**Key Components:**

.. code-block:: none

   roastee/
   ├── src/
   │   ├── api/
   │   │   ├── websocket.ts      # WebSocket connection management
   │   │   ├── bluetooth.ts      # Web Bluetooth integration
   │   │   └── fusain.ts         # Fusain protocol encoding/decoding
   │   ├── components/
   │   │   ├── Dashboard.tsx     # Main control interface
   │   │   ├── Settings.tsx      # Device/app settings
   │   │   └── Setup.tsx         # BLE pairing UI
   │   ├── stores/
   │   │   ├── device.ts         # Device state (telemetry, config)
   │   │   └── auth.ts           # Authentication state
   │   └── sw.ts                 # Service worker
   ├── public/
   │   ├── manifest.json         # Web App Manifest
   │   └── icons/                # App icons
   └── vite.config.ts

Web App Manifest
================

.. code-block:: json

   {
     "name": "Roastee - Thermoquad Controller",
     "short_name": "Roastee",
     "description": "Control your Thermoquad heater",
     "start_url": "/",
     "display": "standalone",
     "background_color": "#1a1a1a",
     "theme_color": "#ff6b00",
     "icons": [
       {
         "src": "/icons/icon-192.png",
         "sizes": "192x192",
         "type": "image/png"
       },
       {
         "src": "/icons/icon-512.png",
         "sizes": "512x512",
         "type": "image/png"
       }
     ]
   }

Service Worker Strategy
=======================

**Caching Strategy:** [12]_

- **Cache-First** for app shell (HTML, CSS, JS, icons)
- **Network-First** for API data (telemetry, device state)
- **Stale-While-Revalidate** for PWA updates

.. code-block:: typescript

   // Service worker registration
   if ('serviceWorker' in navigator) {
     navigator.serviceWorker.register('/sw.js')
       .then(reg => console.log('SW registered'))
       .catch(err => console.error('SW registration failed', err));
   }

   // sw.ts - Cache app shell on install
   self.addEventListener('install', (event) => {
     event.waitUntil(
       caches.open('roastee-v1').then((cache) => {
         return cache.addAll([
           '/',
           '/index.html',
           '/app.js',
           '/app.css',
           '/manifest.json',
         ]);
       })
     );
   });

Web Bluetooth Integration
=========================

.. code-block:: typescript

   // bluetooth.ts
   const THERMOQUAD_SERVICE_UUID = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
   const FUSAIN_CHAR_UUID = 'yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy';

   export async function connectBLE(): Promise<BluetoothDevice> {
     const device = await navigator.bluetooth.requestDevice({
       filters: [{ services: [THERMOQUAD_SERVICE_UUID] }],
     });

     const server = await device.gatt?.connect();
     const service = await server?.getPrimaryService(THERMOQUAD_SERVICE_UUID);
     const characteristic = await service?.getCharacteristic(FUSAIN_CHAR_UUID);

     // Subscribe to notifications for telemetry
     await characteristic?.startNotifications();
     characteristic?.addEventListener('characteristicvaluechanged', handleData);

     return device;
   }

   function handleData(event: Event) {
     const value = (event.target as BluetoothRemoteGATTCharacteristic).value;
     // Decode Fusain packet from value.buffer
   }

**BLE-to-WiFi Transition:**

When user requests BLE pairing from Roastee:

1. Roastee sends ``ENABLE_BLE_PAIRING`` command over WiFi
2. Slate starts BLE advertising with timeout
3. Roastee initiates Web Bluetooth device request
4. User selects device from browser prompt
5. GATT connection established
6. Roastee can now communicate over BLE

Security Considerations
***********************

Open Network Risk Assessment
============================

**Risk:** Malicious actor connects during setup

**Mitigations:**

1. **Physical proximity required**: Must be near heater to see LEDs
2. **No heater control**: Captive portal only allows configuration
3. **Device password required**: Must set password to enable control
4. **Session timeout**: Setup mode times out after inactivity
5. **Visual indicator**: LEDs show when device is in setup mode

**Residual Risk:** Acceptable for consumer IoT device

Alternative: Wi-Fi Easy Connect (DPP)
=====================================

Wi-Fi Alliance's Device Provisioning Protocol (DPP) provides secure
onboarding via QR code scanning. [13]_

**Pros:**

- Cryptographically secure (ECDH key exchange)
- No open network phase
- Industry standard

**Cons:**

- Requires QR code on device (violates "no markings" requirement)
- Limited device support
- More complex implementation

**Recommendation:** DPP is not suitable for this use case due to the
requirement for no device markings. The open network with mandatory
device password approach is acceptable.

BLE Security
============

**Pairing Mode:** LE Secure Connections (LESC) with "Just Works"

**Why "Just Works"?**

- Block has no display or input (cannot show passkey)
- Numeric comparison requires display on both devices
- Application-layer password provides authentication

**Security Layers:**

1. **BLE Encryption:** LESC provides encrypted link
2. **Application Auth:** Device password required for commands
3. **Time-Limited Pairing:** Advertising only enabled on user request

**MITM Considerations:**

"Just Works" is vulnerable to MITM during pairing. Mitigations:

- Pairing only enabled by authenticated user (over WiFi)
- Short pairing window (60 seconds)
- Physical proximity required
- LED indication during pairing mode

Factory Reset and Password Recovery
===================================

Users may forget their device password. Factory reset provides recovery.

See :doc:`rp2350-flash-usage` for detailed NVS implementation.

**Reset Methods by Device:**

.. list-table::
   :widths: 20 30 50
   :header-rows: 1

   * - Device
     - Method
     - Notes
   * - Block (Slate)
     - Button hold during boot
     - Hold 10+ seconds, LED confirms reset
   * - Block (Slate)
     - Roastee command
     - Requires current authentication
   * - Hades (Helios)
     - Jumper pads
     - Short during boot (service access required)
   * - Hades (Helios)
     - Fusain command
     - Via authenticated Slate

**Block Factory Reset Flow:**

1. Power off Block
2. Hold reset button
3. Power on while holding button
4. LEDs show reset pattern (e.g., rapid red blink)
5. After 10 seconds, LEDs turn solid (reset triggered)
6. Release button
7. Block reboots into setup mode (open AP, no password)

**Hades Factory Reset Flow:**

Hades resides inside the heater enclosure with no external buttons. Two
reset options are available:

*Option 1: Jumper Pads (Hardware)*

For service technicians or advanced users:

1. Disconnect heater from power
2. Open heater enclosure to access Hades PCB
3. Locate factory reset pads (labeled RST/GND, 2.54mm spacing)
4. Optionally connect to UART0 (shell/logging port) for confirmation
5. Place jumper or short pads with tweezers
6. Power on heater while pads are shorted
7. Confirm reset via UART0 log: ``No persisted configuration found, using defaults``
8. Remove jumper
9. Close enclosure

**UART0 Confirmation:**

Connecting to Hades' UART0 port provides direct confirmation of reset success
without needing the full system operational. This is useful for bench testing
or when Slate is not available.

*Option 2: Remote Reset via Slate (Software)*

If Slate is still accessible:

1. Open Roastee, authenticate with Slate
2. Navigate to Settings → Advanced → Reset Helios
3. Confirm reset action
4. Slate sends ``FACTORY_RESET`` command via Fusain
5. Helios clears configuration and reboots
6. Helios reconnects with firmware defaults

**What Gets Reset:**

- Device password (Block only - Helios has no password)
- WiFi configuration
- Custom hostname
- BLE bonding information
- User preferences

**What Gets Preserved:**

- Firmware
- Hardware calibration data (if any)

**Security Consideration:**

Factory reset on Block returns the device to open AP mode, allowing anyone
nearby to reconfigure it. This is acceptable because:

- Physical access to power cycle is required
- Heater cannot operate without reconfiguration
- New owner can set their own password

**Roastee Handling:**

When Roastee detects a factory-reset device:

1. Stored auth token becomes invalid
2. Roastee prompts: "Device was reset. Set new password."
3. User completes setup flow again
4. New credentials stored in Roastee

OTA Update Considerations
*************************

OTA updates require internet connectivity. This section addresses how the
onboarding architecture supports firmware updates for both Slate and Helios.

See :doc:`rp2350-flash-usage` for detailed flash storage and MCUboot analysis.

Internet Connectivity Scenarios
===============================

**Scenario A: WiFi Station Mode (Internet-Connected Network)**

If the user configured Slate in station mode and connected to an
internet-accessible network:

1. Roastee checks for updates via internet
2. Roastee notifies user of available update
3. User initiates update in Roastee
4. Slate downloads firmware directly from update server
5. Firmware written to ``slot1_partition`` via MCUboot

This is the simplest OTA scenario.

**Scenario B: WiFi AP Mode (No Internet)**

If the user configured Slate in AP mode (isolated network):

1. User opens Roastee on an internet-connected device
2. Roastee downloads firmware update to device storage
3. User connects to Slate's AP network
4. Roastee transfers firmware to Slate over WebSocket
5. Slate writes firmware to ``slot1_partition``

**Scenario C: BLE with Internet-Connected Device**

If user prefers BLE communication:

1. User opens Roastee on internet-connected phone
2. Roastee downloads firmware update
3. Roastee transfers firmware to Slate over BLE
4. Slate writes firmware to ``slot1_partition``

This scenario works even if Slate has no WiFi connectivity.

Firmware Storage Architecture
=============================

Based on the :doc:`rp2350-flash-usage` research:

**Self-Update (Slate updating itself):**

- Firmware written directly to ``slot1_partition``
- No filesystem needed (NVS/littlefs not used for firmware)
- MCUboot handles validation and slot swap on reboot

**Proxy Update (Slate updating Helios):**

- Slate receives Helios firmware over WiFi or BLE
- Slate buffers firmware in littlefs: ``/lfs/helios_update.bin``
- Slate transfers firmware to Helios over Fusain in chunks
- Helios writes chunks to its ``slot1_partition``

.. list-table:: OTA Storage Requirements
   :widths: 25 25 50
   :header-rows: 1

   * - Update Type
     - Storage Location
     - Size Required
   * - Slate self-update
     - slot1_partition
     - ~400-500 KB (current firmware size)
   * - Helios proxy buffer
     - littlefs on Slate
     - ~150 KB (Helios firmware + margin)
   * - Update metadata
     - NVS
     - < 1 KB (version, status flags)

OTA via Roastee PWA
===================

Roastee acts as the update orchestrator:

**Update Check Flow:**

.. code-block:: none

   Roastee                    Internet                    Slate
      │                          │                          │
      │── Check for updates ────>│                          │
      │<── Update manifest ──────│                          │
      │                          │                          │
      │── Download firmware ────>│                          │
      │<── Firmware binary ──────│                          │
      │                          │                          │
      │── Transfer firmware ───────────────────────────────>│
      │                          │                          │
      │<── Update status ──────────────────────────────────│

**Firmware Transfer Protocol:**

For transferring firmware from Roastee to Slate:

1. **WebSocket (WiFi):** Chunked binary transfer with acknowledgment
2. **BLE (GATT):** MTU-sized chunks with flow control

**Chunk Size Recommendations:**

- WebSocket: 4-16 KB chunks (balance between overhead and memory)
- BLE: 512 bytes (after MTU negotiation, typically 517 byte MTU)

**Transfer Integrity:**

- Each chunk includes sequence number and CRC
- Slate validates complete image before committing
- Resume support for interrupted transfers

Roastee Offline Update Cache
============================

Since Roastee is a PWA with offline capability:

1. User checks for updates while online
2. Roastee downloads and caches firmware in IndexedDB
3. User can later apply update even without internet
4. Useful for: Field updates, intermittent connectivity

**Implementation:**

.. code-block:: typescript

   // Cache firmware update in IndexedDB
   async function cacheFirmwareUpdate(
     deviceType: 'slate' | 'helios',
     version: string,
     firmware: ArrayBuffer
   ) {
     const db = await openDB('roastee-updates', 1);
     await db.put('firmware', {
       deviceType,
       version,
       firmware,
       downloadedAt: Date.now(),
     });
   }

   // Check for cached update
   async function getCachedUpdate(deviceType: string): Promise<ArrayBuffer | null> {
     const db = await openDB('roastee-updates', 1);
     const cached = await db.get('firmware', deviceType);
     return cached?.firmware ?? null;
   }

Configuration Persistence for OTA
=================================

User configuration must survive firmware updates:

**Preserved Across Updates:**

- Device password (NVS)
- WiFi configuration (NVS)
- Custom hostname (NVS)
- BLE bonding information (NVS)
- User preferences (NVS)

**MCUboot Guarantee:**

MCUboot's slot-based architecture preserves the ``storage_partition``
(NVS/littlefs) across firmware updates. Only the code partition is modified.

**First-Boot After Update:**

1. MCUboot validates new firmware in slot0
2. Slate boots with new firmware
3. NVS configuration loaded (unchanged)
4. User session continues without re-authentication

Update Server Considerations
============================

For production deployment, Roastee needs an update server:

**Minimal Requirements:**

- HTTPS endpoint serving firmware manifests
- Version information (semantic versioning)
- Firmware binaries (signed for verification)
- Device compatibility metadata

**Example Manifest:**

.. code-block:: json

   {
     "devices": {
       "slate": {
         "latest": "1.2.0",
         "url": "https://updates.thermoquad.com/slate/1.2.0.bin",
         "sha256": "abc123...",
         "releaseNotes": "Bug fixes and improvements"
       },
       "helios": {
         "latest": "1.1.5",
         "url": "https://updates.thermoquad.com/helios/1.1.5.bin",
         "sha256": "def456...",
         "releaseNotes": "Temperature control improvements"
       }
     }
   }

**Offline-First Consideration:**

The update server is only needed for checking/downloading updates.
Once firmware is cached in Roastee, updates can be applied without internet.

Feasibility Assessment
**********************

Component Feasibility
=====================

.. list-table:: Implementation Feasibility
   :widths: 25 15 60
   :header-rows: 1

   * - Component
     - Status
     - Notes
   * - WiFi AP Mode
     - Feasible
     - CYW43439 supports SoftAP, Zephyr has AP mode support
   * - DHCP Server
     - Feasible
     - Zephyr includes DHCPv4 server
   * - DNS Server
     - Custom Required
     - Minimal DNS server needed for captive portal
   * - mDNS Responder
     - Feasible
     - Zephyr includes mDNS responder
   * - HTTP Server
     - Feasible
     - Zephyr HTTP server supports static files, WebSocket
   * - Captive Portal
     - Feasible
     - Combine DNS hijack + HTTP redirect
   * - PWA Hosting
     - Feasible
     - Embed compressed assets in firmware
   * - BLE Peripheral
     - Feasible
     - Zephyr BLE stack fully supports peripheral role
   * - Web Bluetooth
     - Partial
     - Works on Android/Chrome, NOT iOS Safari
   * - OTA Updates
     - Feasible
     - MCUboot + Roastee orchestration; works via WiFi or BLE

Platform Support Matrix
=======================

.. list-table:: Roastee Platform Support
   :widths: 20 20 20 40
   :header-rows: 1

   * - Platform
     - WiFi
     - BLE
     - Notes
   * - Android (Chrome)
     - Full
     - Full
     - Best experience
   * - Windows (Chrome)
     - Full
     - Full
     - Desktop support
   * - macOS (Chrome)
     - Full
     - Full
     - Desktop support
   * - iOS (Safari)
     - Full
     - None
     - No Web Bluetooth; WiFi-only
   * - iOS (Bluefy)
     - Full
     - Full
     - Third-party browser workaround
   * - Linux (Chrome)
     - Full
     - Flag
     - Requires experimental flag

Known Limitations
=================

1. **iOS BLE**: Web Bluetooth not supported in Safari
2. **HTTPS**: Local network PWAs must handle certificate challenges
3. **DNS Server**: Custom implementation required for Zephyr
4. **Flash Size**: PWA assets add to firmware size (~100-500KB compressed)
5. **Concurrent Connections**: CYW43439 limited to 4 AP clients

Resource Estimates
==================

**Flash Usage (Block/Slate):**

.. list-table::
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
   * - Roastee PWA (gzip)
     - ~65 KB typical, 150 KB max
     - See :doc:`roastee-web-stack` for breakdown
   * - **Total**
     - ~585 KB typical, 770 KB max
     - Fits in 4 MB flash

**RAM Usage:**

.. list-table::
   :widths: 40 30 30
   :header-rows: 1

   * - Component
     - Estimated Size
     - Notes
   * - WiFi buffers
     - ~20 KB
     - TX/RX buffers
   * - HTTP server
     - ~10 KB
     - Connection state
   * - WebSocket
     - ~4 KB per client
     - Per-connection
   * - BLE
     - ~10 KB
     - Stack + buffers

Implementation Roadmap
**********************

Phase 1: Core Infrastructure
============================

1. Implement WiFi AP mode on Block
2. Implement DHCP server
3. Implement minimal DNS server (captive portal)
4. Implement HTTP server with static file serving
5. Test captive portal detection on iOS/Android/Windows

Phase 2: Configuration Portal
=============================

1. Design configuration UI (HTML/CSS/JS)
2. Implement network scanning API
3. Implement configuration persistence (NVS)
4. Implement network transition logic
5. Implement mDNS registration

Phase 3: Roastee PWA
====================

1. Set up TypeScript project with Vite
2. Implement WebSocket client
3. Implement Fusain protocol encoder/decoder
4. Implement service worker for offline support
5. Test PWA installation on multiple platforms

Phase 4: Authentication
=======================

1. Implement device password storage
2. Implement authentication API
3. Implement token-based session management
4. Integrate authentication into Roastee

Phase 5: BLE Integration
========================

1. Implement BLE GATT service on Block
2. Implement time-limited advertising
3. Implement Web Bluetooth client in Roastee
4. Test BLE pairing and communication
5. Implement WiFi-to-BLE transition flow

Phase 6: OTA Updates
====================

1. Integrate MCUboot into Slate and Helios builds
2. Implement firmware transfer API (WebSocket + BLE)
3. Implement chunked transfer with integrity checks
4. Implement Roastee update checking and caching (IndexedDB)
5. Implement proxy update flow (Slate → Helios via Fusain)
6. Test all three OTA scenarios (WiFi station, WiFi AP, BLE)

References
**********

.. [1] Raspberry Pi Radio Module 2 Datasheet
   https://www.raspberrypi.com/news/raspberry-pi-radio-module-2-available-now-at-4/

.. [2] Captive Portal Detection Mechanisms
   https://community.fortinet.com/t5/FortiGate/Technical-Tip-Understanding-Captive-Portal-Auto-Detection/ta-p/400071

.. [3] DHCP Option 114 for Captive Portals
   https://developer.apple.com/news/?id=q78sq5rv

.. [4] PWA Installation Requirements
   https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Tutorials/js13kGames/Offline_Service_workers

.. [5] PWA Install Prompts (2025 Guide)
   https://junkangworld.com/blog/master-pwa-installs-on-ios-android-the-2025-guide

.. [6] Web Bluetooth API Browser Support
   https://caniuse.com/web-bluetooth

.. [7] Zephyr WiFi AP-STA Mode Sample
   https://docs.zephyrproject.org/latest/samples/net/wifi/apsta_mode/README.html

.. [8] Zephyr DHCPv4 Server
   https://github.com/zephyrproject-rtos/zephyr/issues/41864

.. [9] Zephyr mDNS Responder
   https://github.com/zephyrproject-rtos/zephyr/issues/29429

.. [10] Zephyr HTTP Server Documentation
   https://docs.zephyrproject.org/latest/connectivity/networking/api/http_server.html

.. [11] Zephyr Bluetooth LE Host Documentation
   https://docs.zephyrproject.org/latest/connectivity/bluetooth/bluetooth-le-host.html

.. [12] PWA Service Worker Caching Strategies
   https://developers.google.com/codelabs/pwa-training/pwa03--going-offline

.. [13] Wi-Fi Easy Connect (DPP)
   https://www.wi-fi.org/beacon/dan-harkins/wi-fi-easy-connect-simple-and-secure-onboarding-for-iot

**Additional Resources:**

- Web Bluetooth API Documentation: https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API
- BLE Pairing and Security: https://technotes.kynetics.com/2018/BLE_Pairing_and_bonding/

Appendix A: SSID Generation
***************************

Generate unique SSID from RP2350's unique ID:

.. code-block:: c

   #include <zephyr/drivers/hwinfo.h>

   void generate_ssid(char *ssid, size_t len)
   {
       uint8_t device_id[8];
       ssize_t id_len = hwinfo_get_device_id(device_id, sizeof(device_id));

       if (id_len > 0) {
           // Use last 4 bytes for hex suffix
           snprintf(ssid, len, "Thermoquad-%02X%02X%02X%02X",
                    device_id[id_len-4], device_id[id_len-3],
                    device_id[id_len-2], device_id[id_len-1]);
       } else {
           strncpy(ssid, "Thermoquad-0000", len);
       }
   }

Appendix B: Captive Portal Response
***********************************

Example captive portal redirect response:

.. code-block:: none

   HTTP/1.1 302 Found
   Location: http://192.168.4.1/setup
   Content-Length: 0
   Connection: close

For Apple devices expecting specific content:

.. code-block:: html

   <!-- Return this instead of "Success" to trigger portal -->
   <html>
   <head>
       <meta http-equiv="refresh" content="0;url=http://192.168.4.1/setup">
   </head>
   <body>
       <a href="http://192.168.4.1/setup">Click here to configure</a>
   </body>
   </html>

Appendix C: Web Bluetooth GATT Service
**************************************

Proposed GATT service structure for Fusain over BLE:

.. code-block:: none

   Thermoquad Service (UUID: TBD)
   ├── Fusain TX Characteristic (UUID: TBD)
   │   ├── Properties: Write, Write Without Response
   │   └── Purpose: Send Fusain packets to device
   ├── Fusain RX Characteristic (UUID: TBD)
   │   ├── Properties: Notify
   │   └── Purpose: Receive Fusain packets from device
   └── Device Info Characteristic (UUID: TBD)
       ├── Properties: Read
       └── Purpose: Device name, firmware version

**MTU Considerations:**

Default BLE MTU is 23 bytes (20 bytes payload). For larger Fusain packets:

- Request MTU increase (up to 512 bytes)
- Implement packet fragmentation if needed
- Web Bluetooth handles MTU negotiation automatically
