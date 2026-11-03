# Festa F65-Wall v1: stock firmware access notes

**Date:** 2026-11-03
**Scope:** Limited network reachability checks for the user-provided address `192.168.1.83`. No authentication or device commands were attempted.

## Results

| Check | Result |
|---|---|
| Route | `192.168.1.83 via 10.2.1.252 dev eth0 src 10.2.6.106` from the development environment. The target is reached through a gateway, not a directly attached `192.168.1.0/24` interface. |
| ICMP | `ping -c 2 -W 2 192.168.1.83`: 2/2 replies, 0% loss, 0.633 ms average. |
| TCP/22 | No connection; closed or filtered. |
| TCP/80 | No connection; closed or filtered. HTTP request was refused immediately. |
| TCP/443 | No connection; closed or filtered. HTTPS request timed out. |

Commands used:

```sh
ip route get 192.168.1.83
ping -c 2 -W 2 192.168.1.83
# TCP connection checks only for 22, 80, and 443
curl -k -I --connect-timeout 2 --max-time 4 http://192.168.1.83/
curl -k -I --connect-timeout 2 --max-time 4 https://192.168.1.83/
```

The ICMP response establishes network reachability from this environment, but does not identify the responding device. The closed/filtered management ports do not establish whether SSH or web management is disabled, bound to another interface, blocked by a firewall, or inaccessible from this routed development network. No port sweep, credential attempt, or change to the AP was made.

## Not collected

No stock firmware shell, `/proc` data, running DT, boot log, interface map, driver list, or bootloader environment was accessible through these checks. No conclusion about SoC, switch, flash, radio, calibration, or recovery can be drawn from ping alone.

## Festa Cloud Controller path

The owner reports that the device is managed through the Festa Cloud Controller at `https://use1-festa-controller.tplinkcloud.com/`. The login page itself does not expose readable device data to this research environment, and no account credentials were provided or used.

TP-Link's [Festa Cloud-Based Controller User Guide](https://static.tp-link.com/upload/manual/2024/202406/20240611/1910013650_Festa%20Cloud-Based%20Controller_User%20Guide_REV1.3.0.pdf) documents that:

- In **Site View → Devices**, device rows include model, firmware version, uptime, IP, status and traffic; selecting a device opens more properties.
- In **Site View → Tools → Terminal**, the controller provides a terminal for CLI or shell commands. The guide does not state which commands or shell features the F65-Wall firmware exposes, so start with harmless read-only commands and stop if the terminal shows a restricted CLI or unrecognized syntax.
- The Site View **Logs** section displays device/system events and abnormal device behavior.

This gives the owner a way to collect read-only data without opening the AP or enabling direct SSH. First capture the F65-Wall device Properties showing model, firmware/build, IP and uptime. Then, if Terminal is available for that AP, run the following individually and share redacted output:

```sh
uname -a
cat /proc/cpuinfo
cat /proc/mtd
cat /proc/partitions
cat /proc/cmdline
ip -d link show
```

If supported, also run `dmesg` and `ls /sys/firmware/devicetree/base`. These may be long; focus on SoC, memory, flash, Ethernet, switch, wireless, calibration and boot messages. Do not run any `flash`, `mtd`, `reboot`, `sysupgrade`, `fw_setenv`, `dd` write, or configuration command. Do not share serial numbers, MAC addresses, credentials, private keys, or contents of factory/calibration data.

## Runtime output collected

The owner ran `uname -a` in the controller terminal. The screenshot shows:

```text
Linux Festa F65-Wall 4.4.60 #1 SMP PREEMPT Mon Mar 11 19:24:14 CST 2024 armv7 GNU/Linux
BusyBox v1.20.2 (2024-03-11 19:33:00 CST) built-in shell (ash)
```

The shell prompt was `/bin $`. The screenshot also displayed the AP's MAC address; it is deliberately omitted from this report. This runtime evidence verifies a 32-bit ARMv7 Linux kernel (and agrees with the vendor product config's 32-bit kernel setting). It does **not** independently identify the SoC or establish the flash, switch, or radio hardware.

Next useful commands are `cat /proc/cpuinfo`, `cat /proc/mtd`, and `cat /proc/partitions`, one at a time. If `/proc/mtd` is absent or access is restricted, record that result rather than trying alternate write-capable utilities.

The owner subsequently ran `cat /proc/cpuinfo`; the redacted output reports two processors (`processor: 0` and `processor: 1`), both ARMv7 Processor rev 4, CPU architecture 7, implementer `0x51`, part `0x801`, revision 4, and `Hardware: Generic DT based system`. The serial field was all zeroes. These values confirm a two-core ARMv7 runtime but do not by themselves identify the Qualcomm SoC model.

The owner then ran `cat /proc/mtd`. Redacted transcription of the output:

```text
dev:    size     erasesize name
mtd0:   00080000 00020000  "0:SBL1"
mtd1:   00080000 00020000  "0:MIBIB"
mtd2:   00100000 00020000  "0:BOOTCONFIG"
mtd3:   00100000 00020000  "0:BOOTCONFIG1"
mtd4:   00200000 00020000  "0:QSEE"
mtd5:   00040000 00020000  "0:DEVCFG"
mtd6:   00080000 00020000  "0:CDT"
mtd7:   00440000 00020000  "0:APPSBLENV"
mtd8:   00300000 00020000  "0:APPSBL"
mtd9:   00300000 00020000  "0:APPSBL_1"
mtd10:  00100000 00020000  "oops"
mtd11:  02380000 00020000  "rootfs"
mtd12:  02380000 00020000  "rootfs_1"
mtd13:  00800000 00020000  "factory_data"
mtd14:  00c00000 00020000  "runtime_data"
mtd15:  00800000 00020000  "backup_data"
mtd16:  00600000 00020000  "runtime_backup"
mtd17:  00c00800 0001f000  "ubi_rootfs"
mtd18:  002a5dec 0001f000  "kernel"
mtd19:  004d8000 0001f000  "ubi_factory_data"
mtd20:  004d8000 0001f000  "ubi_backup_data"
mtd21:  <entry present but output truncated after colon>
```

The 17 physical partitions from `0:SBL1` through `runtime_backup`, with 0x20000 erase size, include `rootfs` and `rootfs_1` at 0x02380000 each. These names and sizes match the vendor GPL `partition-table` entries for SBL1, MIBIB, BOOTCONFIG, BOOTCONFIG1, QSEE, DEVCFG, CDT, APPSBLENV, APPSBL, APPSBL_1, oops, both file-system slots, factory_data, runtime_data, backup_data, and runtime_back. This strongly corroborates the vendor NAND layout on the running unit.

The output additionally shows `ubi_rootfs`, `kernel`, `ubi_factory_data`, and `ubi_backup_data` entries with 0x1f000 erase sizes and an incomplete final `mtd21:` line. They appear to be derived/logical MTD entries, but their exact parent mapping and safety semantics have not been established. Do not write to any of these MTDs.

The owner then ran `cat /proc/partitions`. It lists ram0–ram15 and mtdblock0–mtdblock21; the mtdblock sizes in 1 KiB units correspond to the reported MTD sizes, including mtdblock17=12290 KiB, mtdblock18=2711 KiB, mtdblock19/20=4960 KiB each, and mtdblock21=8928 KiB. This confirms an mtdblock21 exists even though the supplied `/proc/mtd` transcript ended at an incomplete `mtd21:` line. Its name and exact MTD erase/size values remain unknown until the full final `/proc/mtd` line is captured.

`/proc/partitions` does not identify the active firmware slot or mounted UBI volumes; those outputs were pending at that stage.

The owner supplied `/proc/cmdline`:

```text
console=ttyMSM0,115200n8 ubi.mtd=rootfs root=mtd:ubi_rootfs rootfstype=squashfs rootwait swiotlb=1 coherent_pool=2M
```

This confirms that the current boot attaches UBI to the MTD partition named `rootfs` and selects a SquashFS root volume named `ubi_rootfs`. This is consistent with the vendor's first rootfs slot being active for this boot. The kernel command line does not establish how the bootloader selects the alternate slot or performs fallback. The live `mount` output is still needed to verify the runtime mapping.

The owner then ran `mount`. The live root is `mtd:ubi_rootfs on / type squashfs (ro,relatime)`, confirming the selected root volume is read-only SquashFS. Separate UBI volumes are mounted read-write as UBIFS: `ubi_factory_data` at `/tp_data/manu_data`, `ubi_backup_data` at `/tp_data/backup_data`, and `ubi_runtime_data` at `/tp_data/user_data`. Their contents are sensitive and persistent; only filenames and sizes under `manu_data` were requested, and no contents were read. Their writable mounts are not permission to modify them. The owner confirmed these files in `/tp_data/manu_data`: `radio` (131072 bytes), `radio1` (131072 bytes), `default-mac` (6 bytes), `device-id` (40 bytes), `device-info` (80 bytes), `add-hardver` (4 bytes), and `region` (8 bytes). The stock rootfs startup script later confirmed `radio` feeds IPQ5018 and `radio1` feeds QCN6122; their binary contents/formats remain private and uninspected. The UBI volume names align with the vendor's factory, backup, and runtime data roles; offline analysis identifies the update package's rootfs/kernel volumes but not every live UBI attachment relationship.

The owner then listed `/tp_data/manu_data`. In addition to confirming `radio` and `radio1` are each 131072 bytes, the listing showed root ownership and restrictive permissions on several factory records (`default-mac`, `device-id`, `radio1`, and `region`). No contents were read. These are factory/provisioning data; preserve them and do not use permissions or shell access as a reason to dump or alter them.

The owner then read `/sys/firmware/devicetree/base/model`, which returned `Qualcomm Technologies, Inc. IPQ5018/AP-MP03.3`. This is live runtime device-tree evidence confirming the exact SoC/board platform string and matching the vendor GPL DTS model.

The attempted NUL-to-newline conversion command failed with `-sh: tr: not found`; this BusyBox image does not expose the `tr` applet. The owner ran `busybox --list`; it includes `awk`, but no `od`, `hexdump`, or `xxd`. No device data was changed. A read-only BusyBox awk attempt to split the property on NUL separators is:

```sh
awk 'BEGIN { RS = sprintf("%c", 0) } { print }' /sys/firmware/devicetree/base/compatible
```

If awk rejects the NUL record separator or returns no readable strings, leave the exact compatible value pending; the runtime model already confirms the IPQ5018 platform.

The awk command succeeded and returned `qcom,ipq5018-mp03.3` from `/sys/firmware/devicetree/base/compatible`, confirming the runtime compatible identifier matches the F65-Wall vendor DTS. This is direct device-tree evidence in addition to the model string.

The owner ran `free`; it reports `Mem: total 385256 KiB, used 145900 KiB, free 239356 KiB, shared 64 KiB, buffers 908 KiB`, no swap. This is Linux-visible memory, not a direct measurement of installed DRAM. TP-Link's product config sets 512 MB DDR and its DTS reserves memory for the SoC/radio firmware subsystems, so the lower Linux total is plausible, but physical RAM size should still be described as vendor-configured until corroborated by boot logs or hardware markings.

The owner supplied `ifconfig -a`. Without retaining any raw MAC addresses or traffic counters, the interface names shown were `ath0`, `ath10`, `br0` (`192.168.1.83/24`), `eth0`, `eth1`, `eth1.2`, `eth1.3`, `eth1.4`, `eth1.5`, `lo`, `soc0`, `soc1`, `wifi0`, and `wifi1`. The sanitized address relationships show that the controller-reported device MAC matches `ath0`, `br0`, `eth1`, and `eth1.2`–`.5`; `ath10` is the next address, while `eth0` is distinct. This supports using `gmac1`/`eth1` as the label-MAC interface and keeping `eth0` separate. Vendor `dev_config.json` assigns ETH0 as the uplink (`eth1.2`, `ulPortInSwitch=0`) and ETH1–ETH3 as the three downlinks (`eth1.3`–`.5`). It sets `poeOutSupp=true` and `poeOutBitMap=4`, mapping PoE pass-through to ETH3. This identifies logical roles; the exact jack locations and link behavior remain unverified.

The owner then supplied `lsmod`. Wireless modules include `wifi_2_0`, `wifi_3_0`, `qca_ol`, `umac`, `qdf`, and `ath_pktlog`; the network stack includes `qca_nss_drv`, `qca_nss_dp`, and `qca_ssdk`. This confirms the stock firmware uses Qualcomm's vendor Wi-Fi/NSS/SSDK stack. No upstream `ath11k` module is listed in the stock kernel. The module names alone do not identify the external switch, which was subsequently identified from FCC board photos.

A proposed `dmesg | grep -iE ...` filter failed because BusyBox grep 1.20.2 does not support `-E`; no log data was returned. The `grep` usage lists repeated `-e PATTERN` as supported, so retry with individual `-e` options.

The owner retried with multiple `grep -e` patterns. The returned log excerpt contains:

- One wireless PHY reporting 2.4 GHz capability (`lo_2g=2312`, `hi_2g=2732`) and no 5 GHz range.
- A second wireless PHY reporting 5 GHz capability (`lo_g5=5150`, `hi_g5=5895`) and no 2.4 GHz range.
- CNSS loading `qcn6122/bdwlan.b60` (131072 bytes) and `qcn6122/caldata_1.bin` (131072 bytes, offset `0x20000`). Do not dump or publish calibration contents.
- A vendor log warning `Failed to parse *.ini file @ QCN6122.ini; status:4`. The output does not establish whether this affects operation; both PHY attach/regulatory logs appear in the capture.
- No switch, SSDK, NSS-DP, RTL8367, QCA83xx, or MDIO identification lines in the captured filtered excerpt. Switch silicon was later identified from the FCC board photo.

To resolve the factory-file mapping without accessing the AP's private data, I parsed the published stock update's UBI image and extracted its SquashFS volume into `/tmp`. The stock startup script `/etc/rc.d/rc.ipq` reads MANU_DATA `radio` to `/tmp/IPQ5018/caldata.bin` and `radio1` to `/tmp/qcn6122/caldata_1.bin`. It also selects region-specific BDF files from `/lib/firmware/IPQ5018/bdwlan_${region}.bin` and `/lib/firmware/qcn6122/bdwlan_${region}_1.bin`. The firmware contains US, EU, CA, and JP variants, each 131072 bytes. This confirms calibration mapping and the region-dependent board-data selection. Neither proprietary BDF data nor unique calibration data was added to the repository or copied into OpenWrt packages.

The same rootfs has `/etc/Festa_F65-Wall_1.0/gpio.conf` entry `poe_port3 gpo high low 31`; `rc.modules` loads the GPIO module with that file. This assigns GPIO31 as an active-high, initially-low output associated with ETH3 PoE pass-through, and the vendor `default_config` initializes `poe_out_bitmap` to zero. However, the F65 DTS also assigns GPIO31 to an active-low Wi-Fi key. The DTS and `gpio.conf` therefore conflict; neither proves the physical GPIO function. Actual voltage, downstream power-switch circuitry, current/power budget, and GPIO behavior have not been measured. Do not toggle GPIO31.

The GPIO sources contain additional conflicts: the DTS assigns GPIO25 to reset, while `gpio.conf` assigns GPIO12 to reset; the DTS associates GPIO38 with WPS pinctrl and a USB LED; and GPIO39 is assigned to both switch reset and the active-high 5 GHz LED. REA analysis of the exact stock SSDK module confirms its initializer pulses GPIO39 as the switch reset, but static code does not resolve whether the same electrical net drives the LED or whether that LED assignment is stale. `gpio.conf` also assigns GPIO19 to a button LED and GPIO32 to a blue LED. GPIO25/GPIO12 and GPIO31 assignments remain unresolved; do not manually drive these GPIOs.

The vendor DTS explicitly labels QCN6122 as the 5 GHz radio; the other logged PHY is the integrated IPQ5018 2.4 GHz radio. The startup script's `radio`/`radio1` calibration destinations confirm which factory blob feeds each silicon target. This is strong vendor-software evidence, though not a separate RF-board marking.

Further GPL source inspection found that the F65-specific build config includes `INCLUDE_EXT_ETH_RTL8367S=y`. The selected `qcom-ipq5018-Festa_F65-Wall_1_0_0.dts` contains an enabled `qcom,ess-switch-qca83xx` node with `reset_gpio = <39>`, and FCC close-up photos confirm RTL8367S silicon. GPIO39 is also assigned to the DTS's active-high 5 GHz status LED, while the MDIO1 controller has a separate `phy-reset-gpio = <&tlmm 33 0>`. REA analysis of the exact-build `qca-ssdk.ko` confirms the module initializer drives GPIO39 high, low, then high as a switch-reset pulse, matching the switch-node property. The physical GPIO39/LED net sharing remains unresolved; the distinct GPIO33 property belongs to the MDIO1 PHY nodes. Do not manually drive either pin or copy alternate-board settings into OpenWrt.

An independently downloaded official TP-Link EAP655-Wall v1 GPL archive provides a related vendor-software reference. Its product build selects `DTBPREFIX=qcom-ipq5018-mp03.3`, enables RTL8367S, selects 512 MB DDR and two radios, and defines `RTL8367_PHY_RESET_GPIO=39`; its partition table includes factory records `radio` and `radio1`. The F65 stock SSDK module independently pulses GPIO39 as a switch reset, corroborating the software setting, though neither source resolves physical sharing with the F65's 5 GHz LED. This is not a physical measurement or proof of identical boards. No EAP655 code or board data was copied into OpenWrt.

The owner listed `/sys/bus/mdio_bus/devices`; it contains only `88000.mdio:07`. This matches the vendor DTS's MDIO0 PHY address 7. The DTS's separate MDIO1 bus and PHY addresses 0–3 do not appear as registered devices in this sysfs listing, which does not identify the switch or prove whether the second MDIO controller is active. A PHY ID/driver lookup for the registered device may clarify the direct PHY only.

The owner then read that device's `phy_id`: `0x004dd0c0`. The upstream Linux Qualcomm `at803x` PHY driver identifies `0x004dd0c0` as `IPQ5018_PHY_ID` ([upstream source](https://github.com/torvalds/linux/blob/master/drivers/net/phy/qcom/at803x.c)). This identifies the sole registered MDIO PHY as an IPQ5018 internal PHY and is consistent with the SoC confirmed by the runtime DT. It does not identify the external Ethernet switch; the vendor DTS maps MDIO0/PHY 7 to the IPQ50xx ESS side.

The owner listed `/sys/firmware/devicetree/base/soc/ess-instance`; it contains `ess-switch@0x39c00000` and `ess-switch1@1`. The owner read the second node's `/compatible` property, which returned `qcom,ess-switch-qca83xx`. This confirms the running DT selects a QCA83xx-labeled vendor switch interface for that node. Combined with the separate `qcom,ess-switch-ipq50xx` node, the runtime tree describes the SoC ESS plus an external switch. FCC board close-ups later confirmed the external switch IC as RTL8367S, showing that the OEM compatible is a vendor abstraction here.

The owner listed the contents of `/sys/firmware/devicetree/base/soc/ess-instance/ess-switch1@1`. In addition to `compatible`, `device_id`, `mdio-bus`, and `name`, the node contains `qca,ar8327-initvals`, `qcom,port_phyinfo`, `reset_gpio`, `switch_access_mode`, and the `switch_cpu_bmp`, `switch_lan_bmp`, and `switch_wan_bmp` properties. Their values are binary in sysfs and were not readable with the available BusyBox utilities. The exact-build official firmware DTB was later extracted offline and decodes these values: CPU bitmap `0x40` (vendor comment: CPU port), LAN bitmap `0x1e` (vendor comment: LAN port bitmap), WAN bitmap `0`; `qcom,port_phyinfo` maps switch port IDs 1–4 to PHY addresses 0–3. FCC board close-ups identify the switch IC as RTL8367S. The port bitmap and `port_phyinfo` establish vendor logical grouping, not the physical jack order or live PHY registration.

The owner attempted `ls /sys/kernel/debug` from the controller terminal, but received `Permission denied`. Although `debugfs` is mounted according to the earlier `mount` output, this controller shell cannot enumerate it. No device state was changed. Continue with readable sysfs paths instead; this access restriction does not indicate whether the kernel has switch diagnostics there.

The owner listed `/sys/bus/platform/drivers` and filtered for QCA/ESS/switch-related names. The output includes `ess_ssdk`, `qca-mdio`, and `qca-nss` (as well as PWM, PCIe PHY, SCM restart-reason, and TZ-log drivers). This confirms the vendor ESS/SSDK platform driver is registered in the running kernel, consistent with the loaded `qca_ssdk` module. The directory listing does not show whether a particular ESS device is bound to `ess_ssdk` or identify the physical external switch.

The owner then listed `/sys/bus/platform/drivers/ess_ssdk`; it contains only `bind`, `module`, `uevent`, and `unbind`, with no device symlink. Thus no platform-device binding is exposed in that driver's sysfs directory. This does not rule out SSDK control through another bus or vendor-specific registration path, and it does not identify the switch. Do not write to the driver's `bind`/`unbind` controls.

The owner filtered `/sys/bus/platform/devices` for ESS/switch/MDIO/SSDK names. The matching entries were `88000.mdio`, `90000.mdio`, `98000.ess-uniphy`, and `Fixed MDIO bus.0`; no external-switch- or SSDK-named platform device appeared. Together with the empty `ess_ssdk` binding directory, this shows that the external switch DT node is not exposed as an obvious separately bound platform device by these sysfs listings. The two MDIO controller devices exist, but the earlier `/sys/bus/mdio_bus/devices` output contained only the PHY at `88000.mdio:07`. REA later resolved the stock software path: SSDK device ID 1 uses the `mdio-bus` phandle and sends switch-management transactions to PHY address 29, without a conventional MDIO client appearing in these listings.

The owner ran `ls -l /sys/bus/platform/devices/90000.mdio`. It is a symlink to `../../../devices/platform/soc/90000.mdio`, confirming that the second MDIO platform device has a sysfs device node. This output alone does not show a bound driver; inspect the target directory's entries to determine whether a `driver` link is present.

The owner listed `/sys/devices/platform/soc/90000.mdio`; its `driver` link points to `/sys/bus/platform/drivers/qca-mdio`, and it contains an `mdio_bus` child plus an `of_node` link to `/sys/firmware/devicetree/base/soc/mdio@90000`. Thus the second MDIO controller is actively bound to the Qualcomm MDIO driver. Its bus child contents have not yet been listed, and switch/PHY enumeration on that bus remains unknown.

The owner listed `/sys/devices/platform/soc/90000.mdio/mdio_bus`; it returns only `90000.mdio`, confirming the MDIO bus object is registered under that controller. This listing does not show whether PHY or switch devices are attached beneath the bus object.

The owner listed `/sys/devices/platform/soc/90000.mdio/mdio_bus/90000.mdio`; it contains only the standard `device`, `of_node`, `power`, `subsystem`, and `uevent` entries, with no visible PHY/switch child directories. This agrees with the earlier `/sys/bus/mdio_bus/devices` listing, which showed only `88000.mdio:07`. Therefore no conventional MDIO client device is currently exposed on the `90000.mdio` bus in these sysfs views. The controller is active, but the external switch may be accessed through a vendor-specific path or otherwise not registered as an MDIO client; the evidence does not settle which.

The owner listed the live DT node `/sys/firmware/devicetree/base/soc/mdio@90000`. It contains `ethernet-phy@0` through `ethernet-phy@3`, plus controller properties including `compatible`, `reg`, `status`, `phy-reset-gpio`, and pinctrl fields. Thus the running DT declares four PHY children on this controller even though they are absent from the ordinary MDIO client listings. REA analysis later found that the stock SSDK selects this `mdio-bus` phandle for switch device ID 1 and performs management transactions at PHY address 29 (`0x1d`); raw switch access need not create a conventional MDIO client directory.

The vendor F65 DTS and extracted stock DTB define those MDIO1 PHY child addresses as 0–3 and set `phy-reset-gpio = <&tlmm 33 0>` (GPIO33, active flag 0). Separately, `ess-switch1@1` sets `reset_gpio = <0x27>` (GPIO39), also assigned to the DTS's active-high `wlan_5g` LED. REA confirms the stock SSDK module initializer pulses GPIO39 high for 500 ms, low for 200 ms, then high for 200 ms. This establishes GPIO39 as the vendor software's switch-reset action; whether the physical net also drives the LED remains unknown. GPIO33 is a separate MDIO1 PHY reset property. Do not manually drive either pin.

The owner found `/sys/ssdk` entries for `dev_id`, `dts_dump`, and counter/PHY controls, but reading `/sys/ssdk/dev_id` returns `Permission denied` in the controller shell. `/proc/driver/rtl8367s/phy` is readable and reports five entries (`1 0 0 0 0`, `1000M 10M 10M 10M 10M`, `FD HD HD HD HD`); those live status labels do not identify PHY addresses. REA analysis of the stock SSDK module independently identifies switch-management MDIO PHY address 29 (`0x1d`) on the bus selected by `ess-switch1`'s `mdio-bus` phandle. Reads from `/proc/driver/rtl8367s/init`, `port`, `sgmii`, `vlan`, and `mib` return I/O errors, so they provide no live register or CPU-link details through this terminal. No manual register write was attempted.

### REA analysis of the stock SSDK module

The official update's SquashFS contains `/lib/modules/4.4.60/kernel/qca-ssdk.ko` (1,080,068 bytes; SHA-256 `faca19679d1116f09ec8c6b94f16944283b1542786d5fe96d9f5d6c873988b6d`). I opened this exact ARM ELF module in REA/Ghidra 12.1.4 after verifying the enclosing firmware SHA-256 recorded below. The analysis is static; the module was not run in this environment.

- `ssdk_dt_parse` constructs the `ess-switch%d` node name from the SSDK device index, reads `device_id` and `switch_access_mode`, and `ssdk_plat_init` resolves the node's `mdio-bus` phandle. For the stock `ess-switch1@1` node, the module's SSDK device ID is 1 and the access mode is `mdio`.
- The Realtek `smi_read`/`smi_write` helpers call `qca_mdio_read`/`qca_mdio_write` with SSDK device ID 1 and PHY address `0x1d` (29), then access the RTL8367 register window. `rtk_switch_probe` uses these helpers to read switch identification registers. This identifies the stock software's switch-management address and is consistent with the switch not appearing as a conventional MDIO client in the controller's sysfs listing. It is software configuration evidence, not a live bus scan or DSA probe.
- The module `init_module` code sets GPIO39 high, waits 500 ms, sets it low for 200 ms, then returns it high for 200 ms before creating the `rtl8367s` proc directory. This matches the DT's `reset_gpio = <39>` and confirms the stock driver uses GPIO39 for the switch-reset pulse. After creating the directory, the same initializer calls `rtl8367s_proc_init_set`, which enters `rtk_switch_init` and `rtk_switch_probe`; the probe reads registers `0x1300` and `0x1301` through `rtl8367c_getAsicReg` and `smi_read`. It then calls the SGMII helper with `.LC227` (whose first byte is `0x31`, ASCII `1`) and the PHY helper with `linkup 1`. Those helpers call `rtk_port_macForceLinkExt_set`, `rtk_port_sgmiiNway_set`, and `rtk_port_phyEnableAll_set`. This confirms the stock driver's software initialization requests switch setup, SGMII, and PHY linkup; it does not prove the physical link succeeds. The separate `phy-reset-gpio` property on MDIO1 PHY nodes is GPIO33. Whether GPIO39 also electrically drives the 5 GHz LED remains unknown.
- A separate `rtl8367_sw_init` function contains a GPIO26 reset sequence and SGMII setup and calls the shared `rtk_switch_init`, but REA found no resolved direct callers or xrefs to this helper's entrypoint. The active module-init path reaches `rtk_switch_init` through `rtl8367s_proc_init_set`; the helper's GPIO26 sequence is not evidence for the F65 reset pin. Ghidra's call-reference coverage includes resolved direct calls; indirect or callback use remains unknown.

Key REA Evidence IDs: module overview `ev_af807ac102ebf15b75374e1796a48e182c3fa29dc9b5d016c1fdf7f6d27c5704`; module initializer `ev_6db11ba7a55c5798850adb06cbda2629996617e197068510f811c3b6d3784774` and `ev_db06427998144bddcd966b6684101c80342d47937743be5db7b319513324c9dd`; DT parsing and MDIO-bus phandle resolution `ev_9e41f693c74bea14182f929bc6ee2b017b83510dafb54b2e19f7de735d8f907d` and `ev_c985a6fd17b88a778b606c426be1b46315be731782b0ad589980f204e6fc1dcb`; SMI read/write `ev_6148f00e25738a170f72d9a31d56080770f97d6bbfca2ce9eed0a3417ec4c236` and `ev_ed769c97f7f029ef998338f1d1572e2328b628d6570391e8b4d5ff7ee7e0b48b`; active init/probe/read path `ev_d5823c09facea340c6cd354a98ed2c67affa89b8d4d384317b73edf5371d0`, `ev_24bda4bf78f30b52a2c75a81d7d02b296b4677df79a5d4a880a94138c146ff96`, `ev_a366dba3d7fe22a4ecbd5eee7ac4f70439288d702b726e7ce7daccf35c89b230`, and `ev_4f7b494f7e45b567fa908185672b85dc7ee2acc1d15162e5a732135895d621b1`; active SGMII/PHY setup `ev_f4f9a676487d2ddc5e721ec8e277ead4223082b5527138de8db75d4d36485f33`, `ev_09de24ca390faa14ab5bd9919fab8d5007ed59db935d53359b08b03f9bae4cf4`, and `ev_1e19b3db705abd457323f029dfb969e657ca1c500fbfafb33eacdac0372f4faa`; dormant helper/caller check `ev_2d27db46595c86f0f95ac71ff5d09c2bb65040d9fee22722b088391c6e873cc7`, `ev_aa91f4b5d08b1b2414b850b3cddd3ff1eaa215d47d501af48b1d23126e6958bb`, and `ev_b8cb359d0863603e5e0cef7b028c952e0e84ccb08215031e4b6128f1322c0275`. Fresh module Evidence and snapshot are preserved at `/tmp/f65-rea/qca-ssdk-validation-evidence.json` and `/tmp/f65-rea/qca-ssdk-validation-snapshot.json`; the package inventory bundle is `/tmp/f65-rea/f65-package-validation-evidence.json`. Neither firmware extracts nor calibration data were added to the repository.

An offline copy of TP-Link's official US 1.0.3 Build 20240311 update package was obtained from the [F65-Wall support page](https://www.tp-link.com/us/support/download/festa-f65-wall/v1/). REA's artifact graph independently inventoried the ZIP (SHA-256 `4625aa0fe70c96cfb9fee51dc4c17ce1c48175405e9f1978242a20e39cef2bfe`) and verified its 16,259,377-byte signed firmware entry has SHA-256 `6d19b2901f93d29de47871d6fe01aa29bae152cd4815a8e587acd5dcd2f87b65` (inventory Evidence `ev_479f0675edf53f66c6b1e459681297b5aca76d3a7ee72168f22a968024608d69`). I reran REA's Binwalk 3.1.0 scan and Unblob 26.6.4 extraction against that exact signed image. Binwalk reports one UBI signature at offset 6,164 and size 16,252,928 bytes; its size validation remains unknown (scan Evidence `ev_de75dff48b66ba892037d1ca879261f27edf0d53bcc669385abb10d94e93d298`). To let Unblob invoke the installed `unsquashfs`, the rerun used a temporary compatibility shim that removed the unsupported `-le` flag and limited extraction to one processor. REA then published 685 regular files (88,326,946 bytes), including 132 `.ko` files. The recovered `lib/modules/4.4.60/kernel/qca-ssdk.ko` is 1,080,068 bytes with SHA-256 `faca19679d1116f09ec8c6b94f16944283b1542786d5fe96d9f5d6c873988b6d`; REA reopened it as an ARM ELF and Ghidra's inventory confirms 4,113 procedures and 7,288 defined strings (extraction Evidence `ev_84ae5e8a76f0bbf6006f632672ee0f7b83c20dfa5cf45a4d47039bdcbc12d8a1`; module overview Evidence `ev_af807ac102ebf15b75374e1796a48e182c3fa29dc9b5d016c1fdf7f6d27c5704`). Coverage remains partial: seven unrecognized ranges include the outer 6,164-byte header and 285-byte trailer, opaque portions of the kernel volume and decompressed kernel, 67,210 trailing rootfs bytes, and an 8,192-byte `q6_fw.b20` region; REA also withheld 130 symlink entries. The extracted files and JSON reports remain under `/tmp/f65-rea/rea-extraction-serial-20261103` and `/tmp/f65-rea`, not the repository. The signed image's appended standalone DTB is at offset `0x00f6e050` (60,540 bytes; SHA-256 `412961df54c50ed4667f3b754a107c5392e6f710ff20aab6f4f8837ef4008012`). Decompiling and recompiling it with dtc 1.7.2 reproduced the identical DTB SHA-256; dtc emitted 647 generic warnings about vendor-specific/nonstandard properties. The model/compatible, `mdio@90000` and its PHY children, and the `ess-switch1@1` properties match the owner's runtime observations and GPL DTS. The full decompilation is in [device-tree.dts](device-tree.dts). This is a verified extract of the published stock update DTB, not a live export from RAM and not an upstream-ready board DTS.

Current upstream OpenWrt already has a read-only factory-data mount in `target/linux/qualcommax/ipq50xx/base-files/lib/preinit/09_mount_factory_data` for TP-Link EAP650 Outdoor and RE700X. Its ath11k hotplug script provides a useful handling pattern. The exact F65 stock rootfs script independently confirms `radio` maps to IPQ5018 and `radio1` to QCN6122, each 128 KiB. The radios' mapping is now established; their formats and a clean upstream board-data/calibration integration strategy remain to be worked out. The calibration bytes should remain private.

The [FCC ID 2BCGWF65WALL internal-photo PDF](https://fccid.io/2BCGWF65WALL/Internal-Photos/16-Internal-photos-7362721.pdf) is accessible through an FCC ID mirror. Its close-up photo visibly identifies the F65 external switch IC as **Realtek RTL8367S**. Broad photos show three RJ45 connectors on the main board plus one on a small separate daughterboard; vendor `dev_config.json` identifies the logical uplink/downlinks and assigns PoE pass-through to ETH3. The photos do not resolve the physical jack order or daughterboard circuitry. The Archer AX55 v1 remains a software precedent for TP-Link's OEM `qca83xx` label, but its GPIO, MDIO, port wiring, and reported alternate switch variants must not be copied to the F65.

The owner subsequently listed `/sys/firmware/devicetree/base/soc/mdio@90000`, which contains `ethernet-phy@0` through `ethernet-phy@3`. This confirms the same four node names seen in the official stock DTB. It does not prove these PHY nodes registered with Linux: the controller-terminal sysfs listing of the `90000.mdio` bus parent contains no child devices, and the prior global MDIO client listing exposed only `88000.mdio:07`. Vendor DTS and `dev_config.json` together indicate the switch's PHY addresses 0–2 serve the three downlinks, with address 3 unused; the uplink is outside the switch on the SoC's GEPHY. Upstream's RTL8365MB DSA driver places the chip's PHYs on a switch-internal MDIO bus, which may explain why these are not clients on the parent bus.

Current upstream OpenWrt provides a useful RTL8367S topology reference in `target/linux/qualcommax/dts/ipq5018-mr80x-v2.dts`. That IPQ5018 board describes an RTL8367S at MDIO1 address 29 using `realtek,rtl8365mb`, and places PHY addresses 0–3 under the switch's own `mdio` child. The RTL8365MB driver creates a separate user MDIO bus for those internal PHYs. REA's exact-build SSDK analysis independently identifies F65 switch-management PHY address 29 and reset GPIO39, so these values are no longer unknown in stock software. F65's vendor config identifies ETH0 as outside the switch and ETH1–ETH3 as downlinks, and its port-6 SGMII CPU-link configuration matches the MR80X topology. Physical jack order, GPIO39/LED net sharing, DSA operation, and port mapping remain unverified; MR80X's remaining details must not be copied.

Initial OpenWrt code now adds a read-only mount of `ubi_factory_data` at `/tmp/factory_data` for `tplink,festa-f65-wall-v1`, uses the factory `default-mac` for the LAN/label MAC on the `gmac1`/`eth1` side, and extracts `radio` for IPQ5018 and `radio1` for QCN6122. The preinit hook explicitly adds `/usr/sbin` to its local `PATH` before invoking `ubiattach`; the rebuilt rootfs contains both the hook and `ubiattach`. Stock interface MAC relationships support the `eth1` label assignment and show `ath10` one address after `ath0`, but do not establish how ath11k will handle MACs from these calibration files under OpenWrt. The matching DTS and image recipe exist, and the full F65 profile build completed successfully; neither the hook nor resulting images have been boot-tested on hardware. A hash-only comparison against every `.ipq5018` and `.qcn6122` file in `firmware/qca-wireless` at `5363395811fb3c72da0e62304a26d995c0f9c675` found no exact match for the eight stock regional `bdwlan` files. Regional BDF data is not included because redistribution rights remain unestablished. OpenWrt's package instructions identify the separate `firmware/qca-wireless` BDF submission path followed by an OpenWrt source bump and per-device package entry.

A [third-party Archer AX53 v1 OpenWrt branch](https://github.com/defencore/openwrt_tp-link_archer_ax53_eu_1.0) targets the same IPQ5018 + QCN6122 + RTL8367S component combination. It models an internal-GE-PHY uplink on `dp1` and a port-6 SGMII switch CPU link on `dp2`, consistent with F65 vendor topology evidence. Its assumed switch address `0x1c` conflicts with the F65 stock SSDK's address 29. Its reset GPIO39 matches the F65 stock module's GPIO39 pulse, but the AX53 implementation is untested and its board differs; do not copy its full topology or treat it as physical validation.

The owner captured the end of `/proc/mtd` with `cat /proc/mtd | tail -4`. The complete final entries are:

```text
mtd18: 002a5dec 0001f000 "kernel"
mtd19: 004d8000 0001f000 "ubi_factory_data"
mtd20: 004d8000 0001f000 "ubi_backup_data"
mtd21: 008b8000 0001f000 "ubi_runtime_data"
```

This confirms `mtd21` is `ubi_runtime_data`, and its 0x8b8000-byte size equals the 8928 KiB `mtdblock21` size. It aligns with the `ubi_runtime_data` volume mounted at `/tp_data/user_data`. The UBI logical entries and physical partition relationships are now named, though only read-only observation has been performed.

## Phase report

**Phase:** 2, stock firmware inspection.
**Status:** Complete using the exact published firmware build plus owner-run terminal outputs; a live DTB export was unavailable, but the extracted stock DTB matches the runtime model/compatible and observed switch nodes. Live CPU, full MTD names/sizes, MTD-blocks, active root argument, mounts, model/compatible, Linux-visible memory, interfaces, modules, wireless logs, direct PHY ID, switch properties, MDIO/ESS sysfs state, factory filenames/sizes, exact-build rootfs startup/config files, and static SSDK module behavior have been inspected. REA analysis recovers the stock switch-management MDIO address 29 and GPIO39 reset pulse. The live tree and stock DTB declare four PHY nodes under `mdio@90000`, but no registered MDIO clients have been exposed for them on that parent bus. FCC photos confirm RTL8367S. Debugfs enumeration is denied by the controller terminal.
**Validation:** Ping/TCP checks; official controller manual and support page reviewed; owner-provided outputs confirm Linux 4.4.60, BusyBox 1.20.2, two ARMv7 CPUs, runtime model/compatible `Qualcomm Technologies, Inc. IPQ5018/AP-MP03.3` / `qcom,ipq5018-mp03.3`, 385256 KiB Linux-visible RAM, full MTD names/sizes matching vendor metadata including `ubi_runtime_data` size 0x8b8000, rootfs UBI boot arguments, separate read-only root and writable UBIFS data mounts, `radio` and `radio1` files of 131072 bytes each, `eth1.2`–`.5` interfaces matching vendor logical roles, Qualcomm vendor Wi-Fi/NSS/SSDK modules, 2.4/5 GHz PHYs and QCN6122 calibration loads, MDIO PHY ID `0x004dd0c0` matching upstream `IPQ5018_PHY_ID`, and runtime switch node compatible `qcom,ess-switch-qca83xx`. REA/Ghidra static analysis of exact-build `qca-ssdk.ko` identifies SSDK device ID 1, switch-management PHY address 29, and the GPIO39 reset pulse. GPL `dev_config.json` maps ETH0 uplink outside the switch, ETH1–ETH3 downstream, and PoE bitmap 4 to ETH3; the stock DTS maps port-6 SGMII to `dp2`; stock `rc.ipq` confirms `radio`/`radio1` calibration targets and region BDF selection; `gpio.conf` associates GPIO31 with ETH3 PoE output while the DTS also assigns it to the Wi-Fi key. REA's repeat Binwalk/Unblob extraction independently confirms the UBI-to-UBIFS-to-SquashFS path and the recovered module digest despite partial coverage; the appended DTB round-trips byte-for-byte. FCC photos confirm RTL8367S.
**Next actions:** Verify the stock-derived MDIO address 29, GPIO39 reset pulse, SGMII, and port map through a safe nonpersistent boot; clarify GPIO39/LED sharing and GPIO31's conflicting Wi-Fi/PoE assignments without manually driving the pins. Establish BDF redistribution rights and the documented upstream package submission; establish bootloader/RAM-boot/recovery with physical participation.
