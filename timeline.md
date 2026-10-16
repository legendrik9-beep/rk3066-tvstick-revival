# Project Timeline

Detailed, chronological history of the project: what was done, what went
wrong, and how it was solved. Short version: [journal.md](journal.md).

> **Sources and reliability:** Based on the author's project log. Raw logs
> are not published. ⚠️ marks inferred or unverified results. Day assignments
> for 15/16 Oct are approximate. Binary artifacts (loader, partition images,
> kernel modules) are not published, only the findings.

## Overview

| Phase | Topic | Date | Result |
|---|---|---|---|
| 0 | Reconnaissance | 2026-10-14 | Dumps and backups of the stock system |
| 1 | Repository and docs | 2026-10-14/15 | README, publishing rules |
| 2 | Build environment | 2026-10-15 | WSL2 toolchain, first kernel |
| 3 | Device tree | 2026-10-15 | `rk3066a-hotack.dts` |
| 4 | Recovery loop | 2026-10-15/16 | Loader SHA-1 check and flashing traps solved |
| 5 | Mainline boot | 2026-10-16 | Stage A reached |

---

## Phase 0: Reconnaissance (2026-10-14)

- **Goal**
  - Collect everything needed for the hardware picture and the device tree
    without modifying the device.
- **Done**
  - Opened the case, photographed the PCB, noted chip markings
    (SoC `ANCG606B 1311`, 4x MIRA `PDJ2108DEBG`, Toshiba `TC58TEG5DCJTA00`,
    Realtek `RTL8188ETV`). *(Exact date not recorded, backfilled.)*
  - Enabled USB debugging. ADB works and `adbd` already runs as `uid=0`.
  - UART not done (no adapter, pads not identified).
  - Run 1 (`-ScanI2C`): system info, kernel logs, I2C scan, partition images
    via `/proc/mtd`. Output: `dump_20261014_202125`.
  - Run 2 (`-RegDump`): same plus register dumps (CRU, GRF, DDR PCTL/PUBL,
    GPIO). Output: `dump_20261014_204231`.
  - Pulled 8 partition images with MD5 sums (misc, kernel, boot, recovery,
    backup, cache, kpanic, system). `userdata` and `user` were skipped.
- **Problem: `host_md5.txt` was empty**
  - The comparison was not visible in the script output.
  - Solution: recompute the hashes on the host.
    ⚠️ Do it yourself with `Get-FileHash -Algorithm MD5` before trusting the backups.
- **Findings**
  - Board `rk30sdk`, Android 4.2.2, kernel `3.0.36+ #217 SMP PREEMPT` (Jul 2013).
  - PMIC: TI TPS65910 on I2C bus 1, address `0x2d`.
  - RAM address space `0x60000000`-`0x9FFFFFFF` (1 GiB).
  - Boot image format: `ANDROID!`, page size 16384, kernel `0x60408000`,
    ramdisk `0x62000000`, tags `0x60088000`.
  - Partition table from `cmdline` (`mtdparts`), including an unexplained
    256 MiB gap between `system` and `user`.

---

## Phase 1: Repository and documentation (2026-10-14/15)

- **Done**
  - Created the repo and wrote the README with a verified/inferred status column.
  - Fixed an offset error in the partition table
    (`user` starts at 3252 MiB, not 2996 MiB).
- **Decision: what may be published**
  - Yes: sanitized text dumps, scripts, DTS, kernel config.
  - No: partition images, loader, firmware blobs, kernel modules
    (copyright, personal data, size).
- **Open check**
  - An early analysis claimed that shorting NAND pins 8/9 forces Maskrom mode.
    This conflicts with the usual TSOP48 pinout.
  - ⚠️ **Not verified.** Check against the datasheet before anyone relies on it.

---

## Phase 2: Build environment and first kernel (2026-10-15)

- **Setup**
  - WSL2 Ubuntu with `gcc-arm-linux-gnueabihf`, `device-tree-compiler`, `bison`,
    `flex`, `libssl-dev`, `bc`, `u-boot-tools`.
- **Problem: slow and case-insensitive builds**
  - Cause: the source tree on NTFS (accessed via 9P).
  - Solution: clone mainline Linux 6.6 LTS into native WSL2 ext4 storage.
- **Kernel config**
  - Base: `multi_v7_defconfig`
  - Boot: `ARM_APPENDED_DTB`, `ARM_ATAG_DTB_COMPAT`, `ARM_ATAG_DTB_COMPAT_CMDLINE_EXTEND`
  - Display/GPU: `DRM_ROCKCHIP`, `ROCKCHIP_RK3066_HDMI`, `DRM_DISPLAY_CONNECTOR`, `DRM_LIMA`
  - PMIC: `MFD_TPS65910`, `REGULATOR_TPS65910`
  - Console: `FRAMEBUFFER_CONSOLE`, `LOGO`, `USB_G_SERIAL`
  - Userspace: static BusyBox initramfs built into the `zImage`

---

## Phase 3: Device tree from the stock kernel (2026-10-15)

- **Problem**
  - The stock 3.0 kernel has no device tree. Hardware is described in board
    code, so GPIO assignments cannot simply be read.
- **Approach**
  - Disassembled `rk30_gpio_init` in the stock `kernel.img`.
  - The last GPIO IRQ is 367 (`0x16f`) across 176 GPIOs, which gives a
    **GPIO IRQ base offset of 192**.
  - Combined with `/proc/interrupts` and `regs/gpio6.txt` (pins are inputs):

    | IRQ | Function | Derived pin | Status |
    |---|---|---|---|
    | 353 | IR receiver (`remotectl`) | GPIO6_A1 | ⚠️ |
    | 354 | recovery button (`play`) | GPIO6_A2 | ⚠️ |
    | 355 | `usb_detect` | GPIO6_A3 | ⚠️ |
    | 356 | TPS65910 interrupt | GPIO6_A4 | ⚠️ |

  - ⚠️ `353 - 192 = 161` would be bank 5, pin 1 with 32 pins per bank, not
    bank 6. RK30 numbering may differ, so re-check against the stock kernel's
    GPIO bank table before reusing.
- **Result**
  - `rk3066a-hotack.dts` based on `rk3066a.dtsi`:
    - TPS65910 on `&i2c1`
    - Mali-400 node
    - VOP0 + HDMI pipeline
    - hardware watchdog disabled (prevents resets during initramfs tests)

---

## Phase 4: First boot attempts and the recovery loop (2026-10-15/16)

- **Problem**
  - Booting the custom image from `recovery` showed the stock Android recovery
    animation instead of Linux, and `recovery` was overwritten with the stock
    image again.
  - Two independent causes were found.

- ### Cause 1: the loader checks a SHA-1 of the boot image
  - **Analysis**
    - Extracted `RK30xxLoader(L)_V1.32` from the `backup` partition, decrypted
      its sectors, disassembled `FlashBoot`.
    - `FlashBoot` computes a SHA-1 over kernel data/size and ramdisk data/size
      (plus further header fields) and compares it with the 32-byte `id` field
      at header offset `0x240`.
    - On mismatch ("SHA ERROR!") it takes the "recover-recovery" path: it
      extracts the stock `recovery.img` from the `RKAF` container in `backup`,
      flashes it back to `recovery` and boots it.
  - **Solution**
    - `pack_recovery_image.py` writes the expected SHA-1 (20 bytes + 12 zero
      bytes) at offset `0x240`.
    - Parameters: `ANDROID!`, page size 16384, kernel address `0x60408000`.
  - **Side effect**
    - The mechanism is a safety net: a broken `recovery` image is replaced
      automatically.

- ### Cause 2: flashing from stock Android silently fails or corrupts
  - **Trap 1: `su -c` argument parsing**
    - Stock `su` fails with `exec failed ... No such file or directory` for a
      quoted multi-word command.
    - Workaround: not needed, `adbd` already runs as root.
  - **Trap 2: `flash_image` stops after 16 KB but reports success**
    - It calls `lseek(fd, 0, SEEK_CUR)` on `/dev/mtd/mtd3`. MTD character
      devices return `ESPIPE`, which it reads as end of file.
    - Only the header (16 KB) is written, exit code 0.
  - **Trap 3: `/dev/block/mtdblock3` corrupts the flash**
    - Writes go through the 4 KB page cache, but the Rockchip NAND FTL programs
      full 16 KB pages (32 sectors) and erases only at the start of each 4 MB
      eraseblock.
    - Writes split on 4 KB boundaries hit the same 16 KB page several times
      without erase (`FlashReadRetry error!!` in `dmesg`).
    - Reading back through the block device returns cached data and falsely
      matches.
    - Based on disassembly of the vendor `rk30xxnand_ko.ko`.
  - **Solution**
    - Write to the character device in 16 KB aligned chunks:

```sh
      dd if=recovery-test.img of=/dev/mtd/mtd3 bs=16384 && sync
```

  - **Verification**
    - Drop caches first, then compare:

```sh
      echo 3 > /proc/sys/vm/drop_caches
      cmp -l recovery-test.img /dev/mtd/mtd3   # image length only
```

    - Result: 0 differing bytes over 12,730,368 bytes.

---

## Phase 5: Mainline boots, Stage A complete (2026-10-16)

- **Result**
  - Mainline **Linux 6.6.159 LTS** boots from `recovery` (`mtd3`) with the
    custom device tree and embedded BusyBox initramfs.
  - Stock `kernel` (`mtd1`) and `boot` (`mtd2`) are untouched.
  - Start with `adb reboot recovery`.
- **Subsystem status**

  | Subsystem | Result | Status |
  |---|---|---|
  | CPU / SMP | Both Cortex-A9 cores online (two Tux logos) | ✅ |
  | RAM | 989 MB usable (`free -m`) | ✅ |
  | Display | `rockchip-drm` (`1010c000.vop`) + `10116000.hdmi`, 1920x1080 on HDMI-A-1, fbdev console | ✅ |
  | GPU | Lima probes Mali-400 MP4 (1 GP, 4 PP, 128 KB L2) | ✅ probe / ⚠️ rendering |
  | PMIC | TPS65910 on I2C1 (`0x2d`), 12 rails configured | ✅ |
  | USB host | Internal hub, USB flash drive, Logitech receiver (`046d:c52e`) | ✅ |
  | USB OTG | `g_serial` root shell on `/dev/ttyGS0` (Windows `COM3`, 115200) | ✅ |

- **Side result**
  - The USB gadget serial shell replaces the hardware UART for bring-up, so no
    soldering was needed so far.
- **Known issue**
  - `rockchip-pinctrl` / `gpio-keys: Error applying setting, reverse things back`
    on boot. To be fixed in the next DT revision.

---

## Open items

- [ ] Fix the pinctrl / gpio-keys error
- [ ] Re-check the GPIO bank mapping
- [ ] Lima rendering test (GLES2)
- [ ] WiFi (RTL8188ETV)
- [ ] Root FS on USB/SD instead of initramfs
- [ ] Explain the 256 MiB gap before the `user` partition
- [ ] Verify the NAND Maskrom pins against the datasheet
- [ ] UART boot log (optional, gadget serial works)
- [ ] Stage B: mainline U-Boot

## Lessons learned

1. Check the loader's behavior before debugging the kernel. The recovery loop
   had nothing to do with the kernel.
2. Verify flash writes after dropping caches and via the character device. A
   matching MD5 from `mtdblock` can be a cache artifact.
3. On Rockchip NAND, write whole 16 KB pages aligned to 4 MB erase blocks.
4. Keep a safe boot path: testing from `recovery` kept the stock system intact.
