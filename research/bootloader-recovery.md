# Festa F65-Wall v1: bootloader and recovery research

**Date:** 2026-11-03
**Status:** Source-level findings only; no recovery method is verified.
**Safety:** No bootloader commands have been sent, no flash has been read or written by the agent, and no firmware has been installed.

## Evidence available

The official F65-Wall GPL package includes the product partition metadata at:

`images/eap/ipq50xx_festa_f65_wall_v1/rootfs/fw_data/partition-table`

It describes a 128 MiB NAND layout with redundant 3 MiB `APPSBL` and `APPSBL_1` partitions at offsets `0x00a00000` and `0x00d00000`. Both entries name `openwrt-ipq5018-u-boot.mbn` as their source artifact. This is product packaging metadata; the archive does not provide that MBN binary or establish the bootloader currently installed on the user's AP. It does not disclose U-Boot version, environment contents, boot delay, signature policy, accepted RAM-load image format, TFTP support, or recovery behavior.

The live AP reports the corresponding MTD names `0:SBL1`, `0:MIBIB`, `0:BOOTCONFIG`, `0:BOOTCONFIG1`, `0:QSEE`, `0:DEVCFG`, `0:CDT`, `0:APPSBLENV`, `0:APPSBL`, and `0:APPSBL_1`. Its current kernel command line selects `rootfs` and `ubi_rootfs`; that describes the running Linux boot and does not reveal how the bootloader selects a slot or falls back after a failed boot.

The vendor table also describes two root filesystem slots and separate factory, runtime, backup, and runtime-backup data areas. Those observations are not an OpenWrt flash map and do not authorize writes. Unique calibration, identity, and configuration data must be preserved.

## Recovery assessment

RAM boot is technically plausible on an IPQ5018 platform, but there is no device-specific evidence that this unit exposes an interruptible console, accepts unsigned images, supports TFTP, or has an accessible web recovery path. The owner confirms access is currently limited to the running Linux shell in the cloud controller; it does not provide the bootloader console or environment, and no UART or physical inspection is available in this setup.

Therefore:

- Do not try `fw_setenv`, `mtd`, `nand`, `flash`, `reboot`, or image-loading commands through the controller terminal.
- Do not interrupt boot or connect a UART adapter until the physical UART pads, logic level, and ground have been identified. Never connect UART VCC.
- Do not infer an F65-Wall recovery procedure from the ceiling Festa F65, EAP655-Wall, or Archer AX55.
- Do not create a factory image or write procedure from the observed partition names alone.

## Required evidence before a RAM boot attempt

1. Record an unmodified full UART boot log and identify the bootloader/version.
2. Identify UART voltage and pinout with physical inspection and measurement; connect GND/RX/TX only.
3. Read bootloader environment without changing it, if the console permits a demonstrably read-only command.
4. Establish boot interruption timing, accepted image format, RAM address constraints, signature checks, and available network recovery commands from observed help/output or a device-specific authoritative source.
5. Preserve and verify backups of boot metadata and unique factory/calibration data before any experiment that could affect persistent state.
6. Boot only a purpose-built initramfs after the above are confirmed; persistent installation remains a separate gate requiring explicit authorization and tested recovery.

## Phase report

**Phase:** 3, recovery and bootloader investigation.
**Status:** In progress; source review confirms dual APPSBL partitions, but the installed bootloader and a safe RAM-boot/recovery route are unknown.
**Evidence:** Vendor `partition-table`; owner-provided `/proc/mtd`, `/proc/cmdline`, and controller-terminal access description.
**Changes:** Added this research note.
**Validation:** Read-only source and terminal-output review; no hardware interaction.
**Blockers:** UART/physical inspection and bootloader console access are required to establish recovery behavior.
**Next actions:** With the owner's participation, identify UART pads/logic level and capture a receive-only boot log; do not issue bootloader commands until version and recovery behavior are understood.
