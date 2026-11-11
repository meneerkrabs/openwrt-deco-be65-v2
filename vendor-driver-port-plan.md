# Deco BE65 V2 — vendor and mainline port assessment

Date: 2026-11-10
Scope: TP-Link Deco BE65 V2 (IPQ5332 + QCA8386/QCA8084 + QCN9224),
OpenWrt and the `perceival/openwrt-flint3` `qualcommbe/ipq53xx` Linux 6.18
branch.

## Executive conclusion

There are three distinct routes:

1. **Vendor-rootfs remaster (known-working hardware path).** Keep the vendor
   5.4.213 kernel, DTB, modules, firmware and calibration; replace only the
   userland. This is currently the lowest-risk route to simultaneous 2.4/5/6
   GHz and 2.5 GbE.
2. **QSDK 12.x / OpenWrt 23.05 base.** Rebuild around the public QSDK pieces
   and retain the vendor-only Wi-Fi binaries where licensing allows. More work,
   but closer to a maintainable firmware than a rootfs-only remaster.
3. **Linux 6.18 mainline-style port.** Not ready to flash and not a quick port,
   but no longer correctly described as “QCN9224 impossible.” Public Qualcomm
   downstream ath12k code for Linux 6.1/6.6 contains an IPQ5332 + QCN9224 path.
   That code is a useful porting reference, not proof that upstream ath12k plus
   the BE65 board files will work unchanged.

The BE65 DTS has since been rewritten from board-specific evidence and is built
only into an initramfs diagnostic FIT. It remains **blocked from hardware use**:
this source assessment and an off-device build do not qualify a RAM boot or any
flash operation.

## 1. Hardware and known-good vendor stack

### Device

- SoC: Qualcomm IPQ5332.
- Ethernet: QCA8386 switch with QCA8084 PHY package.
- Wi-Fi:
  - IPQ5332 integrated 2.4 GHz radio, vendor board ID `0x16`.
  - PCIe QCN9224 dual-band 5/6 GHz radio, vendor board ID `0x1019`.
- Vendor kernel: Linux `5.4.213`, QSDK 12.1-derived modules.
- U-Boot FIT config: `config@BE65_2_0_0`.

### Vendor radio path

The vendor rootfs contains:

- `qca_ol.ko`, `umac.ko`, `wifi_3_0.ko`, `qca_spectral.ko`,
  `ipq_cnss2.ko`;
- `qcom,cnss-qcn9224` DT integration and QMI/MHI support;
- `bdwlan.b1019`, `rxgainlut.b1019`, `amss_dualmac.bin`, `m3.bin`,
  `Data_dualmac.msc`, regulatory data and per-region `multi_bdf` trees.

The vendor DTB identifies `wifi2` as QCN9224 5G+6G with board ID `0x1019`
and a reserved-memory/hremote layout. The experimental OpenWrt DTS now carries
the evidence-backed board ID, firmware directory, and active reserved-memory
slot, but disables the endpoint, PCIe1 root controller, and PCIe1 PHY.

### Vendor Ethernet path

The vendor uses the QSDK stack:

```text
qca-ssdk (QCA8386 switch + QCA8084 PHY support)
  + qca-nss-dp (netdev / EDMA / GMAC)
  + qca-nss-ppe (PPE)
  + ECM/NSS acceleration (optional for basic links)
```

The 6.18 tree now contains a dedicated QCA8386 DSA driver, QCA8084 package
support, and the conservative BE65 wiring: one proven CPU conduit (`dp2`), LAN
ports 1--3 enabled, and WAN/CPU port 5 disabled. The diagnostic driver keeps
QCA8386 autonomous learning and LAN-to-LAN bridge offload disabled because no
public MHT reference proves the ATU fast-age command; Linux bridges therefore
forward through the CPU. This is still experimental and requires off-device
validation before any separately authorized RAM boot.

## 2. What QCN9224 versus QCN9274 actually means

### Verified overlap

- PCI device ID `17cb:1109` is named `QCN9274_DEVICE_ID` by upstream ath12k.
- Qualcomm-derived downstream code also calls `0x1109` QCN9224 in MHI and
  platform patches.
- The Telecom Infra Project `wlan-ap` tree contains a Qualcomm downstream
  Wi-Fi 7 ath12k implementation for Linux 6.1 and 6.6 with:
  - explicit QCN9224 register and SRAM definitions;
  - IPQ5332 + QCN9224 DTS/reserved-memory examples;
  - `TARGET_TYPE_QCN9224` handling;
  - the same `0x1109` PCI device routed to QCN9274 hw1.0 or hw2.0 parameter
    tables according to the hardware-version register;
  - a two-radio hw2.0 profile using firmware directory `QCN92XX/hw1.0`.
- `firasshaari/openwifi-kernel-patches` documents QCN9224/QCN9274 as two
  spellings for the same silicon family in its OpenWiFi relevance index.
  That repository is a linux-wireless mailing-list mirror/index, **not** a
  bundle of the required driver patches itself.

This overturns the earlier categorical statement that ath12k has no public
QCN9224 implementation. It does **not** prove that current upstream ath12k can
boot the BE65 radio unchanged.

### Current BE65-specific implementation and remaining runtime unknowns

1. Upstream ath12k still exposes PCI ID `0x1109` as QCN9274, but the BE65 DTS
   opts this one device into `QCN92XX/hw1.0` with
   `qcom,firmware-directory`; genuine QCN9274 devices remain unchanged.
2. The DTS supplies board ID `0x1019`. Patch 301 applies that override to both
   board-data selection and the OTP-derived dual-MAC decision. The local
   `board-2.bin` uses the exact key
   `bus=pci,qmi-chip-id=0,qmi-board-id=4121`.
3. Patches 338–340 provide the directory override, separate
   `amss_dualmac.bin` fallback, and type-7 QMI download of
   `rxgainlut.b1019`. The staging script keeps all user-owned firmware out of
   Git and deliberately excludes `fw_ini_cfg.bin` until the matching AFC
   changes are ported.
4. No QRTR/MHI identity override is justified. The PCI path keeps the upstream
   QCN9274 WLFW service instance (`0x07`); a firmware-advertised
   `MULTI_QRTR_ID` feature, when present, adds the PCI-derived instance and
   programs the QRTR node register. Board ID is not part of that identity.
5. The active vendor reserved-memory window is represented, but the radio
   remains disabled. Runtime evidence is still required for firmware feature
   IEs, WLFW service discovery, MHI/QMI startup, calibration download and
   regulatory behavior.
6. PCI ID equality does not by itself prove firmware-ABI compatibility.
   Therefore no hardware boot or flash is authorized by this source-only
   implementation.

## 3. `qca_ol.ko` decompilation result

The vendor module is unusually favorable for targeted reverse engineering:

- ELF64 AArch64 relocatable kernel module;
- 4,496,688 bytes;
- SHA-256
  `d5d3ec1f96ae533c59656c8bfe3ca70dc7696a494a7047bdaed4406b385306cf`;
- not stripped;
- 4,381 named symbols, including 2,640 text functions;
- symbol and string tables are present, but no DWARF/BTF type information.

A Ghidra 12.1.4 headless proof of concept successfully decompiled selected
entry, attach, target-type and QCN9224 HAL functions. Generated files are in
`../analysis/qca_ol-decomp/`; the reusable script is
`../tools/DecompileNamedFunctions.java`.

Useful confirmed behavior includes:

- `init_module` is the symbolized `qca_ol_mod_init` path and registers the
  target/UMAC callbacks before `pld_init()`;
- `ol_ath_get_wmi_target_type` accepts target type `32`
  (`TARGET_TYPE_QCN9224`) and selects the relevant WMI flavor;
- `ol_ath_soc_attach` contains explicit target-type `0x20` branches;
- `hal_qcn9224v2_attach` installs a QCN9224-specific SRNG table and a large
  set of hardware operations;
- `__qca_ol_update_mlo_capability` exposes the MLO group/SOC setup logic.

Decompilation is therefore useful for answering narrow questions and matching
vendor behavior to public source. Reconstructing the entire 2,640-function
module into a maintainable driver would still be a major and unnecessary
project while public QSDK/downstream ath12k references exist. Ghidra output
also has guessed types and unresolved external relocations, so it is evidence,
not buildable source.

## 4. Ethernet feasibility

Mainline `qca8k` does not currently provide a drop-in QCA8386 path. The public
QSDK SSDK contains QCA8386/QCA8084 support, and this OpenWrt tree already has
part of that stack. There are also out-of-tree QCA8386/QCA8084 DSA efforts,
including an OpenWrt PR referenced below, but they need verification against
BE65’s CPU port, MDIO topology and 2.5G link setup.

Practical choices:

- **Fastest known-good:** keep vendor `qca-ssdk` + `qca-nss-dp` +
  `qca-nss-ppe`.
- **6.18 QSDK hybrid:** add/port `qca-nss-dp`, preserve SSDK/PPE and implement
  the exact vendor DTS wiring.
- **Mainline-style DSA:** evaluate the QCA8386/QCA8084 patch series, initially
  without NSS acceleration. This may give correct links before full routed
  2.5 Gb/s performance.

## 5. Safe development phases

### Phase A — preserve the working recovery/remaster path

- Use only the vendor-rootfs remaster for a hardware write.
- Flash only inactive `rootfs_1` after exact local and remote image checks.
- Use `ubiformat`, attach the new UBI, enumerate its exact volume set, verify
  type/size/content, detach, then change both boot selectors in one
  `fw_setenv --script` environment transaction.
- Keep reboot opt-in.
- Kitchen already has a full per-unit dump and proven reset-button U-Boot HTTP
  recovery. Do not overwrite its ART/factory/runtime data with another unit’s
  data. Back up Office separately before touching it.

### Phase B — correct the 6.18 DTS without flashing

This off-device phase is implemented: copied GL.iNet wiring was removed;
QCA8386/QCA8084 MDIO, conservative ports, GPIOs and the proven `dp2` conduit
were added; both radios are represented but disabled; and PCIe1 plus QPIC are
disabled at their controllers. Proprietary QCN9224 inputs remain separately
staged and ignored by version control. The resulting image remains blocked from
hardware use.

### Phase C — Ethernet bring-up

The QCA8386/QCA8084 series now compiles against Linux 6.18 and represents only
the proven LAN topology. Its first profile deliberately uses software bridging,
disables WAN/CPU port 5, and does not claim PPE/NSS throughput. Link and 2.5G
behavior still require the separately gated, no-write RAM-boot procedure after
the final off-device artifact is reviewed.

### Phase D — QCN9224 downstream-ath12k port

1. Diff TIP’s Linux 6.6 `ath12k/wifi7` implementation against this tree’s
   ath12k patches.
2. Isolate the minimum 0x1109 hw-version, QCN92XX firmware, MHI/QMI, HAL,
   target-type and reserved-memory delta.
3. Map BE65 board ID `0x1019` without modifying the calibration payload.
4. Test PCI/MHI/QMI and firmware boot with radio transmission disabled first.
5. Only after firmware and board-data loading is stable, test 5 GHz, then 6 GHz,
   regulatory behavior, AP/STA and finally MLO.

## 6. Decision matrix

| Route | Full radio | 2.5 GbE | Effort | Current status |
|---|---:|---:|---:|---|
| Vendor-rootfs remaster | Known working | Known working | Lowest | Preferred hardware-test path |
| QSDK 12.x / kernel 5.4 | Expected | Expected | Medium-high | Build/integration project |
| 6.18 + QSDK Ethernet + downstream ath12k | Plausible, unproven on BE65 | Plausible | High | Worth a research branch |
| Pure current upstream 6.18 | No proven BE65 QCN9224 path | No QCA8386 drop-in | Very high | Not a full-hardware route today |
| Full `qca_ol` reconstruction from binary | Theoretically | N/A | Extreme | Not recommended |

## 7. Source references

- Upstream ath12k PCI mapping:
  <https://codebrowser.dev/linux/linux/drivers/net/wireless/ath/ath12k/pci.c.html>
- ath12k support list:
  <https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath12k.html>
- TIP Qualcomm downstream Wi-Fi 7 tree:
  <https://github.com/Telecominfraproject/wlan-ap/tree/main/feeds/qca-wifi-7>
- TIP downstream ath12k Wi-Fi 7 implementation:
  <https://github.com/Telecominfraproject/wlan-ap/tree/main/feeds/qca-wifi-7/mac80211/src/drivers/net/wireless/ath/ath12k/wifi7>
- OpenWiFi linux-wireless patch index supplied during this investigation:
  <https://github.com/firasshaari/openwifi-kernel-patches>
- Public QSDK/QCN9224 host-common mirror:
  <https://github.com/lunatickochiya/openwrt-qsdk12>
- qca-ssdk:
  <https://git.codelinaro.org/clo/qsdk/oss/lklm/qca-ssdk.git>
- qca-nss-ppe:
  <https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-ppe.git>
- qca-nss-dp:
  <https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-dp.git>
- OpenWrt QCA8386/QCA8084 work (verify before reuse):
  <https://github.com/openwrt/openwrt/pull/25902>

## 8. Current engineering status and next gate

1. Keep every experimental image blocked from hardware use.
2. The clean V7 build completed with the builder network disconnected before,
   during, and after compilation and inspection.
3. The FIT output set and embedded DT passed deterministic checks; the final
   size and hash are recorded in `docs/be65-v2-no-write-bring-up.md`.
4. A focused independent source review found no reachable correctness defect in
   the reviewed blocker fixes. Hardware-dependent behavior remains unproven by
   design.
5. Any Gate 1 read-only router contact now requires a new explicit user
   decision; no current text authorizes router contact, loading, boot, or flash.
