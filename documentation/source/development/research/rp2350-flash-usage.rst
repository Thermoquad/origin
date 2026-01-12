RP2350 Flash Usage
##################

:Date: 2026-01-11
:Author: Thermoquad
:Status: Research Complete

.. note::

   **Development Status: Research Phase**

   This document captures research findings and recommendations. Implementation
   has not yet begun. The findings here will guide future development of
   persistent storage and OTA update features in Helios and Slate firmware.

.. contents:: Table of Contents
   :local:
   :depth: 2

Executive Summary
*****************

This research investigates flash storage options for the RP2350 (Raspberry Pi Pico 2)
running Zephyr RTOS. The goal is to enable persistent configuration storage for
Thermoquad firmware applications (Helios, Slate).

**Key Findings:**

- RP2350 flash driver support was added to Zephyr in v4.3.0
- The Pico 2 has 4MB flash, with 3MB available for storage after firmware
- Two storage APIs are available: **NVS** (key-value) and **littlefs** (filesystem)
- **Recommendation:** Use NVS for configuration data, littlefs for logging/files
- **OTA Updates:** Firmware images are written directly to flash slots (not filesystems)
- **Custom boards** can use 8-16MB flash for expanded storage
- **Proxy updates** (Slate → Helios): Use littlefs on Slate to buffer firmware images

Hardware Overview
*****************

RP2350 Flash Specifications
===========================

The Raspberry Pi Pico 2 uses an external QSPI flash chip:

.. list-table:: Flash Specifications
   :widths: 30 70
   :header-rows: 1

   * - Parameter
     - Value
   * - Total Capacity
     - 4 MB (Winbond W25Q32JV or equivalent)
   * - Minimum Erase Unit
     - 4 KB (sector)
   * - Minimum Write Unit
     - 1 byte (optimized for 256-byte pages)
   * - Write Endurance
     - ~100,000 cycles per sector
   * - Data Retention
     - 20+ years
   * - Interface
     - QSPI (Quad SPI)

**Note:** Unlike EEPROM, flash memory requires erasing an entire sector (4KB) before
writing. This impacts wear leveling and storage design. [1]_

Differences from RP2040
=======================

The RP2350 uses a different flash controller than the RP2040:

.. list-table:: Flash Controller Comparison
   :widths: 25 35 40
   :header-rows: 1

   * - Aspect
     - RP2040
     - RP2350
   * - Controller
     - SSI (Synchronous Serial Interface)
     - QMI (QSPI Memory Interface)
   * - Zephyr Header
     - ``hardware/structs/ssi.h``
     - ``hardware/structs/qmi.h``
   * - Driver Support
     - Zephyr v2.6+
     - Zephyr v4.3+ (PR #89182)

This architectural difference caused early compatibility issues with MCUboot and
flash storage on RP2350, which have since been resolved. [2]_

Zephyr Flash Support Status
***************************

Driver Implementation
=====================

The Zephyr flash driver for Raspberry Pi Pico boards is implemented in:

``zephyr/drivers/flash/flash_rpi_pico.c``

Key commits:

- ``5d36e85b99a`` - Add support for RP2350 flash controller
- ``428eced7d06`` - Fix indentation and remove unused variables

The driver uses the Pico SDK's ``hardware/flash.h`` HAL, which abstracts the
differences between RP2040 (SSI) and RP2350 (QMI) controllers.

Flash Partition Layout
======================

Zephyr provides pre-defined partition templates for Raspberry Pi boards:

**Standard Layout (4MB, no bootloader):**

``zephyr/dts/vendor/raspberrypi/partitions_4M_storage.dtsi``

.. code-block:: none

   Address       Size    Purpose
   ──────────────────────────────────────
   0x00000000    1 MB    Code (firmware)
   0x00100000    3 MB    Storage (NVS/littlefs)

**MCUboot Layout (4MB, with bootloader):**

``zephyr/dts/vendor/raspberrypi/partitions_4M_sysbuild.dtsi``

.. code-block:: none

   Address       Size    Purpose
   ──────────────────────────────────────
   0x00000000    64 KB   MCUboot bootloader
   0x00010000    832 KB  Image slot 0 (primary)
   0x000E0000    832 KB  Image slot 1 (secondary)
   0x001B0000    2.3 MB  Storage

Device Tree Configuration
=========================

The Pico 2 board definition (``rpi_pico2.dtsi``) configures:

.. code-block:: devicetree

   / {
       chosen {
           zephyr,flash = &flash0;
           zephyr,flash-controller = &qmi;
       };
   };

   &flash0 {
       reg = <0x10000000 DT_SIZE_M(4)>;
   };

The storage partition is automatically available when using the standard partition
layout. No additional device tree modifications are required.

Storage Options
***************

Zephyr provides two primary storage mechanisms for flash:

NVS (Non-Volatile Storage)
==========================

NVS is a lightweight key-value store optimized for flash memory.

**Characteristics:**

- Simple ID-based storage (16-bit keys)
- Automatic wear leveling across sectors
- Power-loss safe writes
- Minimal RAM overhead (~10KB)
- No directory structure

**Best For:**

- Configuration parameters (WiFi credentials, calibration data)
- Boot counters and usage statistics
- Device state persistence
- Small binary blobs (<4KB)

**Kconfig:**

.. code-block:: kconfig

   CONFIG_NVS=y
   CONFIG_FLASH=y
   CONFIG_FLASH_PAGE_LAYOUT=y
   CONFIG_FLASH_MAP=y

**API Example:**

.. code-block:: c

   #include <zephyr/fs/nvs.h>
   #include <zephyr/storage/flash_map.h>

   static struct nvs_fs fs;

   int storage_init(void)
   {
       struct flash_pages_info info;
       int rc;

       fs.flash_device = FIXED_PARTITION_DEVICE(storage_partition);
       fs.offset = FIXED_PARTITION_OFFSET(storage_partition);

       rc = flash_get_page_info_by_offs(fs.flash_device, fs.offset, &info);
       if (rc) {
           return rc;
       }

       fs.sector_size = info.size;   // 4KB for RP2350
       fs.sector_count = 3;          // Minimum for wear leveling

       return nvs_mount(&fs);
   }

   // Write a configuration value
   int config_save(uint16_t id, void *data, size_t len)
   {
       return nvs_write(&fs, id, data, len);
   }

   // Read a configuration value
   int config_load(uint16_t id, void *data, size_t len)
   {
       return nvs_read(&fs, id, data, len);
   }

**Reference:** Zephyr NVS sample at ``samples/subsys/nvs/`` [3]_

littlefs
========

littlefs is a fail-safe filesystem designed for microcontrollers.

**Characteristics:**

- Full POSIX-like file API (open, read, write, seek)
- Directory support
- Built-in wear leveling
- Power-loss resilient
- Higher RAM overhead (~100KB)

**Best For:**

- Telemetry logging
- Multiple configuration files
- Large data storage (>4KB per item)
- Directory organization needs

**Kconfig:**

.. code-block:: kconfig

   CONFIG_FILE_SYSTEM=y
   CONFIG_FILE_SYSTEM_LITTLEFS=y
   CONFIG_FLASH=y
   CONFIG_FLASH_MAP=y
   CONFIG_FS_LITTLEFS_NUM_FILES=4
   CONFIG_FS_LITTLEFS_CACHE_SIZE=64

**API Example:**

.. code-block:: c

   #include <zephyr/fs/fs.h>
   #include <zephyr/fs/littlefs.h>
   #include <zephyr/storage/flash_map.h>

   FS_LITTLEFS_DECLARE_DEFAULT_CONFIG(storage);

   static struct fs_mount_t lfs_mnt = {
       .type = FS_LITTLEFS,
       .fs_data = &storage,
       .storage_dev = (void *)FIXED_PARTITION_ID(storage_partition),
       .mnt_point = "/lfs",
   };

   int filesystem_init(void)
   {
       return fs_mount(&lfs_mnt);
   }

   int config_save_file(const char *filename, void *data, size_t len)
   {
       struct fs_file_t file;
       char path[64];
       int rc;

       snprintf(path, sizeof(path), "/lfs/%s", filename);

       fs_file_t_init(&file);
       rc = fs_open(&file, path, FS_O_CREATE | FS_O_WRITE);
       if (rc < 0) {
           return rc;
       }

       rc = fs_write(&file, data, len);
       fs_close(&file);

       return rc;
   }

**Reference:** Zephyr littlefs sample at ``samples/subsys/fs/littlefs/`` [4]_

Comparison
==========

.. list-table:: NVS vs littlefs
   :widths: 25 35 40
   :header-rows: 1

   * - Feature
     - NVS
     - littlefs
   * - API Type
     - Key-value (ID-based)
     - File operations (POSIX-like)
   * - RAM Overhead
     - ~10 KB
     - ~100 KB
   * - Complexity
     - Low
     - Medium
   * - Directories
     - No
     - Yes
   * - Max Item Size
     - ~4 KB (sector size)
     - Limited by partition
   * - Best Use Case
     - Configuration
     - Logging/Files

MicroPython Comparison
**********************

MicroPython on Raspberry Pi Pico uses a similar approach:

- **Filesystem:** littlefs (default) or FAT
- **Storage Size:** ~1.4 MB (after firmware)
- **Configuration:** Defined in ``rp2_flash.c`` [5]_

.. code-block:: c

   // MicroPython flash storage allocation
   #ifndef MICROPY_HW_FLASH_STORAGE_BYTES
   #define MICROPY_HW_FLASH_STORAGE_BYTES (1408*1024)  // ~1.4 MB
   #endif

MicroPython reserves approximately 500-600KB for firmware and allocates the
remainder for the filesystem. This validates that flash storage on Pico devices
is practical and well-supported. [6]_ [7]_

OTA Updates and Custom Flash
****************************

This section addresses Over-The-Air (OTA) firmware updates via Fusain protocol
and considerations for custom boards with larger flash chips.

MCUboot Integration
===================

MCUboot is Zephyr's default bootloader for secure firmware updates. It uses a
**slot-based architecture** where firmware images are written directly to flash
partitions, **not** to a filesystem.

**Key Insight:** Neither NVS nor littlefs is used for storing firmware images
during OTA updates. Images are written directly to flash slots.

**MCUboot Partition Layout (4MB flash):**

.. code-block:: none

   Address       Size    Purpose
   ──────────────────────────────────────
   0x00000000    64 KB   MCUboot bootloader
   0x00010000    832 KB  slot0_partition (primary/active)
   0x000E0000    832 KB  slot1_partition (staging/upgrade)
   0x001B0000    2.3 MB  storage_partition (NVS/littlefs)

**Update Process:**

1. New firmware image received over transport (serial, network)
2. Image written directly to ``slot1_partition`` via ``flash_area_write()``
3. Image header and trailer validated
4. MCUboot swaps slot0 and slot1 on next reboot
5. If boot fails, MCUboot reverts to previous image [9]_

**Serial Update via MCUmgr:**

Zephyr provides MCUmgr (formerly mcumgr) for firmware updates over serial.
The SMP (Serial Management Protocol) handles firmware transfer and slot management.

.. code-block:: kconfig

   # Enable MCUmgr for serial firmware updates
   CONFIG_MCUBOOT=y
   CONFIG_MCUMGR=y
   CONFIG_MCUMGR_TRANSPORT_UART=y
   CONFIG_FLASH=y
   CONFIG_IMG_MANAGER=y
   CONFIG_STREAM_FLASH=y

**Reference:** Zephyr MCUmgr Documentation [8]_

Fusain-Based OTA Scenarios
==========================

Two OTA update scenarios are relevant for Thermoquad:

**Scenario 1: Self-Update (Slate or Helios)**

A device receives firmware for itself over Fusain protocol.

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - Component
     - Usage
   * - Firmware Storage
     - Direct flash to ``slot1_partition`` (no filesystem)
   * - NVS
     - Not used for firmware images
   * - littlefs
     - Not used for firmware images
   * - MCUmgr/SMP
     - Standard approach for serial updates

**Recommendation:** Use MCUmgr/SMP over UART. No additional storage needed.

**Scenario 2: Proxy Update (Slate → Helios)**

Slate receives Helios firmware and relays it to the ICU.

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - Component
     - Usage
   * - Firmware Storage
     - **littlefs on Slate** for buffering before transfer
   * - Why littlefs?
     - Need to store full image (~124KB) before fragmented transfer
   * - Alternative
     - Stream directly without buffering (complex, requires reliable link)

**Recommendation:** Use littlefs on Slate to buffer Helios firmware images
before transfer. This allows:

- Verification of complete image before transfer
- Resumable transfers if interrupted
- Multiple retry attempts without re-downloading

**Proxy Update Flow:**

1. Slate receives Helios firmware (over WiFi, Bluetooth, or serial bridge)
2. Slate writes firmware to ``/lfs/helios_update.bin``
3. Slate verifies image integrity (CRC, signature)
4. Slate transfers image to Helios over Fusain in chunks
5. Helios writes chunks directly to ``slot1_partition``
6. Helios reboots and MCUboot validates/swaps

Storage Recommendation for OTA
==============================

.. list-table:: OTA Storage Matrix
   :widths: 25 25 50
   :header-rows: 1

   * - Update Type
     - Storage
     - Reason
   * - Self-update
     - None (direct flash)
     - MCUboot writes directly to slot1
   * - Proxy buffer
     - littlefs
     - Need full image for verification
   * - Update metadata
     - NVS
     - Version info, update status, rollback flags
   * - Configuration
     - NVS
     - Preserved across updates

Custom Board Flash Options
==========================

The RP2350 supports external QSPI flash up to **16 MB**. Custom boards can use
larger flash chips for expanded storage or larger firmware images.

**Common Flash Sizes:**

.. list-table::
   :widths: 20 30 50
   :header-rows: 1

   * - Size
     - Storage Available
     - Use Case
   * - 2 MB
     - ~1 MB
     - Minimal (development boards)
   * - 4 MB
     - ~3 MB
     - Standard (Pico 2)
   * - 8 MB
     - ~7 MB
     - Extended storage, logging
   * - 16 MB
     - ~15 MB
     - Large filesystems, multiple images

**Current Thermoquad Firmware Sizes:**

.. list-table::
   :widths: 30 30 40
   :header-rows: 1

   * - Firmware
     - Size
     - Notes
   * - Helios ICU
     - ~124 KB
     - Minimal UI, focused functionality
   * - Slate Controller
     - ~397 KB
     - LVGL display, networking stack

Both firmwares fit comfortably within the 832 KB MCUboot slot with significant
headroom for future features.

Custom Partition Layouts
========================

For custom boards with larger flash, create a custom partition layout.

**Example: 8 MB Flash with MCUboot:**

.. code-block:: devicetree

   // boards/custom_board.overlay

   &flash0 {
       reg = <0x10000000 DT_SIZE_M(8)>;

       partitions {
           compatible = "fixed-partitions";
           #address-cells = <1>;
           #size-cells = <1>;

           boot_partition: partition@0 {
               label = "mcuboot";
               reg = <0x00000000 DT_SIZE_K(64)>;
               read-only;
           };
           slot0_partition: partition@10000 {
               label = "image-0";
               reg = <0x00010000 DT_SIZE_M(1)>;  // 1 MB
           };
           slot1_partition: partition@110000 {
               label = "image-1";
               reg = <0x00110000 DT_SIZE_M(1)>;  // 1 MB
           };
           storage_partition: partition@210000 {
               label = "storage";
               reg = <0x00210000 DT_SIZE_M(5)>;  // 5+ MB
           };
       };
   };

**Example: 16 MB Flash without MCUboot:**

.. code-block:: devicetree

   &flash0 {
       reg = <0x10000000 DT_SIZE_M(16)>;

       partitions {
           compatible = "fixed-partitions";
           #address-cells = <1>;
           #size-cells = <1>;

           code_partition: partition@0 {
               label = "code";
               reg = <0x00000000 DT_SIZE_M(1)>;  // 1 MB
           };
           storage_partition: partition@100000 {
               label = "storage";
               reg = <0x00100000 DT_SIZE_M(15)>; // 15 MB
           };
       };
   };

Flash Chip Selection
====================

When designing custom boards, consider these QSPI flash options:

.. list-table::
   :widths: 30 15 20 35
   :header-rows: 1

   * - Part Number
     - Size
     - Interface
     - Notes
   * - W25Q32JV
     - 4 MB
     - QSPI
     - Pico 2 default
   * - W25Q64JV
     - 8 MB
     - QSPI
     - Common upgrade option
   * - W25Q128JV
     - 16 MB
     - QSPI
     - Maximum supported

**RP2350 Requirements:** [10]_

- QSPI interface at 3.3V
- XIP (Execute-In-Place) capable
- Minimum 4 KB sector erase
- Compatible with Pico SDK flash HAL

Implementation Plan
*******************

Recommended Approach for Thermoquad
===================================

**Phase 1: NVS for Configuration**

Start with NVS for storing configuration data:

1. Add storage partition overlay (if not using default)
2. Enable NVS in ``prj.conf``
3. Implement configuration module with save/load functions
4. Define configuration IDs for each parameter

**Configuration Items (Helios):**

.. list-table::
   :widths: 20 20 60
   :header-rows: 1

   * - ID
     - Type
     - Description
   * - 0x0001
     - uint32_t
     - Boot counter
   * - 0x0010
     - struct
     - Motor configuration (PID gains, limits)
   * - 0x0011
     - struct
     - Pump configuration (pulse timing)
   * - 0x0012
     - struct
     - Temperature configuration (PID gains)
   * - 0x0020
     - uint32_t
     - Telemetry interval
   * - 0x0021
     - uint32_t
     - Timeout configuration

**Configuration Items (Slate):**

.. list-table::
   :widths: 20 20 60
   :header-rows: 1

   * - ID
     - Type
     - Description
   * - 0x0001
     - uint32_t
     - Boot counter
   * - 0x0010
     - uint8_t
     - Display brightness
   * - 0x0011
     - uint8_t
     - UI theme (light/dark)
   * - 0x0020
     - struct
     - WiFi credentials (SSID, password)
   * - 0x0021
     - uint64_t
     - Paired Helios address

**Phase 2: littlefs for Logging (Future)**

Add littlefs when telemetry logging is needed:

1. Enable littlefs alongside NVS (they can share the partition)
2. Implement log rotation
3. Add USB mass storage for log retrieval (optional)

prj.conf Changes
================

**Minimum for NVS:**

.. code-block:: kconfig

   # Flash support
   CONFIG_FLASH=y
   CONFIG_FLASH_MAP=y
   CONFIG_FLASH_PAGE_LAYOUT=y

   # NVS
   CONFIG_NVS=y

**For littlefs (future):**

.. code-block:: kconfig

   # Flash support
   CONFIG_FLASH=y
   CONFIG_FLASH_MAP=y

   # Filesystem
   CONFIG_FILE_SYSTEM=y
   CONFIG_FILE_SYSTEM_LITTLEFS=y
   CONFIG_FS_LITTLEFS_NUM_FILES=4

Board Overlay (Optional)
========================

The default partition layout should work. If customization is needed:

.. code-block:: devicetree

   // boards/rpi_pico2_rp2350a_m33.overlay

   #include <raspberrypi/partitions_4M_storage.dtsi>

   // Optionally add littlefs node for auto-mount
   / {
       fstab {
           compatible = "zephyr,fstab";
           lfs1: lfs1 {
               compatible = "zephyr,fstab,littlefs";
               mount-point = "/lfs";
               partition = <&storage_partition>;
           };
       };
   };

Factory Reset
*************

Factory reset clears user configuration and returns the device to initial state.
This is essential for password recovery, troubleshooting, and device resale.

Reset Behavior
==============

**What Gets Cleared:**

- Device password
- WiFi configuration (SSID, password, mode)
- Custom hostname
- BLE bonding information
- User preferences (telemetry interval, etc.)
- Boot counter (optional - may want to preserve)

**What Gets Preserved:**

- Firmware (slot0 remains intact)
- Hardware calibration data (if stored separately)

**Post-Reset State:**

- Device boots into unconfigured/setup mode
- Slate: Activates WiFi AP with open network
- Helios: Uses firmware defaults, awaits Slate connection

NVS Factory Reset Implementation
================================

.. code-block:: c

   #include <zephyr/fs/nvs.h>
   #include <zephyr/settings/settings.h>

   int factory_reset(void)
   {
       int rc;

       // Option 1: Clear all NVS data
       rc = nvs_clear(&fs);
       if (rc) {
           LOG_ERR("Failed to clear NVS: %d", rc);
           return rc;
       }

       // Option 2: If using Zephyr settings subsystem
       // rc = settings_delete("config");

       LOG_INF("Factory reset complete, rebooting...");
       sys_reboot(SYS_REBOOT_COLD);

       return 0;  // Never reached
   }

Hardware Reset Triggers
=======================

**Slate (Block):**

Block has RGB LEDs and potentially accessible buttons. Reset can be triggered by:

1. **Button combination**: Hold button during boot for 10+ seconds
2. **Software command**: Authenticated API call from Roastee
3. **LED feedback**: Blink pattern confirms reset in progress

**Helios (Hades):**

Hades resides inside the heater enclosure with no external user interface.
Reset options:

1. **Jumper pads**: Two PCB pads that trigger reset when shorted during boot
2. **Software command**: Via Fusain from authenticated Slate
3. **Service mode**: Intended for use during heater servicing

**Hades Jumper Pad Design:**

.. code-block:: none

   ┌─────────────────────────────────────┐
   │  HADES PCB                          │
   │                                     │
   │    ┌───┐  ┌───┐                     │
   │    │RST│  │GND│   Factory Reset     │
   │    │ ○ │──│ ○ │   Pads (2.54mm)     │
   │    └───┘  └───┘                     │
   │                                     │
   │    Short during boot = factory reset│
   └─────────────────────────────────────┘

**Reset Confirmation:**

Service technicians can confirm a successful reset by connecting to UART0
(shell/logging port on RP2350). After reset, Helios logs a message indicating
no persisted configuration was found:

.. code-block:: none

   [INF] config: No persisted configuration found, using defaults

**Implementation:**

.. code-block:: c

   #include <zephyr/drivers/gpio.h>

   #define FACTORY_RESET_PIN  DT_ALIAS(factory_reset)

   static const struct gpio_dt_spec reset_pin =
       GPIO_DT_SPEC_GET(FACTORY_RESET_PIN, gpios);

   int check_factory_reset(void)
   {
       int val;

       if (!gpio_is_ready_dt(&reset_pin)) {
           return -ENODEV;
       }

       gpio_pin_configure_dt(&reset_pin, GPIO_INPUT | GPIO_PULL_UP);

       // Check if jumper is shorting the pin to ground
       val = gpio_pin_get_dt(&reset_pin);

       if (val == 0) {
           // Pin pulled low - jumper detected
           LOG_WRN("Factory reset jumper detected!");

           // Wait a moment and re-check (debounce)
           k_sleep(K_MSEC(100));
           val = gpio_pin_get_dt(&reset_pin);

           if (val == 0) {
               return factory_reset();
           }
       }

       return 0;  // Normal boot
   }

   // Call early in main() before loading configuration
   void main(void)
   {
       check_factory_reset();
       // ... normal initialization
   }

**Device Tree for Hades:**

.. code-block:: devicetree

   / {
       aliases {
           factory-reset = &factory_reset_pin;
       };

       factory_reset_pin: factory-reset {
           gpios = <&gpio0 XX GPIO_ACTIVE_LOW>;  // XX = chosen GPIO
       };
   };

Remote Factory Reset (via Fusain)
=================================

Slate can command Helios to factory reset over Fusain:

1. User authenticates in Roastee
2. User requests "Reset Helios to defaults"
3. Roastee sends command to Slate
4. Slate sends ``FACTORY_RESET`` command to Helios via Fusain
5. Helios clears NVS and reboots
6. Helios reconnects with default configuration

**Security:** Factory reset command requires authentication to prevent
malicious resets. Consider requiring physical confirmation (e.g., heater
must be in IDLE state, or require button press on Block).

Risks and Mitigations
*********************

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - Risk
     - Mitigation
   * - Flash wear-out
     - Use NVS/littlefs wear leveling; avoid frequent writes
   * - Power loss during write
     - Both NVS and littlefs are power-loss safe by design
   * - Firmware update overwrites storage
     - Use separate partitions; MCUboot preserves storage
   * - Configuration corruption
     - Implement CRC validation; store defaults in code
   * - Forgotten device password
     - Factory reset via jumper pads (Hades) or button (Block)

References
**********

.. [1] Raspberry Pi Pico Datasheet - Flash Memory Specifications
   https://datasheets.raspberrypi.com/pico/pico-datasheet.pdf

.. [2] Zephyr GitHub Issue #88488 - Flash Controller support for RP2350
   https://github.com/zephyrproject-rtos/zephyr/issues/88488

.. [3] Zephyr NVS Documentation
   https://docs.zephyrproject.org/latest/services/storage/nvs/nvs.html

.. [4] Zephyr littlefs Sample
   https://docs.zephyrproject.org/latest/samples/subsys/fs/littlefs/README.html

.. [5] MicroPython RP2 Flash Implementation
   https://github.com/micropython/micropython/blob/master/ports/rp2/rp2_flash.c

.. [6] Raspberry Pi Pico MicroPython Documentation
   https://www.raspberrypi.com/documentation/microcontrollers/micropython.html

.. [7] Random Nerd Tutorials - Pico Files and Directories
   https://randomnerdtutorials.com/raspberry-pi-pico-files-directories-micropython/

.. [8] Zephyr MCUmgr Documentation
   https://docs.zephyrproject.org/latest/services/device_mgmt/mcumgr.html

.. [9] Zephyr MCUboot Integration
   https://docs.zephyrproject.org/latest/services/device_mgmt/dfu.html

.. [10] RP2350 Datasheet - External Flash Interface
   https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf

Appendix A: Zephyr Version Requirements
***************************************

This research was conducted on Zephyr v4.3.0-3027-g545c2870e93.

RP2350 flash support requires:

- Zephyr v4.3.0 or later
- Pico SDK with RP2350 support

Verify flash driver is enabled:

.. code-block:: bash

   west build -t menuconfig
   # Navigate to: Device Drivers → Flash drivers → Flash driver for Raspberry Pi Pico

Appendix B: Flash Partition Macros
**********************************

Zephyr provides macros for accessing flash partitions defined in device tree:

.. code-block:: c

   // Get partition ID (for flash_area_open)
   FIXED_PARTITION_ID(storage_partition)

   // Get flash device pointer
   FIXED_PARTITION_DEVICE(storage_partition)

   // Get partition offset from flash start
   FIXED_PARTITION_OFFSET(storage_partition)

   // Get partition size
   FIXED_PARTITION_SIZE(storage_partition)

These macros reference the ``storage_partition`` node defined in the partition
layout device tree include.
