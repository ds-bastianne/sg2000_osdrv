# SG2000 (CV181x) display and camera on mainline Linux: requirements for a clean-room implementation

Version 1.0, 2026-10-06. Companion to `docs/sg2000-kernel-port-plan.md` (the feasibility assessment); this
document is self-contained so that a new engineering session can implement from it without that context.

Conventions: requirement IDs are `REQ-<area>-<n>`. "MUST", "SHOULD", "MAY" have their usual meaning.
Every hardware fact carries a source tag: **[TRM]** public SG2000 Technical Reference Manual (BSD-2-Clause),
**[MAINLINE]** Linux 7.3-rc6 sources, **[DTS]** the vendor device tree as shipped in community images,
**[VENDOR]** behaviour observed by reading the vendor GPL kernel modules (see section 2 for how such facts may be
used), **[ECO]** public ecosystem sources (Armbian, Sophgo wiki, Sophgo repositories). Effort figures are
engineer-week ranges with stated assumptions. "Unverified" marks statements no source confirmed.

---

## 1. Goal, scope and non-goals

### 1.1 Goal

Deliver mainline-style Linux drivers for the SG2000/SG2002 (CVITEK CV181x family) multimedia blocks so that
display and camera work on a current mainline-based kernel (7.x) with standard userspace APIs, written
independently of the vendor driver code, and suitable for upstream submission.

### 1.2 In scope (ordered by priority)

1. **Display**: a DRM/KMS driver for the display engine (timing generator, DRAM scan-out, overlay planes), a MIPI
   DSI host driver, a DSI D-PHY driver, panel/bridge integration via device tree, fbdev emulation.
2. **Shared infrastructure**: a provider for the VIP_SYS sub-controller (per-block resets, clock gates), a shared
   register-page mechanism for the SC_TOP page, CMA/dma-buf memory model, device-tree bindings and an SoC-level
   `sg2000-vip.dtsi`.
3. **Camera front end**: a V4L2 MIPI CSI-2 receiver sub-device, a CSI D-PHY driver, a VI raw/YUV capture driver
   usable with libcamera's generic pipeline handler and software ISP.
4. **Camera processing (conditional, see 2.4)**: a V4L2 ISP driver with params/statistics queues, a VPSS scaler
   driver, a libcamera pipeline handler with a newly written IPA (AE/AWB/AF).

### 1.3 Non-goals

- Compatibility with the vendor middleware (`cvi_mpi`, `/dev/cvi-*` ioctls, VB pools, physical-address ABI).
- Video codec (Chips&Media CODA980 H.264, WAVE420L H.265, CODAJ12-class JPEG): no mainline driver supports these
  cores; separate project (30-55 engineer-weeks estimated).
- TPU: a mainline-compatible driver already exists in the Armbian `sophgo-sg200x` patch set [ECO].
- IVE (vision accelerator), RGN (software overlay manager), DWA/LDC dewarp (may follow later as a V4L2
  mem2mem driver), FreeRTOS fast-boot camera (`fast_image`, `rtos_cmdqu`), I80/8080 LCD interface (vendor
  implementation incomplete), sub-LVDS/HiSPi/BT-demux camera inputs (no V4L2 bus model).
- `pwm`, `rtc`, `wdt`, `saradc`, thermal cooling: mainline drivers exist or are carried by Armbian.

---

## 2. Licensing posture and source rules

### 2.1 Facts

| Source | Licence status | Evidence |
|---|---|---|
| SG2000 TRM sources (`github.com/sophgo/sophgo-doc`, `SG200X/TRM`) | BSD-2-Clause (`LICENSE` in the repository root, Copyright 2023 Sophgo Technologies) | file read |
| Mainline Linux | GPL-2.0 | `COPYING` |
| libcamera | LGPL-2.1+ (library), documentation CC-BY-SA-4.0 | `COPYING.rst` |
| Armbian `sophgo-sg200x` kernel patches | GPL-2.0 (kernel patches) | patch headers |
| Vendor `osdrv` modules (`interdrv/v2`) | Modules declare `MODULE_LICENSE("GPL")` (26 sites); only 6 files carry an SPDX `GPL-2.0` tag (`fast_image/*`, `rtos_cmdqu/cvi_spinlock.*`); the register headers carry "Copyright (C) Cvitek Co., Ltd. ... All rights reserved." without SPDX; no top-level `LICENSE`/`COPYING` | repository read |
| Vendor middleware `cvi_mpi` (`github.com/sophgo/cvi_mpi`, branch `sg200x-dev`) | No licence file (only `3rdparty/inih/LICENSE.txt`); all ISP sources "All rights reserved"; AE/AWB/AF cores and `isp_algo` shipped as prebuilt objects only (56 riscv64 objects, 0 C files under `modules/isp/algo/{ae,awb,af}`) | repository read |
| Sophgo `linux-common` 6.12.y `drivers/gpu/drm/cvitek` (CV186x DRM driver) | Inside a GPL-2.0 kernel tree, but the files carry no SPDX tag (0 of 15) | repository read |
| Vendor codec firmware (`coda980.bin`, `monet.bin`) | Redistribution terms unknown | not found |

### 2.2 Source rules for the implementation team

- **REQ-LIC-1** Implementers MUST work only from: this document, the SG2000 TRM, mainline Linux sources,
  libcamera sources, the Armbian patch set, public datasheets of panels/bridges/sensors, and bench measurements.
- **REQ-LIC-2** Implementers MUST NOT read, copy, translate or paraphrase code from the vendor `osdrv`
  repository, the `cvi_mpi` middleware, the `sophgo/isp` repository, or the Sophgo `linux-common` DRM driver,
  unless Legal explicitly approves a documented clean-room protocol (separate specification team, written
  functional specification, no code transfer, source log).
- **REQ-LIC-3** The vendor prebuilt 3A objects, ISP tuning files and codec firmware MUST NOT be linked, shipped
  or used as oracles without Legal approval.
- **REQ-LIC-4** All new kernel code MUST carry `SPDX-License-Identifier: GPL-2.0` (or `GPL-2.0 OR MIT` for device
  trees and bindings, following `arch/riscv/boot/dts/sophgo/`), new libcamera code LGPL-2.1+.
- **REQ-LIC-5** Facts in this document tagged [VENDOR] were obtained by reading the vendor GPL driver sources
  during the assessment. They are structural facts (addresses, counts, formats, behaviours), not code. Before the
  implementation starts, Legal MUST confirm that using these facts under the clean-room protocol is acceptable;
  if not, the affected requirements (mainly sections 7.3 and 7.4) are blocked until documentation is obtained
  from Sophgo.

### 2.3 Legal items to escalate before implementation (Dentsply Sirona Legal/Privacy/Compliance)

1. Acceptability of the clean-room protocol and of the [VENDOR] facts in this document.
2. Whether a GPL in-kernel driver may be derived from the vendor `osdrv` headers given their "All rights
   reserved" text (only relevant if REQ-LIC-2 is relaxed).
3. Redistribution of any Sophgo binaries (3A objects, firmware) in product images (expected answer: no).
4. Copyright and attribution handling for upstream submissions.

I cannot confirm the legal sufficiency of any of the above; they are decisions for Legal.

### 2.4 Documentation coverage and its consequence

| Block | Public register documentation | Consequence |
|---|---|---|
| Display engine (VDP DISP, OSD/GOP), MIPI DSI TX controller and PHY | **Yes** [TRM] `video/vdp.rst`, `video/mipi_tx.rst` with register tables | Clean-room implementation possible from TRM + this document |
| MIPI CSI-2 RX (D-PHY, CSI controller, sub-LVDS), VI top (capture front end) | **Yes** [TRM] `video/mipi_rx.rst`, `video/vi.rst` | Clean-room possible |
| ISP pipeline (pre-raw FE/BE, RAW/RGB/YUV top), VPSS scaler, DWA, IVE, codec, TPU, VIP_SYS sub-controller, SC_TOP interrupt register | **No** (the TRM only mentions ISP/VPSS in feature lists) | Not implementable clean-room from public sources; requires Sophgo documentation or a Legal-approved specification process. Section 7.3/7.4 requirements are therefore **conditional** |

---

## 3. Target platform facts

### 3.1 SoC and boards

- SG2000 (marketing name of CV1813H) and SG2002 (CV1812CP): RISC-V T-Head C906 64-bit main core (optionally an
  Arm Cortex-A53 in the same package), a C906L little core (FreeRTOS), an 8051 MCU [TRM system-overview, ECO].
- DMA is **not cache-coherent**; the SoC node carries `dma-noncoherent` in mainline and cache maintenance is
  done through the T-Head CMO errata path [MAINLINE `arch/riscv/boot/dts/sophgo/sg2000.dtsi:21`,
  `arch/riscv/errata/thead/errata.c:74-125`].
- DRAM at 0x80000000: 512 MiB on the Milk-V Duo S (SG2000), 256 MiB on SG2002 boards (Milk-V Duo 256M,
  Sipeed LicheeRV Nano) [MAINLINE `sg2000.dtsi:14-17`, `sg2002.dtsi:17-19`].
- Boards with mainline device trees: Milk-V Duo S (`sg2000-milkv-duo-s.dts`), Duo 256M, LicheeRV Nano B,
  Duo Module 01 EVB (arm64). On the Duo S the MIPI TX pads are muxed to `sdhci1` (SDIO WiFi); DSI output on that
  board needs a DTS variant that frees those pads [MAINLINE `sg2000-milkv-duo-s.dts:116-133`]. The LicheeRV Nano
  (SG2002) has a DSI connector; VIP identity between SG2002 and SG2000 is assumed, unverified. The Duo Module 01
  EVB is reported to carry an LT8912B-class DSI-to-HDMI bridge [ECO BadgeOS, search snippet only; unverified].
- Assumption: Linux runs on the RISC-V C906 (riscv64). All coherency, errata, Kconfig and toolchain requirements
  in this document are for that core; the Arm Cortex-A53 option was not assessed (the Duo Module 01 EVB mainline
  device tree is arm64-only [ECO]).
- Vendor board wiring [DTS]: Duo S panel GPIOs reset `porte 2` (active low), backlight enable `porte 0`,
  panel power `porte 1`; sensor reset `porta 2` (active low); touch controller on `i2c4`. LicheeRV Nano: panel
  reset `porte 0`, backlight `porte 2`, sensor reset `porte 1`. The vendor DT describes **no** panel, sensor, CSI/DSI
  lane topology or pinmux; these come from board schematics.

### 3.2 Mainline 7.3 baseline the implementation builds on

| Provided | Mainline location |
|---|---|
| Clock controller `sophgo,sg2000-clk` at 0x03002000 with the full VIP/VC clock tree | `drivers/clk/sophgo/clk-cv1800.c`, `include/dt-bindings/clock/sophgo,cv1800.h` |
| Reset controller `sophgo,cv1800b-reset` at 0x03003000 (reset-simple, 4 KiB = 32768 indices; macros in `arch/riscv/boot/dts/sophgo/cv18xx-reset.h`) | `drivers/reset/reset-simple.c:154` |
| Pinctrl `sophgo,sg2000-pinctrl` (0x03001000 and 0x05027000) with all MIPI TX/RX and VIVO pads; `PINMUX(pin, func)` binding | `drivers/pinctrl/sophgo/pinctrl-sg2000.c`, `include/dt-bindings/pinctrl/pinctrl-sg2000.h` |
| Top syscon `sophgo,cv1800b-top-syscon` at 0x03000000; RTC syscon at 0x05025000 | `cv180x.dtsi:28-31,469-471` |
| DMA mux, AXI DMA, DesignWare I2C/SPI/UART/GPIO, SDHCI, dwmac, USB2 PHY + dwc2, RTC, SARADC, mailbox (`cv1800-mailbox.c`, no DTS node yet), I2S/codecs | `cv180x.dtsi`, `drivers/mailbox/cv1800-mailbox.c`, `sound/soc/sophgo/` |
| dma-buf heaps (`system`, `cma`; one `/dev/dma_heap/<name>` per `shared-dma-pool` + `reusable` reserved-memory node); no carveout heap; no in-kernel heap allocation API | `drivers/dma-buf/heaps/cma_heap.c:397-421`, `include/linux/dma-heap.h` |
| Generic V4L2 ISP params/stats framing | `include/uapi/linux/media/v4l2-isp.h`, `Documentation/driver-api/media/v4l2-isp.rst` |
| DRM helpers: `drm_gem_dma`, `drm_fbdev_dma`, `drm_mipi_dsi` host API, 106 DSI panel drivers, bridges incl. `lontium-lt8912b.c` and LT9611 | `drivers/gpu/drm/` |
| Sensor drivers present: imx219, imx290, imx334, imx335, imx415, imx678, ov5647, ov5640, ov4689, ov9282, gc2145, gc05a2, gc08a3, os05b10. **Absent**: gc2053, gc2083, gc4653, SmartSens sc*, os04a10 | `drivers/media/i2c/` |

Nothing exists in mainline for VI/ISP, CSI-2 RX, VPSS, display, DSI, D-PHY (RX or TX), VIP_SYS, codec, JPEG,
TPU, IVE, DWA; no reserved-memory/CMA nodes; no `Documentation/devicetree/bindings/{media,display}` entries for
Sophgo.

### 3.3 Kernel configuration requirements

- **REQ-CFG-1** `CONFIG_ERRATA_THEAD=y` (hence `ERRATA_THEAD_CMO`, `RISCV_DMA_NONCOHERENT`, `DMA_DIRECT_REMAP`).
  It has no default and is **not** set in `arch/riscv/configs/defconfig`; without it every DMA engine sees stale
  cache data [MAINLINE `arch/riscv/Kconfig.errata:99-126`].
- **REQ-CFG-2** `CONFIG_CMA=y`, `CONFIG_DMA_CMA=y`, `CONFIG_DMABUF_HEAPS=y`, `CONFIG_DMABUF_HEAPS_CMA=y`,
  `CONFIG_CMA_AREAS` >= number of reserved pools.
- **REQ-CFG-3** `CONFIG_DRM`, `CONFIG_DRM_MIPI_DSI`, `CONFIG_DRM_PANEL_*` as needed, `CONFIG_DRM_LONTIUM_LT8912B`
  or LT9611, `CONFIG_DRM_FBDEV_EMULATION`; `CONFIG_MEDIA_SUPPORT`, `CONFIG_V4L_PLATFORM_DRIVERS`,
  `CONFIG_VIDEO_V4L2_SUBDEV_API`, `CONFIG_MEDIA_CONTROLLER`, `CONFIG_VIDEOBUF2_DMA_CONTIG`.
- **REQ-CFG-4** The design MUST NOT depend on `/dev/mem` (`CONFIG_STRICT_DEVMEM=y` and kernel lockdown MUST be
  compatible). Note: on riscv `STRICT_DEVMEM` is selectable (`GENERIC_LIB_DEVMEM_IS_ALLOWED`, `arch/riscv/Kconfig:125`)
  and defaults off; the Armbian configs leave it off.
- **REQ-CFG-5** Recommended integration base: the Armbian `sophgo-sg200x` kernel family (mainline 7.2 "edge",
  7.3 "bleedingedge") whose patches already add thermal, timer, watchdog, PWM, C906L remoteproc + mailbox DTS,
  eFuse nvmem, I2S, TPU and a 128 MiB default CMA pool [ECO `armbian/build`, `patch/kernel/archive/sophgo-sg200x-7.3/`].

### 3.4 Interrupt numbering

Vendor documentation and DTS use PLIC hardware IRQ numbers; the mainline SG2000 DTS uses
`SOC_PERIPHERAL_IRQ(n) = n + 16` [MAINLINE `sg2000.dtsi:3`]. Vendor hwirq N equals `SOC_PERIPHERAL_IRQ(N - 16)`.

---

## 4. VIP address map and resources

All addresses are physical. Clock names are the vendor consumer names (useful for `clock-names`); IDs are the
mainline `sophgo,cv1800.h` values (every vendor clock has a same-named mainline clock; numeric IDs differ from the
vendor header, translate by name). Reset numbers refer to the mainline `sophgo,cv1800b-reset` controller.

| Block | Base / size | IRQ (vendor hwirq -> mainline) | Clocks (name -> mainline ID) | Resets | Doc |
|---|---|---|---|---|---|
| ISP (vendor "vi") | 0x0A000000 / 0x80000 (143 register sub-blocks; CMDQ at +0x7FC00) | 24 -> `SOC_PERIPHERAL_IRQ(8)`, "isp" | clk_sys_0..3 -> `CLK_SRC_VIP_SYS_0..3` (100-103), clk_axi -> `CLK_AXI_VIP` (99), clk_csi_be -> `CLK_CSI_BE_VIP` (105), clk_raw -> `CLK_RAW_VIP` (134), clk_isp_top -> `CLK_ISP_TOP_VIP` (111), clk_csi_mac0/1/2 -> `CLK_CSI_MAC0/1/2_VIP` (106-108) | isp_top, csi_be via VIP_SYS; `RST_VIPSYS` 6 (whole domain) | [DTS][VENDOR]; not in TRM |
| SCL_TOP page (scaler top, VO mux, BT/LVDS encoders, interrupt mask/status/enable) | 0x0A080000 / 0x1000 | 25 -> `SOC_PERIPHERAL_IRQ(9)`, "sc" (shared by scalers and display vblank) | clk_sc_top -> `CLK_SC_TOP_VIP` (114) | sc_top via VIP_SYS | [VENDOR]; not in TRM |
| IMG_IN x2 (read DMA for display path "D" and video path "V") | 0x0A082000, 0x0A083000 / 0x1000 each | (sc) | clk_img_d/v -> `CLK_IMG_D_VIP` (112) / `CLK_IMG_V_VIP` (113) | img_d, img_v via VIP_SYS | [VENDOR] |
| Scaler cores SC_D, SC_V1, SC_V2, SC_V3 (+GOP per core) | 0x0A084000 + 0x1000 x n / 0x1000 each | (sc) | clk_sc_d -> 115, clk_sc_v1/2/3 -> 116/117/118 | sc_d, sc_v1..3 via VIP_SYS | [VENDOR] |
| VDP DISP (display timing generator, scan-out DMA, CSC, gamma, covers) | 0x0A088000 / 0x400 (TRM); vendor treats 0x0A088000-0x0A088FFF | (sc): `disp_frame_start`, `disp_frame_end` bits | clk_disp -> `CLK_DISP_VIP` (121), `CLK_DISP_SRC_VIP` (161), `CLK_DISPPLL` (5) | disp via VIP_SYS | **[TRM]** `video/vdp.rst` |
| VDP OSD (display GOP overlay) | 0x0A088800 / 0x200 | (sc) | as DISP | | **[TRM]** |
| BT.601/656/1120 encoder registers | 0x0A089000 / 0x1000 | | clk_bt -> `CLK_BT_VIP` (120) | bt via VIP_SYS | [VENDOR]; BT formats described in TRM vdp.rst |
| MIPI DSI TX controller ("DSI MAC") | 0x0A08A000 / 0x1000 | | clk_dsi -> `CLK_DSI_MAC_VIP` (122), `CLK_DSI_ESC` (98) | dsi_mac via VIP_SYS | **[TRM]** `video/mipi_tx.rst` |
| CMDQ (scaler command queue) | 0x0A08B000 / 0x1000 | | | | [VENDOR] |
| IVE | 0x0A0A0000 / 0x3100 | 97 -> `SOC_PERIPHERAL_IRQ(81)` | `CLK_IVE_VIP` (133) | ive_top via VIP_SYS | [DTS]; out of scope |
| DWA / LDC (dewarp) | 0x0A0C0000 / 0x1000 | 28 -> `SOC_PERIPHERAL_IRQ(12)`, "dwa" | clk_sys_0..4 -> 100-104, clk_dwa -> `CLK_DWA_VIP` (119) | ldc via VIP_SYS | [DTS]; out of scope |
| VI top / CSI MAC0 (incl. CSI controller +0x400, sub-LVDS +0x200, output-to-ISP +0x600) | 0x0A0C2000 / 0x2000 | 26 -> `SOC_PERIPHERAL_IRQ(10)`, "csi0" (error counters only) | clk_cam0 -> `CLK_CAM0` (147), clk_cam1 -> `CLK_CAM1` (148), clk_sys_2 -> 102, `CLK_MIPIMPLL` (3), `CLK_DISPPLL` (5), `CLK_FPLL` (2); MAC gate `CLK_CSI_MAC0_VIP` (106); `CLK_CSI0_RX_VIP` (109) exists, role unverified | csi_mac0 via VIP_SYS; `RST_VIP_CAM0` 99 | **[TRM]** `video/vi.rst` (VI top), `video/mipi_rx.rst` (CSI 0x0A0C2400, sub-LVDS 0x0A0C2200) |
| VI top / CSI MAC1 | 0x0A0C4000 / 0x2000 | 27 -> `SOC_PERIPHERAL_IRQ(11)`, "csi1" | `CLK_CSI_MAC1_VIP` (107); `CLK_CSI1_RX_VIP` (110) | csi_mac1 via VIP_SYS | **[TRM]** (CSI 0x0A0C4400, sub-LVDS 0x0A0C4200) |
| VI top MAC2 (parallel/BT inputs only) | 0x0A0C6000 / 0x2000 | none | `CLK_CSI_MAC2_VIP` (108) | csi_mac2 via VIP_SYS | **[TRM]** mentions a BT-only instance |
| VIP_SYS sub-controller | 0x0A0C8000 / use 0x100 (vendor DTS declares 0x20 but registers up to 0xE0 are used) | none | | provides all VIP-internal resets | [DTS][VENDOR]; **not in TRM** |
| MIPI CSI RX D-PHY ("CSI wrap") | 0x0A0D0000 / 0x1000 (PHY top +0x000, 4-lane PHY +0x300, 2-lane PHY +0x600) | | clocks as MAC0 | CSIPHY0 70, CSIPHY0_APB 71, CSIPHY1 72, CSIPHY1_APB 73 (raw indices; no macros) | **[TRM]** `video/mipi_rx.rst` |
| MIPI DSI TX D-PHY | 0x0A0D1000 / 0x100 (offsets up to 0xC8 used) | | `CLK_DISPPLL`, `CLK_MIPIMPLL` (PLL parents, unverified) | DSIPHY 68, DSIPHY_APB 69 (raw) | **[TRM]** `video/mipi_tx.rst` |
| Pad control used by vendor CSI driver | 0x03001C30 / 0x30 (inside the pinctrl block; never actually used by the vendor code) | | | | use pinctrl bias properties instead |
| Video codec H.265 / H.264 / vc_ctrl / vc_sbm / vc_addr_remap | 0x0B020000/0x10000, 0x0B010000/0x10000, 0x0B030000/0x100, 0x0B058000/0x100, 0x0B050000/0x400 | 22, 21, 23 | `CLK_AXI_VIDEO_CODEC` 137, `CLK_H264C` 141, `CLK_APB_H264C` 142, `CLK_H265C` 143, `CLK_APB_H265C` 144, `CLK_VC_SRC0/1/2` 138-140, `CLK_CFG_REG_VC` 154 | `RST_H264C` 3, `RST_H265C` 5, `RST_VCSYS` 95 | [DTS]; out of scope |
| JPEG | 0x0B000000 / 0x300 | 20 | `CLK_JPEG` 145, `CLK_APB_JPEG` 146 | `RST_JPEG` 4 | [DTS]; out of scope |
| TPU (tdma, tiu) | 0x0C100000, 0x0C101000 / 0x1000 each | 75, 76 | `CLK_TPU` 11, `CLK_TPU_FAB` 12 | `RST_TDMA` 7, `RST_TPU` 8, `RST_TPUSYS` 9 | [DTS]; Armbian driver exists |
| Mailbox to C906L | 0x01900000 / 0x1000 (hardware spinlock at +0xC0 used by the vendor protocol) | 101 | | | [DTS][MAINLINE `cv1800-mailbox.c`] |
| SoC blocks poked directly by vendor code and owned by mainline drivers | clock controller 0x03002000 (incl. 0x03002840 DISPPLL synthesizer control, 0x3002008), pinctrl FMUX 0x03001000, top misc 0x03000220, RTC 0x03005144, DDR controller 0x08004544 | | | | new code MUST use the clk/pinctrl/syscon frameworks instead (REQ-CLK-1, REQ-PIN-1) |

Vendor reserved-memory layout for reference [DTS]: Duo S Linux memory 0x80000000 size 0x1FE00000; C906L firmware
2 MiB at the top (`no-map`); remoteproc vrings/buffer 0x8F528000-0x8F630000 (`no-map`); ION carveout 74 MiB
(dynamically placed); framebuffer 5632 KiB at 0x9AE80000. LicheeRV Nano: ION 63 MiB, same layout scaled.

---

## 5. Shared infrastructure requirements

### 5.1 VIP_SYS provider

Facts [VENDOR][DTS]: register offsets 0x00 VIP_RESETS (per-block soft resets: isp_top, img_d, img_v, sc_top, sc_d,
sc_v1, sc_v2, sc_v3, disp, bt, dsi_mac, csi_mac0, csi_mac1, csi_mac2, ldc, clk_div, their APB variants, dsi_phy,
csi_phy0, csi_be), 0x08 interrupt status, 0x10 AXI real-time/offline switch, 0x14 CLK_LP (low-power gates),
0x18 CLK_CTRL0 (clock selects: DSI/BT output divider bit, CSI0/CSI1 RX source select, VI clock source select),
0x1C CLK_CTRL1 (incl. a DWA bit 20), 0x30-0x48 per-MAC "normal clock ratio" dividers used for CSI MAC clocks
(targets 198/297/396/500/594 MHz), 0x70 AXI fabric priority, 0xC0 RESETS1 (incl. ive_top). Bit positions are not
publicly documented and MUST be determined per the clean-room protocol or from Sophgo documentation.

- **REQ-VIPSYS-1** Provide a device-tree node `syscon@a0c8000` (`"syscon", "simple-mfd"`, size 0x100) with a
  reset-controller child (`#reset-cells = <1>`, reset-simple style over offsets 0x00 and 0xC0) and a clock
  child (`#clock-cells = <1>`) exposing the gates in CLK_LP/CLK_CTRL* and the CSI MAC dividers.
- **REQ-VIPSYS-2** All writes to CLK_CTRL0 MUST be bit-field updates (`regmap_update_bits`); the display driver
  MUST NOT alter the CSI/VI select bits and vice versa.
- **REQ-VIPSYS-3** New YAML bindings under `Documentation/devicetree/bindings/{reset,clock,soc}/sophgo,sg2000-vip-*.yaml`,
  `dtbs_check` clean.
- **REQ-VIPSYS-4** Reset pulse semantics: assert, wait >= 20 us, deassert (vendor behaviour [VENDOR]); verify on bench.

### 5.2 SC_TOP shared page and shared interrupt

Facts [VENDOR]: the 4 KiB page at 0x0A080000 holds CFG0/CFG1 (+0x00/+0x04: scaler enables **and** display enable
and "display fed by scaler" bits in one word), BT_CFG (+0x0C), interrupt MASK/STATUS/ENABLE (+0x30/+0x34/+0x38;
status bits 0-1 display frame start/end, 2-18 image-in and scaler events, 19 I80 frame end, 20 timing-generator
lite), LVDS TX (+0x50), BT encoder and sync codes (+0x60/+0x64), VO output type mux (+0x70: disable, RGB, SW, I80,
BT601, BT656, BT1120, BT1120R, serial RGB, HW-MCU), VO pad mux (+0x90..+0xAC, 28 signals onto MIPI TX/RX and VIVO
pads). One interrupt line (vendor 25) serves scalers and display. The vendor driver writes the status word back to
clear it, which is consistent with write-1-to-clear per bit but **unverified** (TRM silent).

- **REQ-SCTOP-1** Model the page as a syscon regmap shared by the display driver and any future scaler driver; no
  driver may `ioremap` it privately.
- **REQ-SCTOP-2** Interrupt ownership: bench-verify per-bit write-1-to-clear on `INTR_STATUS`. If confirmed, both
  drivers MAY request the line `IRQF_SHARED`, each clearing only its own bits and returning `IRQ_NONE` otherwise.
  If not confirmed, implement a small interrupt demultiplexer (irqchip) owning +0x30..+0x38. Until a scaler driver
  exists, the display driver MAY own the page and the line alone.
- **REQ-SCTOP-3** The mask/enable words are shared; updates MUST be read-modify-write under the regmap lock.

### 5.3 Memory model

- **REQ-MEM-1** All frame buffers are dma-buf objects: DRM GEM-DMA for scan-out, videobuf2 dma-contig for capture,
  `/dev/dma_heap/<name>` (CMA heap) for userspace-allocated buffers. No physical address crosses the uAPI.
- **REQ-MEM-2** Device tree declares at least one reserved-memory node `compatible = "shared-dma-pool"; reusable;`
  sized per product (vendor used 74 MiB on Duo S and 63 MiB on LicheeRV Nano for all multimedia buffers; Armbian
  uses a 128 MiB default pool), referenced by `memory-region` from the DRM, ISP and scaler nodes; the kernel
  exposes it automatically as a dma-heap.
- **REQ-MEM-3** A C906L firmware region (2 MiB, `no-map`) and remoteproc vring regions are only declared if the
  little core is used; a region shared with that core cannot be `reusable`.
- **REQ-MEM-4** Buffer alignment: display scan-out stride multiple of 64 bytes, GOP overlay stride multiple of
  16 bytes [VENDOR]; capture strides per TRM VI storage formats (`video/vi.rst`).

### 5.4 Coherency

- **REQ-DMA-1** Use the DMA API only (`dma_alloc_*`, `dma_map_*`, `dma_sync_*`, dma-buf `begin/end_cpu_access`);
  never call `arch_sync_dma_*` (not exported) or issue cache instructions directly.
- **REQ-DMA-2** Devices performing DMA MUST be children of the `soc` node (inherit `dma-noncoherent`).
- **REQ-DMA-3** Acceptance: a CPU-written pattern copied by a hardware engine (scaler or display read-back path)
  and read back must be bit-exact over 10,000 iterations on both cached and write-combined mappings.

### 5.5 Clocks, resets, pinmux

- **REQ-CLK-1** All clock control through the common clock framework using the mainline IDs of section 4. No
  `ioremap` of 0x03002000-0x03002FFF. If a bit the hardware needs is not modelled (candidates: 0x03002840 bit 1
  observed written by the vendor display path; CAM0/CAM1 PLL power-down and divider control for sensor MCLK), extend
  `clk-cv1800.c` with a reviewed patch rather than poking.
- **REQ-CLK-2** Sensor MCLK via `CLK_CAM0`/`CLK_CAM1` (`clk_set_rate`; required rates 24, 25, 26, 27 and
  37.125 MHz [VENDOR]); verify achievable rates on a scope.
- **REQ-CLK-3** `CLK_SRC_VIP_SYS_1` is run at 300 MHz by the vendor configuration [DTS `clock-freq-vip-sys1`];
  express with `assigned-clock-rates`.
- **REQ-RST-1** Top-level resets via `&rst` with `cv18xx-reset.h` macros or raw indices 68-73 (propose macros
  `RST_DSIPHY`, `RST_DSIPHY_APB`, `RST_CSIPHY0`, `RST_CSIPHY0_APB`, `RST_CSIPHY1`, `RST_CSIPHY1_APB` upstream).
- **REQ-PIN-1** Pin functions via `pinctrl-0` states using `PINMUX(PIN_x, func)`; no FMUX register writes. Pure
  MIPI DSI/CSI lane operation appears to need no pin function change [VENDOR, unverified]; BT/parallel/TTL modes
  need VO/VI pad functions (function value 2 for VO on MIPI_TX pads in the vendor table, to be confirmed from the
  pinctrl driver's pad tables). MIPI RX pads need pull-up/down disabled (`bias-disable`).

### 5.6 Device tree

- **REQ-DT-1** Add `arch/riscv/boot/dts/sophgo/sg2000-vip.dtsi` (included by `sg2000.dtsi`/`sg2002.dtsi`) with
  nodes for VIP_SYS, SC_TOP syscon, display controller, DSI, DSI D-PHY, CSI D-PHY, CSI receivers, ISP, scaler, DWA
  (disabled by default), reserved memory; the SG2002 variant differs only in memory size.
- **REQ-DT-2** Board files add panel or bridge nodes (with `port` graph to the DSI node), sensor nodes on the
  I2C bus with `port` graph to a CSI receiver, GPIOs as `reset-gpios`/`enable-gpios`, backlight, pinctrl states.
- **REQ-DT-3** All bindings documented in YAML and `dtbs_check` clean; vendor compatibles `cvitek,*` MUST NOT be used.

---

## 6. Display requirements

### 6.1 Hardware facts

Display engine (VDP DISP) [TRM `video/vdp.rst`; VENDOR for behaviour notes]:
- Timing generator with registers for total, vertical/horizontal sync, data-enable (FDE) and active-window (MDE)
  ranges; progressive only (interlaced is rejected by the vendor driver).
- Scan-out DMA with three plane addresses (Y/C or R/G/B), luma and chroma pitch, crop offset/size, destination
  window; shadow registers latched by a force-update bit, giving clean double-buffered updates at frame end.
- Input formats (vendor list): YUV420/422 planar, NV12/NV21/NV16/NV61, YUYV/YVYU/UYVY/VYUY, RGB888/BGR888 packed;
  also planar RGB and HSV (no DRM fourcc, drop). Input CSC (BT.601/BT.709 YUV to RGB) and output CSC (RGB to YUV for
  BT outputs), pattern generator, background colour, 65-node gamma LUT (applied when AXI idle), four solid-colour
  "cover" windows.
- OSD/GOP overlay [TRM vdp.rst; VENDOR]: 2 layers x 8 windows; formats ARGB8888, ARGB4444, ARGB1555, 256-colour
  LUT, 16-colour LUT, font; per-window address/pitch/crop/size/position, colour key, optional 2x horizontal and
  vertical upscale; the LUT cannot be updated while the window is enabled [VENDOR].
- Outputs: MIPI DSI, BT.601/656/1120 parallel (SAV/EAV codes), LVDS (VESA/JEIDA, 6/8/10 bit, dual link, reusing
  the DSI PHY in LVDS mode), serial RGB, I80/8080 MCU (hardware command FIFO; vendor support incomplete).
- Vblank source: display frame start/end bits in the SC_TOP interrupt status (5.2).
- Boot handoff: the vendor bootloader may already drive the panel (boot logo); the vendor driver detects an
  enabled timing generator and reads the configuration back ("smooth" mode) [VENDOR].

MIPI DSI TX controller [TRM `video/mipi_tx.rst`]:
- 1/2/4 data lanes, lane order and P/N polarity configurable; high-speed up to 2500 Mbps per lane; only data
  lane 0 supports low-power transmit/receive and bus turn-around (up to 10 Mbps).
- DSI RGB16/18/24/30 output; video mode burst, non-burst with sync events, non-burst with sync pulses, and
  command mode are supported by the hardware per TRM (the vendor driver uses burst mode only).
- Controller registers [TRM]: mode enable (idle / high-speed / escape / short packet), two high-speed configuration
  words (lane count, pixel format, line width), escape-mode register with a 16-byte low-power transmit FIFO and
  4-byte receive buffer; ECC and CRC16 of long packets are computed by software [VENDOR].

DSI D-PHY [TRM `video/mipi_tx.rst`, 0x0A0D1000; VENDOR]: 5 physical lanes with per-lane logical role select and
P/N swap; lane enable and preamble (needed above 1.5 Gbps); transmit PLL with VCO/divider programming (the vendor
derives it in-driver and also sets a divider bit in VIP_SYS CLK_CTRL0); HS prepare/zero/trail timing registers;
manual low-power override; LVDS/sub-LVDS enable with bias/voltage select.

### 6.2 Requirements

- **REQ-DRM-1** A DRM/KMS driver `drivers/gpu/drm/sophgo/` with one CRTC (timing generator; `drm_display_mode`
  mapped to total/sync/FDE/MDE registers; enable = timing-generator enable; atomic flush = shadow force-update;
  vblank = frame-end interrupt bit), using `drm_gem_dma` and `drm_fbdev_dma` (`/dev/fb0` in XRGB8888).
- **REQ-DRM-2** Primary plane with at least NV12, NV21, NV16, YUYV, UYVY, RGB888, BGR888, XRGB8888; `atomic_check`
  enforces 64-byte stride alignment, maximum size, and window position limits; crop via source offset/size.
- **REQ-DRM-3** Overlay planes from the OSD/GOP windows (minimum 2, target 4) with ARGB8888, ARGB4444, ARGB1555;
  C8 with palette optional (respect the "LUT not updatable while enabled" constraint); fixed zpos; colour key as a
  property is optional.
- **REQ-DRM-4** CRTC `GAMMA_LUT` with 65 nodes (interpolate from the 256/1024-entry DRM LUT), applied at vblank.
- **REQ-DRM-5** DSI encoder implementing `mipi_dsi_host_ops` (`attach`, `detach`, `transfer`); `transfer`
  implements short and long packets with software ECC and CRC16 like `drivers/gpu/drm/sun4i/sun6i_mipi_dsi.c`,
  reads via bus turn-around on lane 0; `attach` accepts video burst mode first; non-burst and command mode MAY be
  added after bench verification. Lanes 1/2/4, formats RGB565/666/888.
- **REQ-DRM-6** D-PHY as a generic PHY driver `drivers/phy/sophgo/phy-sg2000-dsi-dphy.c` implementing
  `phy_configure` with `phy_mipi_dphy` options (lane count, HS rate, lane remap/polarity from the DSI endpoint
  `data-lanes`/`lane-polarities`), computing the PLL from `CLK_DISPPLL`/`CLK_MIPIMPLL` rates and the VIP_SYS divider
  bit through the syscon; LVDS mode optional later.
- **REQ-DRM-7** Panels and bridges via `drm_of_find_panel_or_bridge` from the DSI node's output port; reset,
  enable and backlight belong to the panel/bridge/backlight nodes, never to the DSI or CRTC driver.
- **REQ-DRM-8** Clocks: `CLK_DISP_VIP`, `CLK_DSI_MAC_VIP`, `CLK_DSI_ESC`, `CLK_BT_VIP`, `CLK_SC_TOP_VIP`,
  `CLK_DISPPLL` through CCF; resets disp/dsi_mac/dsi_phy through the VIP_SYS and top reset providers.
- **REQ-DRM-9** Boot handoff: on probe, either read back and adopt an already-running timing generator or perform
  a full reset sequence; MUST not leave the panel in an undefined state.
- **REQ-DRM-10** Suspend/resume restores the mode; shutdown sends DCS display-off (0x28) through the panel driver.
- **REQ-DRM-11** Parallel BT.601/656/1120 and LVDS encoders are a second increment (bridge chain, pinctrl states),
  I80/MCU is excluded.
- **REQ-DRM-12** No `ioremap` of fixed addresses; all register access through the node's `reg`, the SC_TOP syscon
  and the VIP_SYS syscon.

### 6.3 Display acceptance criteria

- `modetest -s` shows a stable test pattern on a DSI panel (hx8394/ili9881c/st7701 class with an existing
  mainline panel driver) or on HDMI through LT8912B/LT9611 at the native mode.
- Vblank counter within 1% of the mode refresh rate; pixel clock within 2% (scope).
- `kmscube` or Weston at 60 Hz for 10 minutes without underflow; `/dev/fb0` console visible.
- igt `kms_plane`/`kms_atomic` basic tests pass for primary and overlay planes; gamma visible.
- `dtbs_check` clean; no fixed-address `ioremap`; suspend/resume restores output.

Effort (assumption: one DRM-experienced engineer, board with free DSI pads, panel with mainline driver or an HDMI
bridge, scope available): minimal driver 8-14 engineer-weeks plus 1-3 for the syscon/IRQ groundwork; first
picture 4-8; overlays, gamma, BT/LVDS 18-30 total; upstream review +3-6 weeks of engineer time (calendar months).

---

## 7. Camera requirements

### 7.1 MIPI CSI-2 receiver and D-PHY (clean-room possible)

Facts [TRM `video/mipi_rx.rst`, `video/vi.rst`; VENDOR for topology]:
- Up to two MIPI receivers: MAC0 fed by a 4-lane PHY, MAC1 by a 2-lane PHY ("dual mode"); 6 physical D-PHY lanes in
  two 3-lane ports (lanes 0-2, 3-5), any physical lane routable to logical CLK/D0-D3 with P/N swap; clock-lane
  buffer direction between the ports and per-lane deskew phase are configurable; HS settle time register.
- CSI-2 decoder: lane count, data types RAW8/10/12 (0x2A/0x2B/0x2C) and YUV422 8/10-bit (0x1E/0x1F), virtual
  channel mapping to up to 4 outputs, HDR modes (virtual-channel, data-type, DOL line-interleaved, manual
  geometry), error interrupts (ECC, CRC, word count, FIFO full, HDR).
- Sub-LVDS and HiSPi deserialisers, parallel/TTL inputs (DVP, BT.601, BT.656 9-bit, BT.1120, BT-demux 2/3/4
  channels) exist; out of scope except plain BT.656/parallel later.
- Output stage: crop window, info-line strip, YUV swap; **no DMA**: pixels go on-chip to the ISP CSI bridge.
- Sensor MCLK generators `CLK_CAM0`/`CLK_CAM1`.

Requirements:
- **REQ-CSI-1** V4L2 sub-device driver `drivers/media/platform/sophgo/sg2000-csi2rx.c`, one instance per MAC
  (MAC0 4-lane, MAC1 2-lane), entity function `MEDIA_ENT_F_VID_IF_BRIDGE`, sink pad from the sensor (fwnode
  endpoint: `data-lanes`, `clock-lanes`, `lane-polarities`, `link-frequencies`), source pad to the VI/ISP driver;
  `V4L2_SUBDEV_FL_STREAMS` routing for virtual channels in a later increment.
- **REQ-CSI-2** D-PHY driver `drivers/phy/sophgo/phy-sg2000-csi-dphy.c` with two PHY instances (4-lane, 2-lane),
  `phy_configure` deriving HS settle from `link-frequencies` via `phy_mipi_dphy_get_default_config`, lane
  routing/polarity from the endpoint, resets 70-73.
- **REQ-CSI-3** Error interrupts exposed as statistics (debugfs or `v4l2-ctl --log-status`), not as frame events.
- **REQ-CSI-4** Sensors are standard `drivers/media/i2c` sub-devices (bring-up with imx219/ov5647/imx290 class);
  CVITEK-shipped sensors need new drivers (2-4 weeks each, datasheets may be under NDA).
- **REQ-CSI-5** Reference structure: `drivers/media/platform/rockchip/rkisp1/rkisp1-csi.c` (518 lines),
  `drivers/media/platform/cadence/cdns-csi2rx.c` (1.1k), `drivers/media/platform/sunxi/sun6i-mipi-csi2/` (0.8k).

### 7.2 VI top raw/YUV capture (clean-room possible)

Facts [TRM `video/vi.rst`]: two identical VI top instances (0x0A0C2000, 0x0A0C4000) with YCbCr 8-bit storage
formats described; [VENDOR]: in the vendor design frames reach memory through the ISP's CSI bridge and front-end
DMA, not through the VI top itself; the minimal register sequence to capture RAW/YUV to DRAM without the ISP
processing stages is **unverified** and must be established from the TRM VI/ISP interface description and bench
work.

- **REQ-VI-1** A media-controller capture driver with one or two `/dev/videoN` nodes (videobuf2 dma-contig),
  RAW8/10/12 Bayer and YUV422 formats, frame-start/done interrupts, media graph sensor -> csi2rx -> vi.
- **REQ-VI-2** Works with libcamera's `simple` pipeline handler plus software ISP for a CPU-processed preview
  (>= 15 fps at 1080p target).
- **REQ-VI-3** This is the camera go/no-go gate: if RAW capture cannot be achieved from public documentation, the
  conditional ISP work is not started.

### 7.3 ISP (conditional on documentation or Legal-approved specification)

Facts [VENDOR]: register window 0x0A000000 size 0x80000 with 143 sub-blocks; three pre-raw front ends (FE0 with
4 channels at +0x0, FE1 and FE2 with 2 channels at +0x8000 and +0x10000; each with CSI bridge, black level,
RGB map, white-balance gain and DMA control), one pre-raw back end at +0x18000 (crop, black level, AF statistics,
defect pixel correction, write DMA), RAW top at +0x30000 (demosaic, lens shading, GMS, AE histograms, read DMA,
Bayer noise reduction, crop, luma map, white balance, chromatic aberration), RGB top (colour correction, RGB
and Y gamma, motion map, 3D LUT, dehaze, CSC, dither, vertical histogram, HDR fusion and local tone mapping),
YUV top (temporal noise reduction, frame-buffer compression, dither, chroma adjust, Y/C noise reduction, edge
enhancement, Y curve, DCI, local contrast), ISP top at +0x70000 (interrupt events, trigger, shadow control, DMA
cores), a command queue at +0x7FC00. Single interrupt (24) with FE frame start/done per channel, BE done, post
frame/shadow done, error and command-queue events. Default single-sensor flow: FE -> BE -> DRAM -> post (multi-pass),
with an optional online handshake to the scaler. Statistics produced by DMA into fixed-size blocks: AF, GMS,
AE histograms (long/short exposure), AWB, AWB post, DCI, vertical edge histogram, motion map. Vendor tuning
payload per frame: post stage 47,424 bytes, back end 16,632 bytes, front end 104 bytes, double-buffered with
per-block update flags. All 3A ran in userspace in the vendor stack; the kernel never computed exposure or
white balance. Kernel-side non-3A functions in the vendor design that a native driver must place in parameters,
statistics or controls: motion-level and DCI-level derivation from the motion-map/DCI statistics (hints for scaler
and encoder), HDR (FSWDR) hardware report readout, lens-shading buffer export, a black Y-curve applied until the
first tuning arrives, unity white-balance gains at start.

- **REQ-ISP-1** Architecture: V4L2 media-controller ISP driver with sub-devices (or a memory-to-memory job
  scheduler modelled on `drivers/media/platform/raspberrypi/pisp_be/`), capture video nodes (NV12/NV21/YUV420),
  a parameters node (`V4L2_BUF_TYPE_META_OUTPUT`) and a statistics node (`V4L2_BUF_TYPE_META_CAPTURE`) using the
  generic block framing of `include/uapi/linux/media/v4l2-isp.h`; parameters applied at frame boundaries,
  statistics delivered per frame. References: `rockchip/rkisp1` (11-12.6k lines depending on whether the uAPI header is counted), `arm/mali-c55` (5.5k),
  `amlogic/c3/isp` (4.1k), `dreamchip/rppx1` (2.8k).
- **REQ-ISP-2** No kernel-side 3A; no physical addresses or kernel pointers in the uAPI; buffers via vb2.
- **REQ-ISP-3** First target: single sensor, offline FE -> BE -> DRAM -> post, no HDR, no online scaler path.
- **REQ-ISP-4** Precondition: register-level documentation for the ISP from Sophgo, or a Legal-approved
  specification (section 2). Without it this requirement set is blocked.

Effort (assumption: documentation available, experienced V4L2 engineer): 20-35 engineer-weeks kernel.

### 7.4 VPSS scaler (conditional)

Facts [VENDOR]: two read-DMA inputs (IMG_D feeding SC_D for the display path, IMG_V feeding SC_V1-3 for the video
path; sources ISP online, DWA, memory), four scaler cores with crop, bicubic/bilinear/2-tap scaling with
coefficient tables, mirror/flip, CSC/quantisation, border, cover, per-core GOP overlay (2 layers x 8 windows);
maximum 2880 x 2880 per pass with tile mode above; four write DMAs with format conversion, flip, slice-buffer
handshake to the encoder. The scaler enables share the SC_TOP CFG1 word with the display enable (5.2).

- **REQ-VPSS-1** V4L2 mem2mem driver (one device, instances scheduled sequentially) modelled on
  `drivers/media/platform/rockchip/rga/` (2.3k lines): crop, scale, mirror/flip, CSC, format conversion; no
  per-core GOP, privacy mask or encoder handshake.
- **REQ-VPSS-2** Shares SC_TOP via the syscon and the `sc` interrupt per REQ-SCTOP-2.
- **REQ-VPSS-3** Resizer sub-devices inside the ISP media graph (online mode) are a later increment.
- Precondition as REQ-ISP-4. Effort 4-8 engineer-weeks (offline m2m), +6-12 online.

### 7.5 libcamera

- **REQ-LCAM-1** Pipeline handler `src/libcamera/pipeline/sophgo/` modelled on `rkisp1` (media graph enumeration,
  stream configuration, params/stats plumbing, `DelayedControls` for sensor exposure/gain).
- **REQ-LCAM-2** IPA module `src/ipa/sophgo/` with newly written algorithms built on libcamera's `libipa`
  (AGC via `agc_mean_luminance`, AWB via `awb_bayes`/grey world, BLC, CCM, LSC, gamma; AF later) and a YAML tuning
  file format with a documented calibration procedure (colour checker, flat field). The vendor 3A binaries and
  tuning files MUST NOT be used (REQ-LIC-3).
- **REQ-LCAM-3** Acceptance: `cam`/`qcam` preview with AGC and AWB converging within 2 s under D65 and tungsten;
  agreed non-parity quality targets (deltaE against a colour checker, SNR at a given illuminance);
  `lc-compliance` passes. Image quality parity with the vendor tuning is not a requirement.

Effort: pipeline handler + IPA 10-30 engineer-weeks (10-20 if algorithm work builds on `libipa`, more if from
scratch), assuming the ISP driver exposes stats and params per REQ-ISP-1.

---

## 8. Non-functional requirements

- **REQ-NF-1** Upstreamability: coding style, bindings, no vendor-specific uAPI beyond documented V4L2 meta formats;
  patches split per subsystem (dt-bindings, phy, drm, media, riscv dts); coordinate on `sophgo@lists.linux.dev`
  (maintainers Chen Wang, Inochi Amaoto per `MAINTAINERS` and the Sophgo upstream wiki).
- **REQ-NF-2** Security: no physical addresses, kernel pointers or GPIO numbers in any uAPI; no `/dev/mem`;
  bounds-checked parameters; dma-buf import validates contiguity when the engine has no IOMMU (the SG2000 has no
  IOMMU for these engines, assumed, unverified).
- **REQ-NF-3** Performance targets (first milestone): display 1080p60 scan-out with one overlay; camera 1080p30
  capture without frame drops over 10 minutes; CPU load for software ISP preview is not constrained.
- **REQ-NF-4** Testing tools: `modetest`, `kmscube`/Weston, igt-gpu-tools, `v4l2-compliance`, `v4l2-ctl`,
  libcamera `cam`/`lc-compliance`, `dtbs_check`, `checkpatch`, `W=1` warning-free build, sparse.
- **REQ-NF-5** Documentation: each driver with a `Documentation/` entry where the subsystem expects one
  (`Documentation/admin-guide/media/`, `Documentation/userspace-api/media/v4l/metafmt-*.rst` for meta formats);
  register maps used by the implementation kept as living documentation with TRM section references.
- **REQ-NF-6** No reliance on the vendor bootloader state: drivers initialise hardware from reset; boot-logo
  handoff is optional (REQ-DRM-9).

---

## 9. Verification items and known unknowns (measure on the bench before or during implementation)

| # | Item | Why it matters | How to measure |
|---|---|---|---|
| 1 | SC_TOP `INTR_STATUS` (0x0A080034) per-bit write-1-to-clear? | decides shared IRQ vs demultiplexer (REQ-SCTOP-2) | on a vendor 5.10 image: write one bit, read back |
| 2 | Meaning of clock-controller register 0x03002840 bit 1 (vendor sets it when enabling any display output; bit 3 is the DISPPLL synthesizer enable in `clk-cv1800.c:124-126`) | REQ-CLK-1 completeness | compare `clk_summary`/register dumps with and without display active |
| 3 | VIP_SYS CLK_CTRL0 bit 4 written by the DSI PHY PLL code | REQ-VIPSYS clock model | register dump during DSI bring-up on the vendor image |
| 4 | Does pure MIPI DSI/CSI operation need any pin function change? | REQ-PIN-1 | diff FMUX registers before/after vendor display/camera start |
| 5 | DSI non-burst and command mode on silicon | REQ-DRM-5 scope | TRM says supported; test with a panel needing sync pulses |
| 6 | D-PHY TX PLL formula, HS prepare/zero/trail defaults; CSI RX HS settle, deskew phases and clock-lane direction rules | first light; silent link failures otherwise | TRM PHY register tables + scope/logic analyser; capture working vendor register values as oracle only if Legal permits |
| 7 | Minimal VI/ISP register sequence for RAW/YUV capture to DRAM without ISP processing | REQ-VI-1 feasibility | TRM VI chapter; experiments |
| 8 | Whether `CLK_CSI0_RX_VIP`/`CLK_CSI1_RX_VIP` gates must be enabled (the vendor code never requests them) | REQ-CSI-2 | enable and observe PHY lock |
| 9 | Achievable `CLK_CAM0/1` rates (24/25/26/27/37.125 MHz) through `clk-cv1800.c` | sensor MCLK | scope on MCLK pin |
| 10 | Bootloader pre-initialisation of the display (boot logo) on the target board | REQ-DRM-9 | read timing-generator enable at probe |
| 11 | Reset pulse width and ordering for VIP blocks | REQ-VIPSYS-4 | bench |
| 12 | SG2002 (LicheeRV Nano) VIP identity with SG2000 | board choice | compare TRM/ID registers |
| 13 | In-flight upstream series for CV18xx CSI/ISP/DRM/DSI | avoid duplicate work | manual search on lore.kernel.org `sophgo/`, dri-devel, linux-media (was unreachable from the assessment environment) |
| 14 | Is the CSI-2 RX controller / D-PHY a licensed IP block (Cadence, Synopsys) with an adaptable mainline sub-device, or CVITEK-proprietary? | could shrink REQ-CSI effort | compare TRM register names with the mainline Cadence/Synopsys receiver drivers |
| 15 | Register 0x0A0880F8 (display block + 0xF8) is written by the vendor CSI driver; purpose unverified | SC_TOP/display ownership (REQ-SCTOP) | bench: diff before/after camera start on the vendor image |

---

## 10. Reference material index

TRM (BSD-2-Clause) in `github.com/sophgo/sophgo-doc`, `SG200X/TRM/contents/en/` (register tables in
`contents-share/video/*.table.rst`; a PDF release `sg2000-trm-v1.0` exists): `video/vdp.rst` (VDP DISP
0x0A088000-0x0A0883FF, OSD 0x0A088800-0x0A0889FF, LVDS/BT formats), `video/mipi_tx.rst` (controller 0x0A08A000,
PHY 0x0A0D1000), `video/mipi_rx.rst` (PHY 0x0A0D0000/0x0300/0x0600, CSI 0x0A0C2400/0x0A0C4400, sub-LVDS
0x0A0C2200/0x0A0C4200), `video/vi.rst` (VI top 0x0A0C2000/0x0A0C4000, storage formats); plus `clock`, `reset`,
`pinmux-pinctrl`, `system-control`, `system-overview` chapters. The TRM does **not** cover the ISP, VPSS, VIP_SYS,
SC_TOP interrupt register, codec, JPEG, TPU, IVE or DWA.

Mainline reference drivers (7.3): display `drivers/gpu/drm/sun4i/sun6i_mipi_dsi.c` (1265 lines, non-DesignWare
DSI host with software ECC/CRC and external PHY), `drivers/gpu/drm/sprd/sprd_dsi.c` (1066, in-driver PHY PLL),
`drivers/gpu/drm/ingenic/ingenic-drm-drv.c` (1683, primary + overlay planes, fbdev-dma), `drivers/gpu/drm/mxsfb/`
(2327), `drivers/gpu/drm/imx/lcdc/imx-lcdc.c` (534), `drivers/phy/allwinner/phy-sun6i-mipi-dphy.c`;
camera `drivers/media/platform/rockchip/rkisp1/` (CSI sub-device, ISP, params/stats), `cadence/cdns-csi2rx.c`,
`sunxi/sun6i-mipi-csi2/`, `nxp/imx-mipi-csis.c`, `raspberrypi/pisp_be/` (memory-to-memory ISP), `amlogic/c3/isp/`,
`dreamchip/rppx1/`, `rockchip/rga/` (mem2mem scaler), `nxp/dw100/` (dewarp); frameworks
`drivers/media/v4l2-core/v4l2-isp.c`, `drivers/dma-buf/heaps/cma_heap.c`, `drivers/reset/reset-simple.c`,
`drivers/soc/sophgo/`.

libcamera: `src/libcamera/pipeline/rkisp1/`, `src/libcamera/pipeline/simple/`, `src/ipa/rkisp1/algorithms/`,
`src/ipa/libipa/{agc_mean_luminance,awb_bayes}.cpp`, `src/ipa/simple/` (software ISP).

Ecosystem: Armbian `armbian/build` (`config/sources/families/include/sophgo-sg200x_common.inc`,
`patch/kernel/archive/sophgo-sg200x-7.3/`, kernel configs `config/kernel/linux-sophgo-sg200x-*.config`); Sophgo
upstream status wiki (`github.com/sophgo/linux/wiki`, DRM/Media/TPU "Not Started" as of 2026-09-02); BadgeOS
(`github.com/NixVegas/BadgeOS`, wants DRM + DSI for an LT8912B); mailing list `sophgo@lists.linux.dev`.

---

## 11. Delivery milestones and gates (clean-room programme)

| Gate | Deliverable | Exit criteria |
|---|---|---|
| G-L Legal | Clean-room protocol approved; source rules of section 2 confirmed; decision on ISP/VPSS documentation path | written approval |
| G0 Platform | `sg2000-vip.dtsi`, VIP_SYS and SC_TOP syscons with bindings, CMA pools, kernel config; bench items 1-4, 11 measured | `dtbs_check` clean; reset toggles observed; `clk_summary` shows VIP clocks |
| G1 Display first light | REQ-DRM-1/5/6/7 minimal | `modetest` pattern on panel or HDMI bridge; vblank rate within 1% |
| G2 Display MVP | REQ-DRM-1..10 | section 6.3 criteria; RFC posted to dri-devel |
| G3 Raw camera | REQ-CSI-1..4, REQ-VI-1..2 | 1000 RAW10 frames with zero CSI errors; libcamera simple + software ISP preview >= 15 fps; `v4l2-compliance` passes |
| G4 ISP (conditional) | REQ-ISP-1..3 | NV12 1080p30 for 30 min with per-frame params and stats; manual WB/gain/gamma via params visibly correct; `v4l2-compliance` passes; uAPI RFC on linux-media |
| G5 libcamera camera | REQ-LCAM-1..3 | AGC/AWB converge < 2 s; agreed quality targets met |
| G6 Upstream | bindings, DRM + PHY, CSI-RX/VI merged or in `-next` | maintainers' acceptance |

Indicative effort (engineer-weeks; two to three engineers; excludes Legal, procurement and review latency):
platform 2-4; display minimal 9-17, full 18-30; CSI-RX + D-PHY + raw capture 8-16; ISP 20-35 (conditional); VPSS
m2m 4-8 (conditional); libcamera 10-30; sensors 2-4 each; upstream review 8-16 engineer-time over 6-18 calendar
months (assumption).

---

## Appendix: glossary

- **VIP**: the vendor's name for the video input/processing subsystem (ISP, scalers, display, DSI, CSI).
- **VIP_SYS**: its top-level control register block (resets, clock gates, dividers, AXI switches) at 0x0A0C8000.
- **SC_TOP / SCL_TOP**: the scaler top register page at 0x0A080000 that also hosts display enable, output mux
  and the shared interrupt registers.
- **VDP DISP / OSD**: TRM names for the display controller and its overlay (the vendor calls them DISP and GOP).
- **GOP**: graphics overlay plane (2 layers x 8 windows per display, also per scaler core).
- **CIF**: vendor name for the camera interface (CSI-2 receiver, sub-LVDS, parallel inputs).
- **VI**: vendor name for the camera input and ISP driver; TRM "VI" is the capture top block only.
- **VPSS**: vendor name for the scaler/processing subsystem.
- **VB / ION**: the vendor's buffer-pool model on the Android ION allocator (not used in this design).
- **CMA / dma-heap**: mainline contiguous memory allocator and its userspace dma-buf interface.
- **W1C**: write-1-to-clear interrupt status semantics.
- **CMO**: cache maintenance operations (T-Head C906 `dcache.*` instructions used by the mainline errata path).
