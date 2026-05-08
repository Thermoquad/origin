# CYW43439 BT HCI Shared-Bus Driver — Design

## Goal

Enable Bluetooth on Raspberry Pi Pico W, Pico 2W, and RM2-based boards in
Zephyr by implementing a BT HCI driver that uses the CYW43439's shared-memory
circular buffer protocol over the SPI backplane, accessed through WHD.

## Background

The CYW43439 (used on Pico W/2W and Murata RM2 module) does not route BT UART
pins to the host MCU. Instead, BT HCI traffic goes over the same SPI bus as
WiFi via shared-memory circular buffers in the CYW43439's internal RAM.

Zephyr's existing CYW43439 BT support (`Kconfig.infineon`, `CYW43439` option)
requires `BT_H4` (UART), which is unusable on these boards. The
beechwoods-software out-of-tree driver uses pico-sdk's `cyw43-driver` directly,
but its "proprietary source-available" license blocks upstreaming.

This driver uses Infineon's WHD (Wireless Host Driver) backplane access
functions, which are already part of Zephyr's AIROC WiFi driver stack. This is
the upstream-viable path confirmed by Zephyr maintainer @dsseng in issue #53810.

## Architecture

```
┌──────────────────────────────────────────┐
│           Zephyr BT Host Stack           │
│         (bt_hci_driver_api)              │
├──────────────────────────────────────────┤
│         hci_cyw43_sbus.c                 │
│  ┌────────────┬─────────────────────┐    │
│  │  open()    │ FW download         │    │
│  │  send()    │ Circular buffer TX  │    │
│  │  setup()   │ HCI_RESET + addr    │    │
│  │  close()   │ Cleanup             │    │
│  ├────────────┴─────────────────────┤    │
│  │  RX thread (polls BT→Host buf)   │    │
│  ├──────────────────────────────────┤    │
│  │  Shared-bus primitives:          │    │
│  │  - reg_read/write (32-bit)       │    │
│  │  - mem_read/write (bulk)         │    │
│  │  - fw_download (HCD blob)        │    │
│  │  - circular buffer read/write    │    │
│  │  - wake/ready signaling          │    │
│  └──────────────────────────────────┘    │
├──────────────────────────────────────────┤
│  whd_bus_transfer_backplane_bytes()      │
│  (from AIROC WiFi / WHD)                │
├──────────────────────────────────────────┤
│     SPI transport (PIO-SPI or HW SPI)   │
│              → CYW43439                  │
└──────────────────────────────────────────┘
```

The driver is bus-agnostic. It calls WHD's `whd_bus_transfer_backplane_bytes()`
for all backplane access. WHD handles the actual SPI transport — PIO-SPI on
Pico W/2W, hardware SPI on custom boards like Hades with RM2.

## Files

| Action | Path | Purpose |
|--------|------|---------|
| Create | `drivers/bluetooth/hci/hci_cyw43_sbus.c` | Driver (~500 lines) |
| Create | `dts/bindings/bluetooth/infineon,cyw43-bt-hci-sbus.yaml` | DT binding |
| Modify | `drivers/bluetooth/hci/Kconfig.infineon` | Add `BT_CYW43_SBUS` |
| Modify | `drivers/bluetooth/hci/CMakeLists.txt` | Build rule |
| Modify | `boards/raspberrypi/rpi_pico2/rpi_pico2_rp2350a_m33_w.dts` | Add BT node |

## Device Tree

BT HCI node as child of the AIROC WiFi node:

```dts
airoc-wifi@0 {
    /* existing WiFi properties */

    bt_hci: bt-hci {
        compatible = "infineon,cyw43-bt-hci-sbus";
        status = "okay";
    };
};
```

No extra properties — the driver gets the WHD handle from its parent device.

## Kconfig

```kconfig
config BT_CYW43_SBUS
    bool "CYW43xxx Bluetooth over shared SPI bus"
    depends on WIFI_AIROC
    select BT_HAS_HCI_VS
    help
      Bluetooth HCI driver for CYW43439 using the shared SPI backplane
      transport. Used on boards where BT UART pins are not routed to
      the host (e.g., Raspberry Pi Pico W/2W, RM2 module).
```

## WHD Backplane Mapping

All pico-sdk `cyw43_ll_*` primitives map to one WHD function:

| pico-sdk | WHD equivalent |
|----------|---------------|
| `cyw43_ll_write_backplane_reg(ll, addr, val)` | `whd_bus_transfer_backplane_bytes(whd, BUS_WRITE, addr, 4, &val)` |
| `cyw43_ll_read_backplane_reg(ll, addr)` | `whd_bus_transfer_backplane_bytes(whd, BUS_READ, addr, 4, &val)` |
| `cyw43_ll_write_backplane_mem(ll, addr, sz, data)` | `whd_bus_transfer_backplane_bytes(whd, BUS_WRITE, addr, sz, data)` |
| `cyw43_ll_read_backplane_mem(ll, addr, sz, data)` | `whd_bus_transfer_backplane_bytes(whd, BUS_READ, addr, sz, data)` |

WHD handle obtained via: `airoc_wifi_get_whd_interface()` → `iface->whd_driver`
(requires `whd_int.h` internal header for struct definition — include path only).

## Init Sequence

Ported from pico-sdk `cybt_shared_bus.c`:

1. `cybt_fw_download()` — write HCD blob via backplane mem writes at `BTFW_MEM_OFFSET`
2. `cybt_wait_bt_ready()` — poll `BT_CTRL_REG_ADDR` until BT core ready
3. `cybt_init_buffer()` — read `WLAN_RAM_BASE_REG_ADDR`, compute circular buffer addresses
4. `cybt_wait_bt_awake()` — poll wake status
5. `cybt_set_host_ready()` — set bit in `HOST_CTRL_REG_ADDR`
6. `cybt_toggle_bt_intr()` — trigger BT interrupt

## Circular Buffer Protocol

4KB buffers (Host→BT, BT→Host) at offsets from WLAN_RAM_BASE.

TX packet format: `[len_lo, len_hi, 0x00, hci_pkt_type]` + HCI payload.
Write to Host→BT buffer via backplane mem write, update write pointer register.

RX: dedicated thread polls BT→Host buffer read/write pointer registers.
When data available, read payload, parse HCI type, allocate `net_buf`, deliver
via `recv()` callback. Poll interval ~2ms (matches pico-sdk pattern). No host
interrupt available for BT data on shared-bus transport.

## Constants

```c
#define BTFW_MEM_OFFSET          0x19000000
#define BT_CTRL_REG_ADDR         0x18000c7c
#define HOST_CTRL_REG_ADDR       0x18000d6c
#define WLAN_RAM_BASE_REG_ADDR   0x18000d68
#define BTSDIO_FWBUF_SIZE        0x1000
```

## Bus Locking

`k_mutex` in driver data protects all backplane access. WHD does not lock
internally at the backplane level. The mutex prevents corruption of multi-step
operations (window set + transfer).

## BT HCI Driver API

```c
static DEVICE_API(bt_hci, drv) = {
    .open  = cyw43_sbus_open,   /* init shared bus, start RX thread */
    .close = cyw43_sbus_close,  /* stop RX thread, cleanup */
    .send  = cyw43_sbus_send,   /* write to Host→BT circular buffer */
    .setup = cyw43_sbus_setup,  /* HCI_RESET, set address */
};

DEVICE_DT_INST_DEFINE(0, cyw43_sbus_init, NULL, &data, NULL,
                      POST_KERNEL, CONFIG_KERNEL_INIT_PRIORITY_DEVICE, &drv);
```

## Firmware Blob

Existing blob: `modules/hal/infineon/zephyr/blobs/img/bluetooth/firmware/COMPONENT_43439/COMPONENT_MURATA-1YN/bt_firmware.hcd`

HCD format: sequence of `[opcode_lo, opcode_hi, length, data...]` records.
Downloaded to BT core via backplane memory writes during `open()`.

## Reference Code

- `modules/hal/rpi_pico/src/rp2_common/pico_cyw43_driver/cybt_shared_bus/cybt_shared_bus_driver.c` — low-level shared bus primitives
- `modules/hal/rpi_pico/src/rp2_common/pico_cyw43_driver/cybt_shared_bus/cybt_shared_bus.c` — high-level BT bus operations
- `zephyr/drivers/bluetooth/hci/hci_infineon_cyw208xx.c` — Zephyr BT HCI driver pattern reference

## Target Boards

- Raspberry Pi Pico W (rpi_pico/rp2040/w) — if AIROC WiFi driver supports RP2040
- Raspberry Pi Pico 2W (rpi_pico2/rp2350a/m33/w) — primary development target
- Thermoquad Hades (RP2354B + RM2 module) — production target
- Any board with CYW43439 using shared-bus transport via WHD
