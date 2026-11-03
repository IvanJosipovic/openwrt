# Festa F65-Wall v1: flash layout and stock image notes

**Date:** 2026-11-03
**Status:** Read-only source and firmware analysis; this is not an OpenWrt flash map.

## Physical NAND partition metadata

TP-Link's official F65-Wall v1 GPL archive contains `images/eap/ipq50xx_festa_f65_wall_v1/rootfs/fw_data/partition-table`. It declares a 128 MiB NAND device and the following physical partition entries. The owner's live `/proc/mtd` names and sizes match these entries through `runtime_back`.

| MTD | Label | Offset | Size | Live erase size |
|---:|---|---:|---:|---:|
| 0 | `0:SBL1` | `0x00000000` | `0x00080000` | `0x20000` |
| 1 | `0:MIBIB` | `0x00080000` | `0x00080000` | `0x20000` |
| 2 | `0:BOOTCONFIG` | `0x00100000` | `0x00100000` | `0x20000` |
| 3 | `0:BOOTCONFIG1` | `0x00200000` | `0x00100000` | `0x20000` |
| 4 | `0:QSEE` | `0x00300000` | `0x00200000` | `0x20000` |
| 5 | `0:DEVCFG` | `0x00500000` | `0x00040000` | `0x20000` |
| 6 | `0:CDT` | `0x00540000` | `0x00080000` | `0x20000` |
| 7 | `0:APPSBLENV` | `0x005c0000` | `0x00440000` | `0x20000` |
| 8 | `0:APPSBL` | `0x00a00000` | `0x00300000` | `0x20000` |
| 9 | `0:APPSBL_1` | `0x00d00000` | `0x00300000` | `0x20000` |
| 10 | `oops` | `0x01000000` | `0x00100000` | `0x20000` |
| 11 | `rootfs` | `0x01100000` | `0x02380000` | `0x20000` |
| 12 | `rootfs_1` | `0x03480000` | `0x02380000` | `0x20000` |
| 13 | `factory_data` | `0x05800000` | `0x00800000` | `0x20000` |
| 14 | `runtime_data` | `0x06000000` | `0x00c00000` | `0x20000` |
| 15 | `backup_data` | `0x06c00000` | `0x00800000` | `0x20000` |
| 16 | `runtime_backup` (`runtime_back` in vendor table) | `0x07400000` | `0x00600000` | `0x20000` |

The listed physical partitions end at `0x07a00000`; the vendor table declares total flash as `128M` (`0x08000000`). The remaining `0x00600000` is not assigned by these entries. Its purpose is not established here; do not assume it is user-available space.

The table also describes logical product-information records (`partition-table`, `support-list`, `product-info`, `soft-version`, `region`, `default-mac`, `radio`, `radio1`, `device-id`, `user-config`, and `mutil-log`). Their offsets and sizes are zero or virtual in the table, so these must not be treated as extra physical MTD partitions.

## Live runtime MTD and UBI devices

The owner's complete `/proc/mtd` output confirms the following additional MTD entries:

| MTD | Label | Size | Erase size | Runtime evidence |
|---:|---|---:|---:|---|
| 17 | `ubi_rootfs` | `0x00c00800` | `0x1f000` | Selected as `/`'s SquashFS source by `/proc/cmdline` and `mount` |
| 18 | `kernel` | `0x002a5dec` | `0x1f000` | Named in the live MTD table |
| 19 | `ubi_factory_data` | `0x004d8000` | `0x1f000` | Mounted read-write at `/tp_data/manu_data` |
| 20 | `ubi_backup_data` | `0x004d8000` | `0x1f000` | Mounted read-write at `/tp_data/backup_data` |
| 21 | `ubi_runtime_data` | `0x008b8000` | `0x1f000` | Mounted read-write at `/tp_data/user_data` |

These appear as UBI-derived/logical devices alongside the physical MTD partitions. Their exact parent/volume relationships and OpenWrt write semantics have not been established. Preserve all factory, radio/calibration, identity, boot-environment, backup, and runtime data. The writable mounts are not permission to modify them.

The owner confirmed `/tp_data/manu_data` contains `radio` and `radio1`, each 131072 bytes, plus small identity/product metadata files (`default-mac`, `device-id`, `device-info`, `add-hardver`, and `region`). The later directory listing confirms these live under the factory-data UBIFS mount; the radio and several identity files are root-owned with restrictive permissions. Only names, sizes, and metadata were shared; file contents were not requested or read. The stock `rc.ipq` startup script later confirmed `radio` is used for IPQ5018 and `radio1` for QCN6122. Their private binary formats and exact in-OpenWrt extraction/conversion requirements remain undetermined. Preserve the entire factory-data volume.

The current kernel command line is:

```text
console=ttyMSM0,115200n8 ubi.mtd=rootfs root=mtd:ubi_rootfs rootfstype=squashfs rootwait swiotlb=1 coherent_pool=2M
```

`/` is mounted read-only as SquashFS from `mtd:ubi_rootfs`. This proves which root is active for the current boot, not bootloader slot-selection or fallback behavior.

## Official stock update image

TP-Link's US support page lists `Festa F65-Wall(US)_V1_1.0.3 Build 20240311`, published 2024-03-28, as a 15.21 MB download. The firmware binary inside the ZIP is 16,259,377 bytes and SHA-256 `6d19b2901f93d29de47871d6fe01aa29bae152cd4815a8e587acd5dcd2f87b65`. The owner reports the same firmware build designation; the image was inspected offline and never sent to the device.

The image contains TP-Link `support-list` and `soft-version` records, UBI structures and SquashFS content. An FDT magic value occurs at offset `0x00cc2814`; its header declares total size `0x002a5dec`, numerically equal to the live `kernel` MTD size. However, generic `dtc` rejects this structure because its string-table offsets are invalid, so it is not treated here as a verified FIT/kernel image. A separate valid 60,540-byte F65 DTB is appended at offset `0x00f6e050`; its SHA-256 is `412961df54c50ed4667f3b754a107c5392e6f710ff20aab6f4f8837ef4008012`. The decompiled tree is preserved as [device-tree.dts](device-tree.dts). Runtime model/compatible and switch-node properties shared by the owner match this tree.

The filename includes `_up_signed`, and the package carries TP-Link support/version metadata. This supports treating the vendor update as a signed/wrapped image format; it does not establish the exact signature checks performed by the stock web updater or bootloader, nor does it show that OpenWrt can be installed through that updater.

An offline `ubi_reader` parse of the signed update finds one UBI image with 2 KiB minimum I/O, 128 KiB PEBs, 124 used PEBs, and two static volumes: `ubi_rootfs` reserves 100 PEBs and `kernel` reserves 22. The extracted `ubi_rootfs` volume begins with SquashFS magic (`hsqs`) and is 12,584,960 bytes; the extracted `kernel` volume is 2,776,556 bytes. These correspond closely to live logical MTD entries `ubi_rootfs` (0x00c00800) and `kernel` (0x002a5dec), respectively. The UBI image is much smaller than the physical 35.5 MiB `rootfs` partition, leaving unallocated PEB space. This clarifies the stock update's volume payloads but does not prove the bootloader's FIT parsing, slot selection, image acceptance, or update procedure. Extracted payloads remain in `/tmp` only.

The decompiled DTS was recompiled with dtc 1.7.2; despite vendor-property warnings, the resulting DTB is byte-identical to the extracted 60,540-byte DTB. The ZIP, signed firmware binary, and extracted DTB remain in `/tmp` only and are not repository deliverables. No proprietary radio firmware or calibration data was copied into the repository.

## Upstream image-builder comparison

The current upstream Archer AX55 v1 IPQ5018 recipe uses `Device/FitImageLzma` plus `Device/UbiFit`, with 128 KiB erase blocks, 2 KiB pages, 128 MiB NAND, and `IMAGE_SIZE := 43008k`. It also emits an `initramfs-factory.ubi` artifact when building an initramfs. This is a useful example of the existing `qualcommax/ipq50xx` image pipeline and a nonpersistent test artifact.

The F65's physical `rootfs` and `rootfs_1` partitions are each only `0x02380000` (35.5 MiB), and its kernel/data volumes and APPSBL offsets differ from Archer AX55. Do not copy Archer's `IMAGE_SIZE`, FIT config, partition boundaries, network DTS, or factory artifact recipe to the F65. The image pipeline is a candidate implementation mechanism only; a F65-specific image remains unbuilt and unverified.

## Safety conclusions

- The matching live layout corroborates the vendor physical partition boundaries, but it is insufficient to create a safe OpenWrt `mtdparts`, factory image, sysupgrade, or write procedure.
- Dual `rootfs` slots and dual `APPSBL` labels do not by themselves prove automatic rollback or a boot-count/fallback mechanism.
- The signed stock image's internal layout does not prove its UBI segments can be repurposed for OpenWrt.
- No flash writes, bootloader commands, or firmware installation were performed.
