# CYW43439 BT HCI Shared-Bus Driver — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement a Zephyr BT HCI driver for CYW43439 shared-bus transport using WHD backplane access, enabling Bluetooth on Pico W/2W and RM2 boards.

**Architecture:** The driver sits between Zephyr's BT host stack and WHD's `whd_bus_transfer_backplane_bytes()`. It ports the shared-memory circular buffer protocol from pico-sdk's `cybt_shared_bus_driver.c` / `cybt_shared_bus.c`, replacing `cyw43_ll_*` calls with WHD equivalents. A dedicated RX thread polls the BT→Host buffer.

**Tech Stack:** Zephyr RTOS, C (Zephyr coding style: tabs, `/* */` comments), WHD (Infineon), CYW43439 BT firmware blob (HCD format)

**Style:** All code in this plan follows **Zephyr upstream conventions** — tabs for indentation, `/* */` comments, SPDX headers, Zephyr commit message format (`drivers: bluetooth: hci:`).

**Repo:** `Thermoquad/zephyr` branch `feat/bt-hci-cyw43-shared-bus`

**Working directory:** `/home/kazw/Work/Thermoquad/zephyr`

---

## Key Reference Files

Read these before starting any task:

- **Pico-SDK shared bus driver:** `../modules/hal/rpi_pico/src/rp2_common/pico_cyw43_driver/cybt_shared_bus/cybt_shared_bus_driver.c`
- **Pico-SDK shared bus high-level:** `../modules/hal/rpi_pico/src/rp2_common/pico_cyw43_driver/cybt_shared_bus/cybt_shared_bus.c`
- **Zephyr CYW208xx HCI driver (pattern):** `drivers/bluetooth/hci/hci_infineon_cyw208xx.c`
- **Zephyr UART HCI driver (FW loading):** `drivers/bluetooth/hci/hci_uart_infineon.c`
- **WHD backplane API:** `../modules/hal/infineon/whd-expansion/WHD/COMMON/inc/whd_bus_common.h`
- **WHD types (BUS_READ/BUS_WRITE):** `../modules/hal/infineon/whd-expansion/WHD/COMPONENT_WIFI6/inc/whd_types.h`
- **WHD internal header (struct whd_interface):** `../modules/hal/infineon/whd-expansion/WHD/COMPONENT_WIFI6/src/include/whd_int.h`
- **AIROC WiFi header (whd_interface_t):** `drivers/wifi/infineon/airoc_wifi.h`
- **Pico 2W board DTS:** `boards/raspberrypi/rpi_pico2/rpi_pico2_rp2350a_m33_w.dts`
- **FW blob array:** `modules/hal_infineon/btstack-integration/w_bt_firmware_controller.c` (provides `brcm_patchram_buf[]`)

---

### Task 1: Device Tree Binding

Create the DT binding YAML for the shared-bus BT HCI node.

**Files:**
- Create: `dts/bindings/bluetooth/infineon,cyw43-bt-hci-sbus.yaml`

**Step 1: Create the binding file**

```yaml
# Copyright (c) 2026 Thermoquad
#
# SPDX-License-Identifier: Apache-2.0

description: |
    Infineon CYW43xxx Bluetooth HCI over shared SPI backplane.

    Used on boards where BT UART pins are not routed to the host MCU
    (e.g., Raspberry Pi Pico W/2W, Murata RM2 module). BT HCI traffic
    goes over the same SPI bus as WiFi via shared-memory circular buffers
    in the CYW43439 internal RAM, accessed through WHD backplane functions.

    Must be a child node of an infineon,airoc-wifi node.

    Example:

      airoc-wifi@0 {
          compatible = "infineon,airoc-wifi";
          /* ... WiFi properties ... */

          bt_hci: bt-hci {
              compatible = "infineon,cyw43-bt-hci-sbus";
              status = "okay";
          };
      };

compatible: "infineon,cyw43-bt-hci-sbus"

include: [bt-hci.yaml]
```

**Step 2: Verify the binding parses correctly**

Run: `python3 scripts/dts/dtsh.py --help` or just proceed to Task 3 where the DTS is added — the build will validate it.

**Step 3: Commit**

```bash
git add dts/bindings/bluetooth/infineon,cyw43-bt-hci-sbus.yaml
git commit -m "$(cat <<'EOF'
drivers: bluetooth: hci: add DT binding for CYW43 shared-bus BT

Add device tree binding for Infineon CYW43xxx Bluetooth HCI over
the shared SPI backplane transport. This transport is used on boards
where BT UART pins are not routed to the host MCU (Pico W/2W, RM2).

The BT HCI node must be a child of an infineon,airoc-wifi node.

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 2: Kconfig and CMake

Add the build system integration for the new driver.

**Files:**
- Modify: `drivers/bluetooth/hci/Kconfig.infineon` (add `BT_CYW43_SBUS` near existing `CYW43439`)
- Modify: `drivers/bluetooth/hci/CMakeLists.txt` (add source file)

**Step 1: Add Kconfig option**

In `drivers/bluetooth/hci/Kconfig.infineon`, add a new config block **before** `if BT_AIROC` (this is a separate driver, not part of the AIROC UART flow):

```kconfig
config BT_CYW43_SBUS
	bool "CYW43xxx Bluetooth over shared SPI backplane"
	default y
	depends on DT_HAS_INFINEON_CYW43_BT_HCI_SBUS_ENABLED
	depends on WIFI_AIROC
	select BT_HAS_HCI_VS
	help
	  Bluetooth HCI driver for CYW43xxx using the shared SPI backplane
	  transport via WHD. Used on boards where BT UART is not routed to
	  the host (Raspberry Pi Pico W/2W, Murata RM2).
```

Place this at the very top of the file, before `if BT_AIROC`.

**Step 2: Add CMake source**

In `drivers/bluetooth/hci/CMakeLists.txt`, add after the CYW208XX line (line 42):

```cmake
zephyr_library_sources_ifdef(CONFIG_BT_CYW43_SBUS hci_cyw43_sbus.c)
```

Also add the WHD include paths needed by the driver:

```cmake
if(CONFIG_BT_CYW43_SBUS)
  zephyr_include_directories(
    ${ZEPHYR_HAL_INFINEON_MODULE_DIR}/whd-expansion/WHD/COMPONENT_WIFI6/inc
    ${ZEPHYR_HAL_INFINEON_MODULE_DIR}/whd-expansion/WHD/COMPONENT_WIFI6/src/include
    ${ZEPHYR_HAL_INFINEON_MODULE_DIR}/whd-expansion/WHD/COMMON/inc
    ${ZEPHYR_BASE}/drivers/wifi/infineon
  )
endif()
```

**Step 3: Commit**

```bash
git add drivers/bluetooth/hci/Kconfig.infineon drivers/bluetooth/hci/CMakeLists.txt
git commit -m "$(cat <<'EOF'
drivers: bluetooth: hci: add Kconfig and CMake for CYW43 shared-bus

Add BT_CYW43_SBUS Kconfig option and CMake build rules for the
CYW43xxx shared-bus BT HCI driver. Depends on WIFI_AIROC since
it accesses the BT controller through WHD backplane functions.

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 3: Board DTS Update

Add the BT HCI node to the Pico 2W board DTS.

**Files:**
- Modify: `boards/raspberrypi/rpi_pico2/rpi_pico2_rp2350a_m33_w.dts`

**Step 1: Add BT HCI child node inside the airoc-wifi node**

Inside the `airoc-wifi@0` node (after the `cyw43_gpio` child node, around line 83), add:

```dts
		bt_hci: bt-hci {
			compatible = "infineon,cyw43-bt-hci-sbus";
			status = "okay";
		};
```

**Step 2: Commit**

```bash
git add boards/raspberrypi/rpi_pico2/rpi_pico2_rp2350a_m33_w.dts
git commit -m "$(cat <<'EOF'
boards: rpi_pico2: add BT HCI shared-bus node for Pico 2W

Add infineon,cyw43-bt-hci-sbus node as a child of the airoc-wifi
node on the Pico 2W board. This enables Bluetooth over the shared
SPI backplane transport.

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 4: Driver Implementation — Backplane Primitives

Create the driver file with the low-level backplane access functions ported from `cybt_shared_bus_driver.c`.

**Files:**
- Create: `drivers/bluetooth/hci/hci_cyw43_sbus.c`

**Step 1: Create the driver file with includes, constants, types, and backplane primitives**

The driver file layout follows Zephyr conventions. This step creates the foundation: includes, constants, data structures, and the core backplane read/write functions that map `cyw43_ll_*` → `whd_bus_transfer_backplane_bytes()`.

```c
/*
 * Copyright (c) 2026 Thermoquad
 *
 * SPDX-License-Identifier: Apache-2.0
 */

/*
 * Bluetooth HCI driver for CYW43xxx over the shared SPI backplane.
 *
 * On boards where BT UART is not routed to the host (Pico W/2W, RM2),
 * BT HCI goes over shared-memory circular buffers in the CYW43439's
 * internal RAM, accessed through WHD backplane functions.
 *
 * Ported from pico-sdk cybt_shared_bus_driver.c / cybt_shared_bus.c.
 */

#include <errno.h>
#include <stddef.h>
#include <string.h>

#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/drivers/bluetooth.h>
#include <zephyr/init.h>
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include <zephyr/sys/byteorder.h>

#include <whd.h>
#include <whd_types.h>
#include <whd_bus_common.h>
#include <whd_int.h>

/* AIROC WiFi driver — provides the WHD interface handle */
#include <airoc_wifi.h>

LOG_MODULE_REGISTER(cyw43_sbus, CONFIG_BT_HCI_DRIVER_LOG_LEVEL);

#define DT_DRV_COMPAT infineon_cyw43_bt_hci_sbus

/* CYW43439 backplane addresses */
#define BTFW_MEM_OFFSET			0x19000000
#define BT_CTRL_REG_ADDR		0x18000c7c
#define HOST_CTRL_REG_ADDR		0x18000d6c
#define WLAN_RAM_BASE_REG_ADDR		0x18000d68

/* Circular buffer size (4KB each direction) */
#define BTSDIO_FWBUF_SIZE		0x1000

/* Circular buffer register offsets from WLAN_RAM_BASE */
#define BTSDIO_OFFSET_HOST_WRITE_BUF	0x0000
#define BTSDIO_OFFSET_HOST_READ_BUF	BTSDIO_FWBUF_SIZE
#define BTSDIO_OFFSET_HOST2BT_IN	0x00002000
#define BTSDIO_OFFSET_HOST2BT_OUT	0x00002004
#define BTSDIO_OFFSET_BT2HOST_IN	0x00002008
#define BTSDIO_OFFSET_BT2HOST_OUT	0x0000200C

/* Host control register bits */
#define BTSDIO_REG_DATA_VALID		BIT(1)
#define BTSDIO_REG_WAKE_BT		BIT(17)
#define BTSDIO_REG_SW_RDY		BIT(24)

/* BT control register bits */
#define BTSDIO_REG_BT_AWAKE		BIT(8)
#define BTSDIO_REG_FW_RDY		BIT(24)

/* BT firmware power-up */
#define BT2WLAN_PWRUP_WAKE		0x03
#define BT2WLAN_PWRUP_ADDR		0x640894

/* Timing */
#define FW_READY_POLL_INTERVAL_MS	1
#define FW_READY_POLL_COUNT		300
#define FW_READY_WAIT_MS		150
#define BT_AWAKE_POLL_INTERVAL_MS	1
#define BT_AWAKE_POLL_COUNT		300
#define RX_POLL_INTERVAL_MS		2

/* Circular buffer helpers */
#define CIRC_BUF_CNT(in, out)		(((in) - (out)) & (BTSDIO_FWBUF_SIZE - 1))
#define CIRC_BUF_SPACE(in, out)		CIRC_BUF_CNT((out), ((in) + 4))

#define ROUNDUP4(x)			(((x) + 3) & ~3)

/* FW download alignment */
#define BTFW_SD_ALIGN			4
#define BTFW_MAX_BLOCK_SIZE		252

/* BT firmware blob (provided by hal_infineon btstack-integration) */
extern const uint8_t brcm_patchram_buf[];
extern const int brcm_patch_ram_length;

/* Circular buffer address info */
struct sbus_buf_info {
	uint32_t h2b_buf_addr;
	uint32_t h2b_in_addr;
	uint32_t h2b_out_addr;
	uint32_t b2h_buf_addr;
	uint32_t b2h_in_addr;
	uint32_t b2h_out_addr;
};

struct cyw43_sbus_data {
	bt_hci_recv_t recv;
	whd_driver_t whd_drv;
	struct sbus_buf_info buf;
	uint32_t host_ctrl_cache;
	struct k_mutex bus_mutex;
	struct k_thread rx_thread;
	bool running;

	K_KERNEL_STACK_MEMBER(rx_stack, 1536);
};

/*
 * Backplane register read/write via WHD.
 *
 * These replace the pico-sdk's cyw43_ll_read/write_backplane_reg().
 * HOST_CTRL_REG reads are cached to avoid bus traffic (matches pico-sdk).
 */
static int sbus_reg_write(struct cyw43_sbus_data *data, uint32_t addr,
			   uint32_t val)
{
	whd_result_t ret;

	ret = whd_bus_transfer_backplane_bytes(data->whd_drv, BUS_WRITE,
					       addr, sizeof(uint32_t),
					       (uint8_t *)&val);
	if (ret != WHD_SUCCESS) {
		LOG_ERR("backplane reg write 0x%08x failed: %d", addr, ret);
		return -EIO;
	}

	if (addr == HOST_CTRL_REG_ADDR) {
		data->host_ctrl_cache = val;
	}

	return 0;
}

static int sbus_reg_read(struct cyw43_sbus_data *data, uint32_t addr,
			  uint32_t *val)
{
	whd_result_t ret;

	/* Cache reads from HOST_CTRL_REG (matches pico-sdk optimization) */
	if (addr == HOST_CTRL_REG_ADDR) {
		*val = data->host_ctrl_cache;
		return 0;
	}

	ret = whd_bus_transfer_backplane_bytes(data->whd_drv, BUS_READ,
					       addr, sizeof(uint32_t),
					       (uint8_t *)val);
	if (ret != WHD_SUCCESS) {
		LOG_ERR("backplane reg read 0x%08x failed: %d", addr, ret);
		return -EIO;
	}

	return 0;
}

/*
 * Backplane memory read/write via WHD.
 *
 * These replace the pico-sdk's cyw43_ll_read/write_backplane_mem().
 * Handles splitting transfers that cross the 4KB backplane window boundary.
 */
static int sbus_mem_write(struct cyw43_sbus_data *data, uint32_t addr,
			   const uint8_t *buf, uint32_t len)
{
	whd_result_t ret;
	uint32_t chunk;

	while (len > 0) {
		chunk = len;
		/* Don't cross 4KB backplane window boundary */
		if ((addr & 0xFFF) + chunk > 0x1000) {
			chunk = 0x1000 - (addr & 0xFFF);
		}

		ret = whd_bus_transfer_backplane_bytes(data->whd_drv, BUS_WRITE,
						       addr, chunk,
						       (uint8_t *)buf);
		if (ret != WHD_SUCCESS) {
			LOG_ERR("backplane mem write 0x%08x failed: %d",
				addr, ret);
			return -EIO;
		}

		addr += chunk;
		buf += chunk;
		len -= chunk;
	}

	return 0;
}

static int sbus_mem_read(struct cyw43_sbus_data *data, uint32_t addr,
			  uint8_t *buf, uint32_t len)
{
	whd_result_t ret;
	uint32_t chunk;

	while (len > 0) {
		chunk = len;
		if ((addr & 0xFFF) + chunk > 0x1000) {
			chunk = 0x1000 - (addr & 0xFFF);
		}

		ret = whd_bus_transfer_backplane_bytes(data->whd_drv, BUS_READ,
						       addr, chunk, buf);
		if (ret != WHD_SUCCESS) {
			LOG_ERR("backplane mem read 0x%08x failed: %d",
				addr, ret);
			return -EIO;
		}

		addr += chunk;
		buf += chunk;
		len -= chunk;
	}

	return 0;
}

/* Toggle the data-valid bit to signal the BT controller */
static int sbus_toggle_bt_intr(struct cyw43_sbus_data *data)
{
	uint32_t val;
	int err;

	err = sbus_reg_read(data, HOST_CTRL_REG_ADDR, &val);
	if (err) {
		return err;
	}

	val ^= BTSDIO_REG_DATA_VALID;
	return sbus_reg_write(data, HOST_CTRL_REG_ADDR, val);
}
```

**Step 2: Commit**

```bash
git add drivers/bluetooth/hci/hci_cyw43_sbus.c
git commit -m "$(cat <<'EOF'
drivers: bluetooth: hci: add CYW43 shared-bus backplane primitives

Add the foundation of the CYW43xxx shared-bus BT HCI driver with
backplane register and memory read/write functions. These map the
pico-sdk's cyw43_ll_* primitives to WHD's
whd_bus_transfer_backplane_bytes().

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 5: Driver Implementation — Init, FW Download, Buffer Setup

Add the init sequence: firmware download, BT ready wait, circular buffer initialization, and wake/ready signaling.

**Files:**
- Modify: `drivers/bluetooth/hci/hci_cyw43_sbus.c` (append after backplane primitives)

**Step 1: Add firmware download and init functions**

Append to `hci_cyw43_sbus.c` after `sbus_toggle_bt_intr()`:

```c
/*
 * Download BT firmware to the CYW43439 via backplane memory writes.
 *
 * The HCD firmware blob format (non-hex variant) is:
 *   [version_len] [version_string\0] [record_count]
 *   followed by records: [num_bytes] [addr_hi] [addr_lo] [type] [data...]
 *
 * We write each data record to BTFW_MEM_OFFSET + dest_addr, with 4-byte
 * alignment handling matching cybt_fw_download() in the pico-sdk.
 */
static int sbus_fw_download(struct cyw43_sbus_data *data)
{
	const uint8_t *p = brcm_patchram_buf;
	uint8_t write_buf[BTFW_MAX_BLOCK_SIZE + BTFW_SD_ALIGN];
	uint8_t hex_buf[BTFW_MAX_BLOCK_SIZE];
	uint8_t version_len;
	uint8_t num_bytes, type;
	uint16_t addr;
	uint32_t dest_addr;
	uint16_t hi_addr = 0;
	int addr_mode = 0; /* 0=extended, 1=extended, 2=segment, 3=linear32 */
	uint32_t abs_base_addr32 = 0;
	uint8_t *mem_ptr;
	int err;

	if (brcm_patch_ram_length == 0) {
		LOG_ERR("BT firmware blob is empty");
		return -ENOENT;
	}

	/* Skip version string: [len] [string\0] [record_count] */
	version_len = *p;
	LOG_INF("BT FW version: %s", p + 1);
	p += version_len + 1; /* skip version string */
	p += 1;               /* skip record count */

	/* Power up BT */
	err = sbus_reg_write(data, BTFW_MEM_OFFSET + BT2WLAN_PWRUP_ADDR,
			     BT2WLAN_PWRUP_WAKE);
	if (err) {
		return err;
	}

	/* Align write buffer */
	mem_ptr = write_buf;
	if ((uintptr_t)mem_ptr % BTFW_SD_ALIGN) {
		mem_ptr += BTFW_SD_ALIGN - ((uintptr_t)mem_ptr % BTFW_SD_ALIGN);
	}

	while (true) {
		/* Read 4-byte record header */
		num_bytes = *p++;
		addr = (*p++) << 8;
		addr |= *p++;
		type = *p++;

		if (num_bytes == 0) {
			break;
		}

		memcpy(hex_buf, p, num_bytes);
		p += num_bytes;

		/* Process address type */
		if (type == 4) { /* extended address */
			hi_addr = (hex_buf[0] << 8) | hex_buf[1];
			addr_mode = 1;
			continue;
		} else if (type == 2) { /* segment address */
			hi_addr = (hex_buf[0] << 8) | hex_buf[1];
			addr_mode = 2;
			continue;
		} else if (type == 5) { /* absolute 32-bit address */
			abs_base_addr32 = (hex_buf[0] << 24) |
					  (hex_buf[1] << 16) |
					  (hex_buf[2] << 8) |
					  hex_buf[3];
			addr_mode = 3;
			continue;
		} else if (type != 0) { /* not data */
			continue;
		}

		/* Type 0: data record */
		dest_addr = addr;
		if (addr_mode == 1) {
			dest_addr += (uint32_t)hi_addr << 16;
		} else if (addr_mode == 2) {
			dest_addr += (uint32_t)hi_addr << 4;
		} else if (addr_mode == 3) {
			dest_addr += abs_base_addr32;
		}

		uint32_t fw_addr = BTFW_MEM_OFFSET + dest_addr;
		uint32_t write_len = 0;
		uint32_t pad;

		/* Align start address to 4 bytes */
		if (fw_addr % 4) {
			uint32_t start_data;

			pad = fw_addr % 4;
			fw_addr &= ~3UL;

			err = sbus_mem_read(data, fw_addr,
					    (uint8_t *)&start_data, 4);
			if (err) {
				return err;
			}

			for (uint32_t i = 0; i < pad; i++) {
				mem_ptr[write_len++] =
					((uint8_t *)&start_data)[i];
			}
		}

		memcpy(&mem_ptr[write_len], hex_buf, num_bytes);
		write_len += num_bytes;

		/* Align end to 4 bytes */
		uint32_t end_addr = fw_addr + write_len;

		if (end_addr % 4) {
			uint32_t end_data;

			err = sbus_mem_read(data, end_addr & ~3UL,
					    (uint8_t *)&end_data, 4);
			if (err) {
				return err;
			}

			for (uint32_t i = end_addr % 4; i < 4; i++) {
				mem_ptr[write_len++] =
					((uint8_t *)&end_data)[i];
			}
		}

		/* Write to BT firmware memory */
		err = sbus_mem_write(data, fw_addr, mem_ptr, write_len);
		if (err) {
			return err;
		}
	}

	LOG_INF("BT FW download complete");
	return 0;
}

static int sbus_wait_bt_ready(struct cyw43_sbus_data *data)
{
	uint32_t val;
	int err;

	k_sleep(K_MSEC(FW_READY_WAIT_MS));

	for (int i = 0; i < FW_READY_POLL_COUNT; i++) {
		err = sbus_reg_read(data, BT_CTRL_REG_ADDR, &val);
		if (err) {
			return err;
		}

		if (val & BTSDIO_REG_FW_RDY) {
			return 0;
		}

		k_sleep(K_MSEC(FW_READY_POLL_INTERVAL_MS));
	}

	LOG_ERR("BT FW ready timeout");
	return -ETIMEDOUT;
}

static int sbus_wait_bt_awake(struct cyw43_sbus_data *data)
{
	uint32_t val;
	int err;

	for (int i = 0; i < BT_AWAKE_POLL_COUNT; i++) {
		err = sbus_reg_read(data, BT_CTRL_REG_ADDR, &val);
		if (err) {
			return err;
		}

		if (val & BTSDIO_REG_BT_AWAKE) {
			return 0;
		}

		k_sleep(K_MSEC(BT_AWAKE_POLL_INTERVAL_MS));
	}

	LOG_ERR("BT awake timeout");
	return -ETIMEDOUT;
}

static int sbus_set_bt_awake(struct cyw43_sbus_data *data, bool awake)
{
	uint32_t val;
	int err;

	err = sbus_reg_read(data, HOST_CTRL_REG_ADDR, &val);
	if (err) {
		return err;
	}

	uint32_t new_val = awake ? (val | BTSDIO_REG_WAKE_BT)
				 : (val & ~BTSDIO_REG_WAKE_BT);

	if (new_val != val) {
		return sbus_reg_write(data, HOST_CTRL_REG_ADDR, new_val);
	}

	return 0;
}

static int sbus_init_buffer(struct cyw43_sbus_data *data)
{
	uint32_t ram_base;
	int err;

	err = sbus_reg_read(data, WLAN_RAM_BASE_REG_ADDR, &ram_base);
	if (err) {
		return err;
	}

	LOG_DBG("WLAN RAM base: 0x%08x", ram_base);

	data->buf.h2b_buf_addr = ram_base + BTSDIO_OFFSET_HOST_WRITE_BUF;
	data->buf.b2h_buf_addr = ram_base + BTSDIO_OFFSET_HOST_READ_BUF;
	data->buf.h2b_in_addr = ram_base + BTSDIO_OFFSET_HOST2BT_IN;
	data->buf.h2b_out_addr = ram_base + BTSDIO_OFFSET_HOST2BT_OUT;
	data->buf.b2h_in_addr = ram_base + BTSDIO_OFFSET_BT2HOST_IN;
	data->buf.b2h_out_addr = ram_base + BTSDIO_OFFSET_BT2HOST_OUT;

	/* Zero all buffer pointers */
	uint32_t zero = 0;

	sbus_reg_write(data, data->buf.h2b_in_addr, zero);
	sbus_reg_write(data, data->buf.h2b_out_addr, zero);
	sbus_reg_write(data, data->buf.b2h_in_addr, zero);
	sbus_reg_write(data, data->buf.b2h_out_addr, zero);

	return 0;
}
```

**Step 2: Commit**

```bash
git add drivers/bluetooth/hci/hci_cyw43_sbus.c
git commit -m "$(cat <<'EOF'
drivers: bluetooth: hci: add CYW43 shared-bus init and FW download

Add firmware download via backplane memory writes, BT ready/awake
polling, and circular buffer initialization. Ported from pico-sdk
cybt_shared_bus_driver.c and cybt_shared_bus.c.

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 6: Driver Implementation — TX, RX, and HCI Driver API

Add the circular buffer TX/RX functions, the RX polling thread, and the Zephyr BT HCI driver API methods.

**Files:**
- Modify: `drivers/bluetooth/hci/hci_cyw43_sbus.c` (append after init functions)

**Step 1: Add TX/RX and driver API**

Append to `hci_cyw43_sbus.c`:

```c
/*
 * Write an HCI packet to the Host→BT circular buffer.
 *
 * Format: [len_lo, len_hi, 0x00, hci_pkt_type, payload...]
 * The 4-byte header is prepended to the HCI payload.
 */
static int sbus_hci_write(struct cyw43_sbus_data *data, uint8_t pkt_type,
			   const uint8_t *payload, uint16_t len)
{
	uint32_t buf_ptrs[4];
	uint32_t h2b_in, h2b_out, space;
	uint8_t __aligned(4) hdr[4];
	uint32_t total_len;
	int err;

	hdr[0] = len & 0xFF;
	hdr[1] = (len >> 8) & 0xFF;
	hdr[2] = 0x00;
	hdr[3] = pkt_type;

	total_len = ROUNDUP4(len + 4);

	k_mutex_lock(&data->bus_mutex, K_FOREVER);

	/* Wake BT and wait for it to be awake */
	sbus_set_bt_awake(data, true);
	sbus_wait_bt_awake(data);

	/* Read buffer pointers */
	err = sbus_mem_read(data, data->buf.h2b_in_addr, (uint8_t *)buf_ptrs,
			    sizeof(buf_ptrs));
	if (err) {
		goto out;
	}

	h2b_in = buf_ptrs[0];
	h2b_out = buf_ptrs[1];
	space = CIRC_BUF_SPACE(h2b_in, h2b_out);

	if (total_len > space) {
		LOG_ERR("H2B buffer full: need %u, have %u", total_len, space);
		err = -ENOMEM;
		goto out;
	}

	/*
	 * Write header + payload to circular buffer. We write header
	 * and payload as separate mem_writes because the payload from
	 * net_buf may not be 4-byte aligned, but the 4-byte header
	 * ensures the payload starts at an aligned offset in the buffer.
	 */
	if (h2b_in + total_len <= BTSDIO_FWBUF_SIZE) {
		/* No wrap needed */
		err = sbus_mem_write(data, data->buf.h2b_buf_addr + h2b_in,
				     hdr, 4);
		if (!err && len > 0) {
			/* Write payload, padding to 4-byte alignment */
			uint8_t __aligned(4) pad_buf[BTFW_MAX_BLOCK_SIZE];

			memcpy(pad_buf, payload, len);
			if (len % 4) {
				memset(pad_buf + len, 0, 4 - (len % 4));
			}
			err = sbus_mem_write(data,
					     data->buf.h2b_buf_addr + h2b_in + 4,
					     pad_buf, ROUNDUP4(len));
		}
	} else {
		/*
		 * Wrap around: write what fits at the end,
		 * then continue from the beginning.
		 */
		uint32_t first = BTSDIO_FWBUF_SIZE - h2b_in;
		uint8_t __aligned(4) wrap_buf[BTFW_MAX_BLOCK_SIZE + 4];

		memcpy(wrap_buf, hdr, 4);
		memcpy(wrap_buf + 4, payload, len);
		if ((len + 4) % 4) {
			memset(wrap_buf + 4 + len, 0, 4 - ((len + 4) % 4));
		}

		if (first >= 4) {
			err = sbus_mem_write(data,
					     data->buf.h2b_buf_addr + h2b_in,
					     wrap_buf, first);
			if (!err && total_len > first) {
				err = sbus_mem_write(data,
						     data->buf.h2b_buf_addr,
						     wrap_buf + first,
						     total_len - first);
			}
		} else {
			err = sbus_mem_write(data, data->buf.h2b_buf_addr,
					     wrap_buf, total_len);
		}
	}

	if (err) {
		goto out;
	}

	/* Update write pointer */
	uint32_t new_in = (h2b_in + total_len) & (BTSDIO_FWBUF_SIZE - 1);

	err = sbus_reg_write(data, data->buf.h2b_in_addr, new_in);
	if (err) {
		goto out;
	}

	err = sbus_toggle_bt_intr(data);

out:
	k_mutex_unlock(&data->bus_mutex);
	return err;
}

/*
 * Read one HCI packet from the BT→Host circular buffer.
 * Returns the number of bytes read (including the 4-byte header),
 * or 0 if no data is available.
 */
static int sbus_hci_read(struct cyw43_sbus_data *data, uint8_t *buf,
			  uint32_t buf_size, uint32_t *out_len)
{
	uint32_t buf_ptrs[4];
	uint32_t b2h_in, b2h_out, count;
	uint8_t __aligned(4) hdr[4];
	uint32_t hci_len, read_len;
	int err;

	*out_len = 0;

	k_mutex_lock(&data->bus_mutex, K_FOREVER);

	sbus_set_bt_awake(data, true);
	sbus_wait_bt_awake(data);

	/* Read all 4 buffer pointers in one transfer */
	err = sbus_mem_read(data, data->buf.h2b_in_addr, (uint8_t *)buf_ptrs,
			    sizeof(buf_ptrs));
	if (err) {
		goto out;
	}

	b2h_in = buf_ptrs[2];
	b2h_out = buf_ptrs[3];
	count = CIRC_BUF_CNT(b2h_in, b2h_out);

	if (count == 0) {
		sbus_toggle_bt_intr(data);
		goto out;
	}

	/* Read 4-byte header */
	read_len = 4;
	if (read_len > count) {
		read_len = count;
	}

	if (b2h_out + read_len <= BTSDIO_FWBUF_SIZE) {
		err = sbus_mem_read(data, data->buf.b2h_buf_addr + b2h_out,
				    hdr, read_len);
	} else {
		uint32_t first = BTSDIO_FWBUF_SIZE - b2h_out;

		err = sbus_mem_read(data, data->buf.b2h_buf_addr + b2h_out,
				    hdr, first);
		if (!err) {
			err = sbus_mem_read(data, data->buf.b2h_buf_addr,
					    hdr + first, read_len - first);
		}
	}
	if (err) {
		goto out;
	}

	b2h_out = (b2h_out + ROUNDUP4(read_len)) & (BTSDIO_FWBUF_SIZE - 1);

	/* Parse header: [len_lo, len_hi, len_b2, pkt_type] */
	hci_len = hdr[0] | ((uint32_t)hdr[1] << 8) | ((uint32_t)hdr[2] << 16);

	if (hci_len == 0) {
		sbus_reg_write(data, data->buf.b2h_out_addr, b2h_out);
		sbus_toggle_bt_intr(data);
		goto out;
	}

	if (hci_len + 4 > buf_size) {
		LOG_ERR("RX packet too large: %u", hci_len);
		sbus_reg_write(data, data->buf.b2h_out_addr, b2h_out);
		sbus_toggle_bt_intr(data);
		err = -ENOMEM;
		goto out;
	}

	/* Store header in output buffer */
	memcpy(buf, hdr, 4);

	/* Read payload */
	read_len = ROUNDUP4(hci_len);
	count = CIRC_BUF_CNT(b2h_in, b2h_out);
	if (read_len > count) {
		read_len = count;
	}

	if (b2h_out + read_len <= BTSDIO_FWBUF_SIZE) {
		err = sbus_mem_read(data, data->buf.b2h_buf_addr + b2h_out,
				    buf + 4, read_len);
	} else {
		uint32_t first = BTSDIO_FWBUF_SIZE - b2h_out;

		err = sbus_mem_read(data, data->buf.b2h_buf_addr + b2h_out,
				    buf + 4, first);
		if (!err) {
			err = sbus_mem_read(data, data->buf.b2h_buf_addr,
					    buf + 4 + first,
					    read_len - first);
		}
	}
	if (err) {
		goto out;
	}

	b2h_out = (b2h_out + read_len) & (BTSDIO_FWBUF_SIZE - 1);

	/* Update read pointer */
	sbus_reg_write(data, data->buf.b2h_out_addr, b2h_out);
	sbus_toggle_bt_intr(data);

	*out_len = hci_len + 4;

out:
	k_mutex_unlock(&data->bus_mutex);
	return err;
}

/*
 * RX polling thread. Reads HCI packets from the BT→Host circular buffer
 * and delivers them to the Zephyr BT host stack.
 */
static void cyw43_sbus_rx_thread(void *p1, void *p2, void *p3)
{
	const struct device *dev = p1;
	struct cyw43_sbus_data *data = dev->data;
	uint8_t __aligned(4) rx_buf[BTSDIO_FWBUF_SIZE];
	uint32_t rx_len;
	int err;

	while (data->running) {
		err = sbus_hci_read(data, rx_buf, sizeof(rx_buf), &rx_len);
		if (err) {
			LOG_ERR("RX read error: %d", err);
			k_sleep(K_MSEC(RX_POLL_INTERVAL_MS));
			continue;
		}

		if (rx_len == 0) {
			k_sleep(K_MSEC(RX_POLL_INTERVAL_MS));
			continue;
		}

		/* rx_buf[3] is the HCI packet type */
		uint8_t pkt_type = rx_buf[3];
		uint8_t *pkt_data = rx_buf + 4;
		uint32_t pkt_len = rx_len - 4;
		struct net_buf *buf;

		switch (pkt_type) {
		case BT_HCI_H4_EVT:
			buf = bt_buf_get_evt(pkt_data[0], false, K_NO_WAIT);
			break;
		case BT_HCI_H4_ACL:
			buf = bt_buf_get_rx(BT_BUF_ACL_IN, K_NO_WAIT);
			break;
		case BT_HCI_H4_ISO:
			buf = bt_buf_get_rx(BT_BUF_ISO_IN, K_NO_WAIT);
			break;
		default:
			LOG_WRN("Unknown HCI type: 0x%02x", pkt_type);
			continue;
		}

		if (!buf) {
			LOG_ERR("Failed to allocate RX buffer (type 0x%02x)",
				pkt_type);
			continue;
		}

		if (net_buf_tailroom(buf) < pkt_len) {
			LOG_ERR("RX buffer too small: %u < %u",
				net_buf_tailroom(buf), pkt_len);
			net_buf_unref(buf);
			continue;
		}

		net_buf_add_mem(buf, pkt_data, pkt_len);
		data->recv(dev, buf);
	}
}

/*
 * Zephyr BT HCI driver API
 */
static int cyw43_sbus_open(const struct device *dev, bt_hci_recv_t recv)
{
	struct cyw43_sbus_data *data = dev->data;
	whd_interface_t iface;
	int err;

	iface = airoc_wifi_get_whd_interface();
	if (!iface) {
		LOG_ERR("AIROC WiFi not initialized");
		return -ENODEV;
	}

	data->whd_drv = iface->whd_driver;
	data->recv = recv;
	data->host_ctrl_cache = 0;

	k_mutex_init(&data->bus_mutex);

	LOG_INF("Downloading BT firmware...");

	k_mutex_lock(&data->bus_mutex, K_FOREVER);

	err = sbus_fw_download(data);
	if (err) {
		LOG_ERR("FW download failed: %d", err);
		k_mutex_unlock(&data->bus_mutex);
		return err;
	}

	err = sbus_wait_bt_ready(data);
	if (err) {
		k_mutex_unlock(&data->bus_mutex);
		return err;
	}

	err = sbus_init_buffer(data);
	if (err) {
		k_mutex_unlock(&data->bus_mutex);
		return err;
	}

	err = sbus_wait_bt_awake(data);
	if (err) {
		k_mutex_unlock(&data->bus_mutex);
		return err;
	}

	uint32_t val;

	err = sbus_reg_read(data, HOST_CTRL_REG_ADDR, &val);
	if (!err) {
		/* Force a real read since cache is empty */
		whd_bus_transfer_backplane_bytes(data->whd_drv, BUS_READ,
						HOST_CTRL_REG_ADDR,
						sizeof(uint32_t),
						(uint8_t *)&val);
		data->host_ctrl_cache = val;
	}

	val = data->host_ctrl_cache | BTSDIO_REG_SW_RDY;
	sbus_reg_write(data, HOST_CTRL_REG_ADDR, val);
	sbus_toggle_bt_intr(data);

	k_mutex_unlock(&data->bus_mutex);

	/* Start RX thread */
	data->running = true;
	k_thread_create(&data->rx_thread, data->rx_stack,
			K_KERNEL_STACK_SIZEOF(data->rx_stack),
			cyw43_sbus_rx_thread, (void *)dev, NULL, NULL,
			K_PRIO_COOP(7), 0, K_NO_WAIT);
	k_thread_name_set(&data->rx_thread, "cyw43_bt_rx");

	LOG_INF("CYW43 BT HCI shared-bus opened");
	return 0;
}

static int cyw43_sbus_close(const struct device *dev)
{
	struct cyw43_sbus_data *data = dev->data;

	data->running = false;
	k_thread_join(&data->rx_thread, K_MSEC(1000));
	data->recv = NULL;

	LOG_INF("CYW43 BT HCI shared-bus closed");
	return 0;
}

static int cyw43_sbus_send(const struct device *dev, struct net_buf *buf)
{
	struct cyw43_sbus_data *data = dev->data;
	uint8_t pkt_type;
	int err;

	pkt_type = net_buf_pull_u8(buf);

	LOG_DBG("TX type=0x%02x len=%u", pkt_type, buf->len);

	err = sbus_hci_write(data, pkt_type, buf->data, buf->len);

	net_buf_unref(buf);

	return err;
}

static int cyw43_sbus_setup(const struct device *dev,
			     const struct bt_hci_setup_params *params)
{
	int err;

	/* Send HCI_RESET */
	err = bt_hci_cmd_send_sync(BT_HCI_OP_RESET, NULL, NULL);
	if (err) {
		LOG_ERR("HCI_RESET failed: %d", err);
		return err;
	}

	return 0;
}

static int cyw43_sbus_init(const struct device *dev)
{
	ARG_UNUSED(dev);
	return 0;
}

static DEVICE_API(bt_hci, cyw43_sbus_api) = {
	.open = cyw43_sbus_open,
	.close = cyw43_sbus_close,
	.send = cyw43_sbus_send,
	.setup = cyw43_sbus_setup,
};

#define CYW43_SBUS_DEVICE_INIT(inst)						\
	static struct cyw43_sbus_data cyw43_sbus_data_##inst;			\
	DEVICE_DT_INST_DEFINE(inst, cyw43_sbus_init, NULL,			\
			      &cyw43_sbus_data_##inst, NULL,			\
			      POST_KERNEL, CONFIG_KERNEL_INIT_PRIORITY_DEVICE,	\
			      &cyw43_sbus_api);

CYW43_SBUS_DEVICE_INIT(0)
```

**Step 2: Commit**

```bash
git add drivers/bluetooth/hci/hci_cyw43_sbus.c
git commit -m "$(cat <<'EOF'
drivers: bluetooth: hci: add CYW43 shared-bus TX, RX, and HCI API

Complete the CYW43xxx shared-bus BT HCI driver with:
- Circular buffer TX (Host→BT) with wrap-around handling
- Circular buffer RX (BT→Host) with polling thread
- Zephyr BT HCI driver API (open/close/send/setup)
- Device instantiation via DEVICE_DT_INST_DEFINE

The driver is now functionally complete and ready for build testing.

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 7: Build Test

Verify the driver compiles for the Pico 2W target.

**Files:** None (build verification only)

**Step 1: Build for Pico 2W with BT enabled**

From the Thermoquad workspace root, with the Zephyr venv activated:

```bash
source ../../.venv/bin/activate
export ZEPHYR_BASE="/home/kazw/Work/Thermoquad/zephyr"

# First, fetch the BT firmware blob if not already present
west blobs fetch hal_infineon

# Build the BT shell sample for Pico 2W
west build -b rpi_pico2/rp2350a/m33/w samples/bluetooth/peripheral_hr \
    -d build-bt-test \
    -- -DCONFIG_BT_CYW43_SBUS=y
```

**Step 2: Fix any compilation errors**

Address build errors iteratively. Common issues:
- Missing include paths → fix in CMakeLists.txt
- Wrong WHD type names → check `whd_types.h`
- Missing Kconfig dependencies → add to Kconfig.infineon

**Step 3: Verify build size**

```bash
west build -d build-bt-test -t rom_report
```

**Step 4: Commit any fixes**

```bash
git add -u
git commit -m "$(cat <<'EOF'
drivers: bluetooth: hci: fix CYW43 shared-bus build issues

Fix compilation issues found during build testing for
rpi_pico2/rp2350a/m33/w target.

Signed-off-by: Kaz Walker <the.kaz.walker@gmail.com>
EOF
)"
```

---

### Task 8: Push and Verify

Push all commits to the Thermoquad fork and verify the branch.

**Step 1: Push to remote**

```bash
git push thermoquad feat/bt-hci-cyw43-shared-bus
```

**Step 2: Verify commit log**

```bash
git log --oneline thermoquad/feat/bt-hci-cyw43-shared-bus --not zephyrproject-rtos/main
```

Should show 5-7 commits, all with proper `drivers: bluetooth: hci:` prefixes.

---

## Testing Strategy

This is a hardware-dependent driver. Full testing requires a Pico 2W board.

**Build verification (CI-able):**
- Compile for `rpi_pico2/rp2350a/m33/w` with BT enabled
- Verify no warnings with `-Wall -Werror`

**Hardware verification (manual, on Pico 2W):**
1. Flash `samples/bluetooth/peripheral_hr` with BT enabled
2. Check serial console for "BT FW download complete" and "CYW43 BT HCI shared-bus opened"
3. Scan from a phone — should see the "Zephyr Heartrate Sensor" advertisement
4. Connect and verify heart rate notifications flow

**Known risk areas:**
- Bus mutex contention with WiFi — if WiFi and BT are used simultaneously, backplane access serialization may cause latency
- FW download alignment — the 4-byte alignment padding logic is subtle; errors here cause silent firmware corruption
- Circular buffer wrap-around — off-by-one in pointer arithmetic causes data corruption or hangs
