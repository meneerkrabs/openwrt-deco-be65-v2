# TP-Link Deco BE65 V2 no-write bring-up

## Status and scope

This is a future hardware procedure, not authorization to execute it. The
experimental BE65 V2 image must not be loaded or booted until the user makes a
new explicit decision after reviewing the final artifact hash and this
procedure. Flashing is outside the scope entirely.

The first bring-up exists only to collect evidence for the IPQ5332, QCA8386,
and QCA8084 paths. It must not make persistent changes and must not attempt to
start either Wi-Fi device.

## Build-time invariants

Before considering hardware contact, a clean, network-isolated build must prove
all of the following:

- the builder has no Docker network while compiling and validating;
- the only image-like target artifact is
  `openwrt-qualcommbe-ipq53xx-tplink_be65-v2-initramfs-uImage.itb`;
- there is no factory, sysupgrade, UBI, IMG, or persistent BIN artifact;
- the FIT has exactly one configuration, `config@BE65_2_0_0`, and it is the
  default;
- the FIT contains only `kernel-1` and `fdt-1`;
- the embedded DT identifies `TP-Link Deco BE65 V2` / `tplink,be65-v2`;
- QPIC NAND and QPIC BAM are disabled in the embedded DT;
- the integrated IPQ5332 radio, PCIe1 root controller, PCIe1 PHY, and static
  QCN9224 function are disabled;
- QCA8386 port 0 uses `dp2`, only ports 1--3 are user LAN ports, and port 4
  (WAN) plus CPU port 5 are disabled;
- all generated checksums verify before the artifact leaves the builder.

Any failed or ambiguous check stops the procedure.

## Validated off-device build record V7 (2026-11-11)

The final diagnostic artifact was built and inspected entirely off-device. The
builder was disconnected before, during, and after compilation and final
inspection (`NetworkSettings.Networks = {}`). OpenWrt's `sha256sums` file
verified every published target file.

- artifact: `openwrt-qualcommbe-ipq53xx-tplink_be65-v2-initramfs-uImage.itb`
- size: 22,247,680 bytes
- SHA-256: `8e7223c715e6fc59568842a5d4c2b2077b62614368e3010d9aeabcbd3f98a5e7`
- FIT default and sole configuration: `config@BE65_2_0_0`
- image nodes: `kernel-1` and `fdt-1`
- persistent-install artifacts: none

A deterministic inspection of the embedded DT verified that QPIC NAND, QPIC
BAM, integrated Wi-Fi, the PCIe1 root controller, its two-lane PHY at
`phy@4b1000`, and the static QCN9224 endpoint are disabled. It also verified
that QCA8386 port 0 references the proven `dp2` conduit, only `lan1`--`lan3`
are enabled user ports, and WAN/CPU port 5 remain disabled.

This hash identifies the validated artifact only. Recording or reviewing it
does not authorize contacting the router, loading the artifact, or booting or
flashing it.

## Non-negotiable no-write rules

The following commands or operations are prohibited during this bring-up:

- `saveenv`, `env save`, `fw_setenv`, or any boot-environment update;
- `nand`, `mtd`, `ubi`, `ubiformat`, `flash`, `erase`, `write`, `cp` to a flash
  address, or a vendor upgrade command;
- booting a factory, sysupgrade, UBI, or stock-remaster image;
- enabling QPIC, mounting an existing rootfs/overlay, or attaching a UBI volume;
- enabling either Wi-Fi node or substituting unreviewed firmware/calibration;
- connecting a WAN or production LAN cable;
- continuing after an unexpected reboot, watchdog reset, kernel panic, or
  storage device appears.

Power removal is the rollback. No persistent state should exist to undo.

## Authorization gates

### Gate 1: read-only bootloader inventory

This gate requires permission to contact the device over UART, but does not
permit loading or booting the experimental image.

1. Isolate the BE65 from the production network. Connect only UART and power.
2. Capture the complete power-on log without sending input.
3. If interruption is authorized, stop autoboot with the verified `tpl` token.
4. Use only read-only discovery commands exposed by that exact bootloader, such
   as `version`, `help`, `bdinfo`, and `printenv`.
5. Record the DRAM map, current load variables, boot command syntax, TFTP
   support, and whether the bootloader can inspect a FIT without writing it.
6. Power off. Review the transcript before defining a RAM load address or boot
   command.

Do not guess a load address. The FIT kernel itself targets `0x41000000`, so the
TFTP staging buffer must be proven not to overlap the decompressed kernel, DT,
bootloader, stack, reserved memory, or any other live region.

### Gate 2: RAM load and FIT inspection

This gate requires a second explicit decision based on Gate 1. It permits a
network transfer into verified DRAM but not a Linux boot.

1. Use a directly connected, isolated host with a static address and no routing
   or Internet sharing.
2. Recompute the artifact SHA-256 on that host and compare it with the reviewed
   build record.
3. Transfer the FIT only to the load address established from Gate 1 evidence.
4. Use the bootloader's read-only image-inspection command to verify all FIT
   hashes and confirm that `config@BE65_2_0_0` exists.
5. Do not run the boot command. Capture the transcript, power off, and review
   the evidence.

A transfer to DRAM is acceptable only if the UART log proves the destination
range. A timeout, short transfer, hash mismatch, unexpected configuration, or
unclear memory range is an immediate stop.

### Gate 3: one-time RAM boot

This gate requires a new explicit user decision after Gate 2. The exact command
must be filled in from the captured bootloader syntax; it must select
`config@BE65_2_0_0` explicitly and must reference only the verified RAM copy.
No command in this document authorizes that boot now.

Keep UART logging active from before the boot command until power-off. Do not
allow fallback to a flash-resident image if the RAM boot fails.

## Expected Linux observations

The serial transcript should show:

1. Linux 6.18.39 starting with the BE65 V2 DT and an embedded initramfs.
2. No MTD, NAND, UBI, ubiblock, or overlay device being registered or attached.
3. No QCN9224 MHI/QMI startup and no integrated IPQ5332 ath12k startup.
4. QCA8084 package clocks, resets, SerDes, and PHYs 1--4 probing without an
   unwind, timeout, or retry loop.
5. QCA8386 registering as a six-port DSA switch with only `lan1`, `lan2`, and
   `lan3` exposed as user ports. Autonomous switch learning and LAN-to-LAN
   hardware bridge offload remain disabled; bridging runs through the CPU.
6. No `wan` interface and no secondary CPU conduit.
7. No kernel warning, Oops, panic, lockup, or watchdog reset.

The disabled fourth PHY may still be discovered as package hardware, but its
DSA port must remain disabled and must not become `wan`.

## Read-only evidence collection

After the serial log has shown a clean boot and no persistent storage, collect
only volatile state. Suitable evidence includes:

- `cat /proc/device-tree/model`;
- `cat /proc/cmdline`;
- `cat /proc/iomem`;
- `dmesg`;
- `ip -d link show`;
- `ls -l /sys/class/net`;
- `ethtool -i lan1`, `ethtool lan1`, and equivalents for LAN2/LAN3;
- confirmation that `/sys/class/mtd` is absent or empty;
- confirmation that `/sys/class/ubi` is absent or empty;
- confirmation that no block-backed overlay is mounted.

Do not run stress tests during the first boot. Link testing starts with one
isolated cable on LAN1, then LAN2, then LAN3, one at a time. A port appearing
under the wrong label or any activity on the physical WAN port is an abort.

## Abort criteria

Power off immediately if any of the following occurs:

- any MTD/UBI device becomes writable, attaches, or is mounted;
- the FIT configuration differs from `config@BE65_2_0_0`;
- the model/compatible pair is not the BE65 V2 pair;
- either Wi-Fi device starts probing;
- WAN or CPU port 5 becomes active;
- QCA8084 cleanup, clock-parent restoration, reset, or QXGMII setup fails;
- QCA8386 reports a clock or conduit error but continues forwarding;
- the console is lost, the unit reboots, or a watchdog fires;
- any instruction would require improvising an address, command, topology, or
  polarity not already established by evidence.

After an abort, retain the UART transcript and do not retry with changed
parameters until the source or procedure has been reviewed off-device.

## Explicitly deferred work

This procedure does not qualify persistent storage, sysupgrade, the WAN port,
CPU port 5, integrated Wi-Fi, QCN9224 firmware/MHI/QMI/calibration, regulatory
operation, or production deployment. Each needs a later evidence-backed change,
a new off-device build and review, and separate hardware authorization.
