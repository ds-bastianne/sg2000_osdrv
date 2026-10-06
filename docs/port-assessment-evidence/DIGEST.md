# Phase-1 digest: SG2000 (CV181x) osdrv -> newer kernel feasibility inputs

All facts below were verified by Phase-1 readers with file:line evidence (full reports: this directory, *.txt:
sys-base, cif-snsr, vi-isp, vpss, vo-mipitx-fb, codec-others, api-delta, vendor-deps, mainline-infra, ecosystem).
Repos: /home/user/sg2000_osdrv (vendor out-of-tree drivers, interdrv/v2, GPL-2.0 modules, snapshot 2024-07-31),
/home/user/sg2000_linux_5.10 (vendor 5.10.4 kernel), /home/user/mainline_linux (7.3-rc6, 2026-10-06).
Extra cloned sources (ecosystem reader): /tmp/claude-0/-home-user/be69dd8d-dbf3-58dc-9770-31db5f4edebc/scratchpad/ext/
{armbian-build, BadgeOS, sophgo-sg200x-debian (has real board/SoC DTS: configs/common/dts/sg200x/sg200x_base.dtsi, configs/duos/dts/*.dts),
linux-common (Sophgo vendor 6.12.y kernel with drivers/gpu/drm/cvitek for CV186x), osdrv-bm1688 (vendor osdrv branch with 6.x guards),
sophgo-doc (SG2000 TRM rst), sophpi (SDK manifests), libcamera} and wiki/linux.wiki (sophgo upstream status, 2026-09-02).

## Architecture of osdrv (what must be ported)
- No V4L2, no DRM. Every module is a misc/char device with custom ioctls; buffers are identified by PHYSICAL ADDRESS in the uAPI.
- Foundation: sys (ION wrapper, cache ops by paddr, module bind graph) -> base (VB buffer pools, per-channel job queues, inter-module
  callback bus with 13 module ids, VIP_SYS register block owner at 0x0A0C8000 for VIP-domain resets/clock gates). ~55 base + 17 sys exports.
  VB_BLK handles given to userspace are raw kernel pointers. Dependency chain (Module.symvers): sys -> base -> {vpss, cif, rgn, dwa, ive};
  vo <- base+vpss+sys; fb <- base+vpss; vi <- base+sys; vcodec -> jpeg -> cvi_vc_drv; rtos_cmdqu -> fast_image.
- ION: all contiguous memory comes from vendor ION CARVEOUT heap via sys_ion_alloc/_nofd (confined to sys.c, 15 call sites; 48 external
  call sites use the exported sys_ion_* wrappers; direct cvi_ion_*/ion.h users outside sys: tpu, fast_image, dwa ldc_test, vcodec vdi.h, jpeg jdi.h).
  Userspace middleware also uses /dev/ion directly (ION_IOC_ALLOC + CVITEK flush/invalidate-by-paddr ioctls) for the ISP working pool.
- Cache maintenance: sys_cache_flush/invalidate (+fb, tpu, vpss RGN_EX) call arch_sync_dma_for_device() directly; vendor kernel exports it,
  mainline does NOT (only __dma_sync_single_for_cpu/device exported). SG2000 is dma-noncoherent (sg2000.dtsi:21); mainline handles C906 via
  ERRATA_THEAD_CMO (depends on ERRATA_THEAD, which has NO default and is NOT set in arch/riscv/configs/defconfig -> must be enabled).
- Clocks: 45 consumer clock names; EVERY one has a same-named mainline clock in drivers/clk/sophgo/clk-cv1800.c (sophgo,cv1800.h), e.g.
  clk_disp->CLK_DISP_VIP 121, clk_dsi->CLK_DSI_MAC_VIP 122, clk_isp_top->CLK_ISP_TOP_VIP 111, clk_raw->CLK_RAW_VIP 134, clk_sc_top->114,
  clk_cam0/1->CLK_CAM0/1. Only DT renumbering needed. BUT cif.c (38 sites), scaler.c, jpeg_common.c poke clock-controller registers
  0x03002xxx directly (cam PLL power/dividers, DISPPLL bit at 0x03002840) -> must become CCF calls or they race the mainline clk driver.
- Resets: mainline reset-simple 'sophgo,cv1800b-reset' (reg 0x1000 -> 32768 indices) covers top-level RST_VIPSYS 6, RST_VCSYS 95, H264C/H265C/JPEG,
  TPU; CSIPHY/DSIPHY indices 68-73 have no macro in cv18xx-reset.h but raw numbers work. ALL VIP-internal resets (isp_top, img_*, sc_*, disp, bt,
  dsi_mac, csi_mac0-2, ldc, dsi_phy, csi_phy0, csi_be, ive) are raw writes to VIP_SYS (0x0A0C8000) via base.ko -> no mainline provider;
  need a syscon + reset-simple/clk-gate node or keep base.ko as owner.
- Pinctrl: vo.c (52 sites) and cif.c (19 sites) write FMUX registers 0x03001000 directly via vendor PINMUX_CONFIG macro (parallel/BT/TTL modes
  only, apparently not needed for pure MIPI). Mainline pinctrl-sg2000 owns that block and defines all the pads -> convert to DT pinctrl states.
- Board DTS: absent from both kernel repos (SDK 'build' repo). Real vendor DTS is available in scratchpad ext/sophgo-sg200x-debian.

## Display path (vo 6.9k LOC, fb 0.9k, display HAL inside vpss: scaler.c lines ~2249-4300 + dsi_phy.c 440)
- HW: DISP timing generator + 3-plane DMA read engine (SCL_TOP 0x0A080000 + 0x8000), DISP GOP overlay (2 layers x 8 ARGB/LUT windows),
  VO output mux (DSI / BT601/656/1120 / LVDS / I80-MCU / parallel RGB), CVITEK-specific MIPI DSI MAC (+0xA000; video BURST mode only,
  1/2/4 lanes, SW-computed ECC/CRC, 16-byte LPDT FIFO), D-PHY at 0x0A0D1000 (lane remap, P/N swap, in-driver PLL math that also writes
  VIP_SYS CLK_CTRL0 bit), 65-node gamma LUT, VBAT power-fail IRQ. Single device/layer/channel. I80 path incomplete in vendor code.
- vo/mipi_tx/fb own NO registers and NO IRQ: everything goes through 72 EXPORT_SYMBOL_GPL from vpss (59 sclr_* + 13 dphy_*), the shared
  'sc' IRQ is owned by vpss (single INTR_STATUS register shared by scalers and display vblank) and forwarded to vo by callback.
  SC_TOP config register (disp enable) is shared with scaler enables under one spinlock -> DRM driver and any scaler driver must share a regmap.
- Panel init is 100% userspace: /dev/cvi-mipi-tx ioctls push lane map, timing, pixel clock and raw DCS packets; kernel has no panel tables.
  Frames arrive as paddrs of VB blocks (from VPSS bind or VO_SDK_SEND_FRAME); fbdev (cvifb) is a thin wrapper over GOP layer 1.
- Mainline fit: DRM/KMS (1 CRTC + primary plane + GOP overlay planes + DSI encoder with mipi_dsi_host_ops.transfer + separate D-PHY phy driver;
  drm_gem_dma + drm_fbdev_dma gives /dev/fb0). References: sun6i_mipi_dsi.c 1265 L, sprd_dsi.c 1066 L, ingenic-drm 1683 L, mxsfb 2327 L.
  106 DSI panel drivers and the LT8912B DSI->HDMI bridge exist upstream. fbdev is deprecated for new drivers.
- Public docs: SG2000 TRM (sophgo-doc) documents VDP DISP 0x0A088000, OSD 0x0A088800, MIPI TX ctrl 0x0A08A000 + PHY 0x0A0D1000 at register level.
- Sophgo's own 6.12 vendor kernel (linux-common) has a GPL DRM/KMS driver for the sibling CV186x (drivers/gpu/drm/cvitek, 5.3k LOC: disp, dsi,
  lvds, dw_hdmi, mipipll) -> architecture template; register names do not overlap with CV181x headers, so each write must be re-validated.
- Duo S mainline DTS muxes MIPI_TX pads to sdhci1 (SDIO WiFi) -> DSI bring-up needs a board with free DSI pads or re-muxing.
- Effort ranges from readers: minimal DRM (DSI, 1 plane): 8-14 eng-weeks (first picture 3-6); full (overlays, BT/LVDS, gamma): 18-30;
  forward-port vo/mipi_tx/fb as-is on top of ported base/sys/vpss: 3-6 (vpss agent) / 6-12 (vo agent).

## Camera path
- cif (CSI-2 RX, 9.7k LOC incl. 3.4k register headers): pure bridge, NO DMA; 3 MACs (2 MIPI-capable: MAC0 4-lane PHY, MAC1 2-lane), 6 physical
  D-PHY lanes freely routable with P/N swap, sub-LVDS/HiSPi/BT656/BT1120/TTL inputs, VC/DT/DOL HDR. Userspace pushes the whole receiver config
  (lane map, hs_settle, HDR mode) in one ioctl; streaming starts immediately. Uses reset framework (phy0/phy-apb0/phy1/phy-apb1 = 70-73),
  6 clocks, snsr-reset GPIOs; also pokes VIP_SYS MAC dividers and 0x03002xxx PLL regs. 2 IRQs only count errors.
  snsr_i2c: I2C relay for frame-synchronous exposure/gain bursts (depends on vendor i2c_adapter.i2c_idx and I2C_M_WRSTOP -> I2C_M_STOP).
  Mainline fit: V4L2 subdev (like rkisp1-csi.c 518 L, cdns-csi2rx 1113 L, sun6i-mipi-csi2 771 L) + phy driver; snsr_i2c becomes unnecessary
  (sensors as drivers/media/i2c subdevs). Common CVITEK sensors (gc2053/gc2083/gc4653, SmartSens sc*, os04a10) have NO mainline driver;
  imx219/imx290/imx335/imx415/ov5647/ov5640 do. TRM documents MIPI RX and VI top at register level.
- vi/ISP (24k LOC; register-programming layer vip/*_ip_ctrl.c 9k LOC with ~zero kernel API use + 28k lines generated register headers;
  kernel glue 11.8k LOC): 3 pre-raw FEs -> 1 BE -> RAWTOP/RGBTOP/YUVTOP post stage, 143 register blocks in 0x0A000000+0x80000, single 'isp' IRQ,
  hi-tasklet + 4 SCHED_FIFO kthreads. ALL 3A (AE/AWB/AF) and tuning run in closed userspace: kernel only DMAs statistics into memblocks
  (paddrs published by ioctl) and applies double-buffered tuning nodes (post node 47 KB, be 16.6 KB, fe 104 B) written by userspace into
  kzalloc'd memory exposed by PHYSICAL ADDRESS + kernel virtual pointers (needs /dev/mem; STRICT_DEVMEM breaks it). ISP working pool is
  allocated by userspace from ION and handed over as paddr/size. Sensor drivers are userspace libs; exposure/gain register lists are queued
  per frame and fired by the kernel via snsr_i2c. Default single-sensor path fe->be->DRAM->post (multi-pass, M2M-like); optional online
  ISP->VPSS handshake. Mainline fit: V4L2 media-controller ISP with params (META_OUTPUT) / stats (META_CAPTURE) using the new generic
  include/uapi/linux/media/v4l2-isp.h framing (used by rkisp1 ext params, rcar-isp, dreamchip rppx1); references rkisp1 12.3k L, mali-c55 5.5k,
  c3-isp 4.1k, pisp_be 1.8k (M2M). libcamera has NO pipeline handler for Sophgo; vendor 3A is closed -> would have to be re-implemented.
  TRM does NOT document the ISP; only GPL osdrv headers do (so an in-kernel ISP driver is a GPL derivative, not clean-room).
- vpss (25.8k LOC, 23.4k compiled): IMG_IN x2 + 4 scalers (SC_D, SC_V1-3; IMG_V shared by 3 outputs = 1-in/3-out) + ODMA x4 + per-SC GOP,
  online (ISP->VPSS without DRAM) and offline modes; 49 ioctls; 96 call sites into base/sys VB framework. Mainline fit: offline scaler as
  V4L2 m2m (rockchip rga 2345 L) with 1:1 limitation; online mode needs it as a subdev inside the ISP media graph. Also hosts the display HAL.
- Effort ranges from readers: cif lift-and-shift 2-4 wk; cif V4L2 subdev + D-PHY 4-8 (+1-3 HDR, +2-4 per missing sensor);
  vi lift-and-shift 6-10; vi native V4L2 ISP 20-35 kernel + 15-30 libcamera/IPA/3A; vpss lift 3-6; vpss m2m 4-8; vpss subdev part 6-12.

## Other modules (codec-others report)
- Codec: Chips&Media CODA980 (H.264, 0x9800) + WAVE420L (HEVC, 0x4201) + CODAJ12-class JPEG. cvi_vc_drv compiles 55.7k LOC, ~70% C&M host API
  and sample code inside the kernel; firmware read by kernel from /usr/share/fw_vcodec via filp_open; firmware redistribution licence unclear.
  Mainline wave5 supports only WAVE5xx; coda supports CODA960 not 980; no CODAJ12 driver. New V4L2 stateful driver: 30-55 wk; lift: 6-12 wk.
- TPU: already forward-ported to mainline 7.2/7.3 by a Cvitek engineer in Armbian (patch 0064: ION -> dma-buf import from CMA heap,
  GET_DMABUF_PADDR ioctl, dma_sync_sgtable_*, uAPI binary-compatible with libcviruntime). This is the proven ION-removal recipe.
- Drop in favour of mainline: rtc (rtc-cv1800), saradc (sophgo-cv1800b-adc), wdt (dw_wdt + DTS node), pwm (pwm-sophgo-sg2042 register map
  identical, needs cv18xx compatible; Armbian carries a pwm patch), mailbox (cv1800-mailbox.c matches rtos_cmdqu registers; hwspinlock at +0xc0
  missing), mon/fast_image/clock_cooling (debug/niche). rgn is software-only and follows VPSS/VO. dwa -> V4L2 m2m (nxp dw100 1737 L). ive: no framework.

## Cross-cutting 5.10 -> 7.3 breakage (api-delta report)
- Blockers: ION removal (design work, 2-4 wk shim in sys.c); arch_sync_dma not exported (0.5-1 wk); strncpy REMOVED from 7.3 (171 sites, 1-2 wk);
  -Werror -Wextra in 18 Makefiles vs much stricter default warnings (remove -Werror first; restoring it 2-4 wk).
- Mechanical: 26 remove() -> void, 13 class_create, of_gpio.h removal (4 lookups, ~20 legacy gpio sites), MODULE_IMPORT_NS("DMA_BUF"),
  sched_setscheduler not exported + MAX_USER_RT_PRIO gone (14 sites), timers (7), vm_flags_set (4), DEFINE_SEMAPHORE (2), thermal 5-arg (2),
  FBINFO_DEFAULT (1), PDE_DATA (28), vendor headers (streamline_annotate, tee_cv_private, pinctrl-cv181x.h), efuse exports (6), i2c_idx (4), PWM API rewrite.
- Vendor kernel also auto-boosts RT priority of named cvitask_* threads (CONFIG_SCHED_CVITEK) and ignores vermagic; both disappear on mainline.
- Compile-only estimate for ALL of interdrv/v2 against 7.3 (-Werror removed, no hardware work): 8-15 eng-weeks.
- Infrastructure layer before VI/VO bring-up (vendor-deps report): 8-17 eng-weeks.

## Ecosystem (as of 2026-10-06)
- sophgo/linux wiki (2026-09-02): DRM, Media, TPU rows 'Not Started'; base peripherals upstream (clk 6.10, pinctrl 6.12, reset 6.17, mailbox 6.16, ...);
  under review: efuse, timer, watchdog, PWM v8, thermal v5, remoteproc C906L v2, I2S v4. Duo S minimal DTS in 7.3.
- No public CV18xx CSI/ISP/DRM/DSI patch series found (lore was blocked; search-snippet based -> 'likely').
- Armbian sophgo-sg200x on mainline 7.2/7.3 with ~60 patches (thermal, mdio, timer, wdt, pwm, remoteproc C906L + mailbox DTS, efuse nvmem, I2S,
  TPU, 128 MiB CMA pool) -> natural integration target for a display/camera port; no video drivers.
- BadgeOS (mainline) wants a DRM/KMS driver for DISP+DSI to drive the LT8912B HDMI bridge; none exists yet.
- scpcom/Fishwaldo Debian: vendor 5.10 + osdrv, DSI panels and LT9611 DSI->HDMI (bootloader does part of panel init), cameras via osdrv.
- Sophgo keeps SG200x SDKs on linux_5.10 (osdrv weekly releases through 2026-08-24; local snapshot is 2 years stale -> rebase first).
  Sophgo's BM1688/CV186 line uses linux-common 6.12.y with a vendor DRM driver and an osdrv bm1688 branch with 6.x API guards (still ION).
- libcamera: no Sophgo pipeline handler.
