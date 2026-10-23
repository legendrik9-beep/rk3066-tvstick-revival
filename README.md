# rk3066-tvstick-revival
This repo is about reverse engineering and mainline Linux bring-up for the Rockchip RK3066 Android TV box/stick


First Build published now! Check build_1.0 !!!


My specific device:

**Legend:** ✅ verified from dump or chip marking · ⚠️ inferred / not yet verified

## Device

| Component        | Details
|------------------|-------------------------------------------------------------------------------------------|
| Manufacturer/ODM | Shenzhen Hotack Technology ✅ (from `ro.build.host`, `ro.dlna.devicename` in build.prop=  |
| Product name     | Conrad Electronics SE / Model: U2A                                                         |
| Board / SDK      | `rk30sdk`, build `RK3066_R-BOX_ANDROID4.2.2-SDK_V1.0.0_130308` ✅                         |
| Stock OS         | Android 4.2.2, kernel `3.0.36+ #217 SMP PREEMPT` (Jul 4, 2013) ✅                         |

## Hardware

| Component       | Details                                                                                        | Status                                   |
|-----------------|------------------------------------------------------------------------------------------------|------------------------------------------|
| SoC             | Rockchip RK3066, marking `ANCG606B 1311`                                                       | ✅                                       |
| CPU             | Dual-core Cortex-A9; cpufreq steps 252 / 504 / 816 / 1008 / 1200 / 1272 / 1416 MHz             | ✅                                       |
| GPU             | Mali-400 MP4: GP `0x10090000`, PP0–3 `0x10098000` / `0x1009a000` / `0x1009c000` / `0x1009e000` | ✅                                       |
| RAM             | 4× MIRA `PDJ2108DEBG`; address space 0x60000000–0x9FFFFFFF (1 GiB)                             | ✅ address space / ⚠️ chip capacity      |
| DDR controller  | DesignWare PCTL `0x20020000`, PUBL `0x20040000`; register dumps in `dumps/regs/`               | ✅                                       |
| Flash           | Toshiba `TC58TEG5DCJTA00`, TSOP48, marking `NF6845 1307`                                       | ✅                                       |
| NAND ID (dmesg) | `98 d7 84 93 72`                                                                               | ✅ (interpretation ⚠️)                  |
| PMIC            | TI TPS65910 on I2C bus 1, address `0x2d` (no readable marking, identified via I2C)             | ✅ bus/address / ⚠️ chip identification |
| WiFi            | Realtek `RTL8188ETV`, Android module `8188eu.ko`                                               | ✅ chip / ⚠️ bus (USB)                  |
| Bluetooth       | not enabled (`ro.rk.bt_enable=false`)                                                          | ✅                                       |
| Audio           | HDMI audio only, no separate codec in dumps                                                    | ✅                                       |
| Input           | IR receiver (`rkxx-remotectl`), button (`rk29-keypad`)                                         | ✅                                       |
| UART            | not investigated, pads not identified                                                          | X                                         |

## Storage

Partitions from the kernel `cmdline` (`mtdparts`), sizes in 512-byte sectors:

| Name     | Offset   | Size     |
|----------|----------|----------|
| misc     | 4 MiB    | 4 MiB    |
| kernel   | 8 MiB    | 12 MiB   |
| boot     | 20 MiB   | 12 MiB   |
| recovery | 32 MiB   | 16 MiB   |
| backup   | 48 MiB   | 512 MiB  |
| cache    | 560 MiB  | 128 MiB  |
| userdata | 688 MiB  | 1536 MiB |
| kpanic   | 2224 MiB | 4 MiB    |
| system   | 2228 MiB | 768 MiB  |
| user     | 3252 MiB | rest     |

  * ⚠️ Offsets were converted from the sector values. Note the 256 MiB gap between the
       end of `system` (2996 MiB) and the start of `user` (3252 MiB); this is taken
       as-is from the cmdline and has not been explained yet.

## Boot image format ✅

From `boot.img` / `recovery.img`: magic `ANDROID!`, page size 16384,
kernel address `0x60408000`, ramdisk `0x62000000`, tags `0x60088000`.

## Dumps

Partition images are not published (copyright, personal data, size).
Only checksums are included, see `dumps/MD5SUMS.txt`.
