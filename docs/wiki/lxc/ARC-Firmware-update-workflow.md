# Intel ARC A770 Firmware Update Workflow (host)

This documents the steps used to update the A770 LE (SOC1) firmware on **primary-host**. Run these from the **host** (not inside the `dev` LXC), because the MEI device nodes are only present on the host (`/dev/mei1`).

## Environment
- Clones: `/root/ARC-Firmware-Tool` and `/root/ARC-Firmware` (public repos). If missing, run:
  - `git clone https://github.com/Solaris17/ARC-Firmware-Tool.git /root/ARC-Firmware-Tool`
  - `git clone https://github.com/Solaris17/ARC-Firmware.git /root/ARC-Firmware`
- Use the prebuilt Linux CLI in `/root/ARC-Firmware-Tool/Linux Build/`:
  - `cd /root/ARC-Firmware-Tool/Linux\ Build`
  - `chmod +x igsc`
  - Run commands with `LD_LIBRARY_PATH=. ./igsc ...` (so it finds `libigsc.so.0`).
- Device: A770 LE is `/dev/mei1` (8086:56a0, subsys 8086:1020).

## Identify current versions
```
cd /root/ARC-Firmware-Tool/Linux\ Build
LD_LIBRARY_PATH=. ./igsc list-devices --info
LD_LIBRARY_PATH=. ./igsc fw version --device /dev/mei1
LD_LIBRARY_PATH=. ./igsc oprom-code version --device /dev/mei1
LD_LIBRARY_PATH=. ./igsc oprom-data version --device /dev/mei1
LD_LIBRARY_PATH=. ./igsc fw-data version --device /dev/mei1
```

## Pick firmware files (A770 LE / SOC1)
- Main FW: `/root/ARC-Firmware/Latest/dg2_gfx_fwupdate_SOC1.bin` (FW version DG02_1.xxxx).
- OPROM code: `/root/ARC-Firmware/Latest/dg2_c_oprom.rom`.
- OPROM data: `/root/ARC-Firmware/Latest/dg2_d_intel_a770_oprom-data.rom`.
- Config/FW data: `/root/ARC-Firmware/Latest/fwdata/dg2_intel_a770_config-data.bin` (may already match OEM; igsc will skip if identical).
- If you prefer stepping through versions, use the same filenames from the desired numbered release directory (e.g., `/root/ARC-Firmware/101.4972/…`).

## Flash sequence (used successfully)
Run from `/root/ARC-Firmware-Tool/Linux Build` with `LD_LIBRARY_PATH=.`. Order mirrors the Intel flash log sequence.

1) Main firmware (SOC1):
```
LD_LIBRARY_PATH=. ./igsc fw update --device /dev/mei1 --image /root/ARC-Firmware/Latest/dg2_gfx_fwupdate_SOC1.bin
```

2) OPROM data (skip if igsc says installed is newer/equal):
```
LD_LIBRARY_PATH=. ./igsc oprom-data update --device /dev/mei1 --image /root/ARC-Firmware/Latest/dg2_d_intel_a770_oprom-data.rom
```
Add `--allow-downgrade` if intentionally moving backward.

3) OPROM code:
```
LD_LIBRARY_PATH=. ./igsc oprom-code update --device /dev/mei1 --image /root/ARC-Firmware/Latest/dg2_c_oprom.rom
```

4) FW data / config (optional; may be identical OEM payload):
```
LD_LIBRARY_PATH=. ./igsc fw-data update --device /dev/mei1 --image /root/ARC-Firmware/Latest/fwdata/dg2_intel_a770_config-data.bin
```
If igsc reports an OEM mismatch or same version, use `--allow-downgrade` only when you deliberately want that change.

## Verify after flashing
```
LD_LIBRARY_PATH=. ./igsc list-devices --info
LD_LIBRARY_PATH=. ./igsc fw version --device /dev/mei1
LD_LIBRARY_PATH=. ./igsc oprom-code version --device /dev/mei1
LD_LIBRARY_PATH=. ./igsc oprom-data version --device /dev/mei1
LD_LIBRARY_PATH=. ./igsc fw-data version --device /dev/mei1
```

## Notes
- Current state after this run: FW `DG02_1.3266`, OPROM CODE `14 00 31 04`, OPROM DATA `14 00 2C 04`, FW Data `Major 101 / OEM 15 / VCN 1`.
- Reboot/power-cycle the host after flashing so new firmware/OPROM is loaded cleanly.
- Host is a live environment; re-clone the two repos before the next update if they are gone.

## April 2026 re-check on a live host

This workflow was re-run on a live host after enabling IOMMU, Re-Size BAR, and SR-IOV in firmware and booting a live image with `i915-sriov-dkms`.

Observed device state:

- device: `8086:56a0 / 8086:1020`
- MEI node: `/dev/mei1`
- current FW: `DG02_1.3266`
- current OPROM CODE: `14 00 36 04 00 00 00 00`
- current OPROM DATA: `14 00 2C 04 00 00 00 00`
- current FW Data: `Major 101 / OEM 15 / VCN 1`

Image comparison against `/root/ARC-Firmware/Latest`:

- main FW image matches device (`DG02_1.3266`)
- OPROM CODE image matches device (`14 00 36 04 ...`)
- OPROM DATA image matches device (`14 00 2C 04 ...`)
- FW Data reports the same visible version, but `igsc` still refuses normal update because of OEM compatibility rules

Safe no-downgrade update checks:

- `fw update`: refused because image and device are the same version
- `oprom-code update`: refused because installed version is newer or equal
- `oprom-data update`: refused because installed version is newer or equal
- `fw-data update`: refused on OEM compatibility despite matching visible version strings

Conclusion:

- for this host, the Arc A770 is **not blocked by stale A770 firmware/oprom payloads**
- reflashing the same A770 `Latest` images is not the next useful move for missing SR-IOV capability
- if experimentation continues, it should focus on:
  - platform/PCIe link training issues
  - alternative firmware/config payload hypotheses
  - not reapplying the same baseline A770 images
