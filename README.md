# Intel Arc A750 (DG2 / Alchemist) macOS Driver Port

Experimental port of [`yusufklncc/intel-gpu-macos-driver`](https://github.com/yusufklncc/intel-gpu-macos-driver) (originally targeting Battlemage / Xe2) to **Intel Arc Alchemist (DG2-G10 / Arc A750, PCI ID `0x56A1`)** on **macOS Tahoe**.

> **Status:** Experimental / Work in Progress. Patches are split into incremental milestones for controlled hardware testing and safe console fallback.

---

## Repository Contents

| File | Description | Target Base |
| :--- | :--- | :--- |
| `milestone1-dg2-0x56a1.patch` | **Milestone 1:** `IntelXe.kext` DG2 PCI matching, GuC `70.53.0`, PAT/MOCS, LRC offsets, and Xe2 register guards. | `yusufklncc/intel-gpu-macos-driver@main` |
| `milestone2-dg2-fb.patch` | **Milestone 2:** `IntelXeFB.kext` Display 13 (`Xe_HPD`) UEFI GOP adoption, DG2 AUX/EDID probe, Small-BAR guards, and Display 14+ / C20 isolation. | `main` + `milestone1-dg2-0x56a1.patch` |
| `dg2-a750-m1-m2.patch` | Combined single patch containing both **Milestone 1 + Milestone 2**. | `yusufklncc/intel-gpu-macos-driver@main` |

---

## Milestones

### Milestone 1 — Core GPU & GuC Bring-Up (`IntelXe.kext`)
- **PCI Matching & Gating:** Adds `IntelXe-DG2` personality (`0x56a18086` and sibling DG2 IDs) gated behind `-intelxe_allow_dg2`.
- **Firmware Bundling:** Packages DG2 GuC `70.53.0` (`dg2_guc_70.bin`) via `kernel/IntelXe/build.py --platform dg2`.
- **Memory & Page Tables:** Implements pre-Xe2 PAT (`xelp_pat_table`) and DG2 MOCS (`XEHP_LNCFCMOCS0`).
- **Engine Contexts:** Adds DG2 LRC register offsets (`dg2_rcs_offsets`, `dg2_xcs_offsets`, `gen12_xcs_offsets`).
- **Xe2 Guarding:** Guards Xe2-only MMIO registers, forcewake acks, and SLPC frequency reads behind `graphicsVerx100` checks.

### Milestone 2 — Initial DisplayPort Output via GOP Reuse (`IntelXeFB.kext`)
- **Tester Personality & Gating:** Emits `IntelXeFB-DG2` (`IOPropertyMatch: IntelXeReady = True`) in `build.py`, gated separately behind `-intelxe_allow_dg2_fb`.
- **UEFI GOP Pipe & Port Auto-Detection:** Adopts the active UEFI GOP pipe, linear `XRGB8888` scanout surface, and `TRANS_DDI_FUNC_CTL` (`30:27`) port selection for `Xe_HPD` (Display 13).
- **DG2 AUX Channel & EDID Probe:** Maps Display 13 AUX channels (`DDI A` `0x64010`, `DDI B` `0x64110`, `DDI C` `0x64210`, `USBC1/TC1` `0x64310`, `DDI D` `0x64710`) using `ICL_PWR_WELL_CTL_AUX1`/`2` (`0x45440`/`0x45444`), without Display 14+ `AUX_POWER_REQUEST` (`bit 19`). Falls back to the GOP mode if EDID read fails.
- **Static Power Wells (DMC Deferred):** Defers `dg2_dmc_ver2_08.bin` loading and clears `DC_STATE_EN` (`0x45504`) to `0` while keeping UEFI GOP power wells active.
- **Small-BAR & Safe Abort Guarantees:** Validates BAR0 responsiveness (`rd(0) != 0xFFFFFFFF`), `IntelXe` BAR2 mapping (`intelxeGetDeviceInfo`), GOP surface bounds, and write-combined `fSurfMap` **before** mutating `DC_STATE_EN`, NVRAM boot-guard, GGTT reservations, cursor, `IOFramebuffer::start()`, or PCI `class-code`. Any failure cleanly aborts `IntelXeFB::start()` so macOS preserves the UEFI GOP console.
- **Display 14+ / C20 PHY Isolation:** Guards Battlemage-only registers (`PMDEMAND0`/`1`, `DCPR_STATUS_1`, `MTL_MEM_SS_INFO_*`, `MTL_LATENCY_*`, `TRANS_SET_CONTEXT_LATENCY`, `CHICKEN_MISC_2`, `DBUF_CTL_S2`/`S3`, `SHOTPLUG_CTL_*`, `SHPD_FILTER_CNT`, `SOUTH_CHICKEN1`, `TRANS_VRR_*`, `IntelXeLink` C20 MSGBUS, VRR, HPD, and Pipe B) behind `fDisplayVer >= 14`.
- **Pre-Xe2 Flush & VRAM Guards:** Uses `deviceWmb()` instead of Xe2-only `0xb404` in `IntelXe::l2Flush()` and checks `vramMap()` in `xeDispVramAlloc()`.

### Milestone 3 — Planned
- Synopsys (SNPS) PHY PLL & TX-swing programming (`intel_snps_phy`) for dynamic DisplayPort link training and resolution/refresh-rate modesetting.
- `PCH_DG2` hotplug detection (`SDEISR`), multi-monitor (Pipe B), and `dg2_dmc_ver2_08.bin` DMC firmware loading.

---

## Boot Arguments

| Boot Argument | Target Kext | Purpose |
| :--- | :--- | :--- |
| `-intelxe_allow_dg2` | `IntelXe.kext` | Enables `IntelXe` matching and GuC bring-up on DG2 (`0x56A1`). |
| `-intelxe_allow_dg2_fb` | `IntelXeFB.kext` | Enables `IntelXeFB` Display 13 GOP adoption on DG2 (`0x56A1`). |
| `-intelxe_nofb` / `-intelxefb_off` | `IntelXeFB.kext` | Disables `IntelXeFB` while keeping `IntelXe` active (keeps UEFI GOP console). |

---

## Requirements & Hardware Notes

- **Target Hardware:** Intel Arc A750 (`8086:56A1`, DG2-G10).
- **OS:** macOS Tahoe.
- **Display Connection:** A DisplayPort monitor must be connected and lit by UEFI GOP at boot for Milestone 2 (`IntelXeFB` adopts the active GOP surface rather than performing cold SNPS PHY bring-up).
- **Resizable BAR:** Full BAR2 visibility is required for `IntelXeFB`. With Small-BAR / `ResizeAppleGpuBars=-1` (where the GOP surface or `IntelXe` BAR2 window is not fully mappable), `IntelXeFB` aborts startup cleanly to avoid black-screening the UEFI GOP console.

---

## Applying & Building from Upstream

```bash
# 1. Clone upstream driver repository
git clone https://github.com/yusufklncc/intel-gpu-macos-driver.git
cd intel-gpu-macos-driver

# 2a. Apply the combined Milestone 1 + 2 patch:
git apply /path/to/dg2-a750-m1-m2.patch

# OR 2b. Apply incremental milestone patches in order:
# git apply /path/to/milestone1-dg2-0x56a1.patch
# git apply /path/to/milestone2-dg2-fb.patch

# 3. Verify device parity checks
python3 tools/parity/devices.py

# 4. Build kexts on macOS (requires Xcode / MacKernelSDK + DG2 GuC firmware)
python3 kernel/IntelXe/build.py --platform dg2
```

---

## Credits

- Upstream driver: [`yusufklncc/intel-gpu-macos-driver`](https://github.com/yusufklncc/intel-gpu-macos-driver)
