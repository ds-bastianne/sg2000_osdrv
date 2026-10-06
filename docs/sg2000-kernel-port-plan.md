# SG2000 (CV181x) multimedia drivers on a modern kernel: feasibility assessment and porting plan

Status: assessment, 2026-10-06. Inputs: this `osdrv` snapshot (branch `sg200x-dev`, last commit 2024-07-31),
the vendor Linux 5.10.4 tree (`sg2000_linux_5.10`), mainline Linux 7.3-rc6 (`mainline_linux`, 2026-10-06),
plus public sources listed in section 7. Evidence is cited as `path:line` in the repository named by context.
Effort figures are engineer-week ranges with stated assumptions; they are estimates, not measurements.
Nothing in this document was compiled or run on hardware.

---

## 1. Bottom line

**Porting is feasible, but it is not a port in the usual sense for display, and camera feasibility depends on
whether the closed vendor middleware is kept.**

| Question | Answer | Confidence |
|---|---|---|
| Can the display path (VO, MIPI DSI TX, framebuffer) work on mainline 7.x? | Yes. The realistic route is a **new DRM/KMS driver** that reuses the vendor register sequences; the vendor `vo`/`fb` code is an fbdev/ioctl design with userspace panel init and cannot be upstreamed as-is. First picture 3-6 weeks, reviewed driver 8-14 weeks (DSI only), 18-30 weeks full-featured. | medium-high |
| Can the camera path (CSI-2 RX, ISP, scaler) work on mainline 7.x? | **Yes if the vendor middleware (closed 3A/ISP tuning, userspace sensor drivers) is kept**: forward-port `sys/base/cif/snsr_i2c/vi/vpss` as out-of-tree modules keeping the vendor ioctl ABI, roughly 15-30 weeks including the shared infrastructure. **A native V4L2 + libcamera camera stack is a multi-quarter programme** (20-35 weeks kernel + 15-30 weeks libcamera/3A) because all 3A runs in closed userspace and the ISP is documented only by GPL headers. | forward-port: medium; native: low-medium |
| Does mainline already have the SoC plumbing? | Yes. Clocks (every vendor clock name has a mainline ID), top-level resets, pinctrl, DTS for Duo S, DMA, I2C/SPI/UART, USB, SD/eMMC, Ethernet, RTC, ADC, mailbox, audio are upstream. Nothing exists for VI/ISP, CSI-RX, VPSS, VO/DSI, codec, JPEG, TPU, IVE, DWA. | high |
| Is anyone else doing this? | No upstream or public RFC work for CV18xx display or camera was found. Armbian runs Duo S on mainline 7.2/7.3 with ~60 patches including a Cvitek-authored TPU port that solved the ION-removal problem the same way this plan proposes. Sophgo itself wrote a GPL DRM driver for the sibling CV186x on a 6.12 vendor kernel, which is a usable template. | medium (list archives were not reachable) |
| Single biggest technical blocker? | The Android ION allocator and physical-address buffer ABI used by every module. It is confined behind one wrapper file (`sys.c`), so a CMA/dma-buf shim is bounded work (2-4 weeks), but userspace also talks to `/dev/ion` directly for the ISP pool. | high |
| Single biggest non-technical blocker? | The camera image pipeline's intelligence (AE/AWB/AF, tuning, sensor drivers) is closed vendor userspace. Any plan that drops the vendor middleware must re-create it. | high |

Recommended approach (details in sections 8 and 9): a staged hybrid. Prove the kernel-side feasibility
with a compile-only forward-port of the whole module set (8-15 weeks, mostly mechanical), bring the camera up
first as a forward-port with the vendor middleware retained, and build the display as a new DRM/KMS driver
instead of forward-porting `vo`/`fb`. Decide about a native V4L2/libcamera camera stack only after the
forward-ported camera works and the 3A question (section 11) is answered.

---

## 2. What was examined and how

- All 24 module directories of `interdrv/v2` (170 `.c`, 340 `.h`) were read, with deep reads of `sys`, `base`,
  `cif`, `snsr_i2c`, `vi`, `vpss`, `vo`, `fb`, and a structural survey of `vcodec`/`jpeg`/`cvi_vc_drv`, `rgn`,
  `dwa`, `ive`, `tpu`, `mon`, `rtos_cmdqu`, `fast_image`, `pwm`, `rtc`, `wdt`, `saradc`, `clock_cooling`, `wiegand`.
- Every kernel symbol or header the modules take from the vendor 5.10 kernel was traced to its vendor source and
  checked against mainline 7.3.
- Mainline 7.3 was surveyed for SG2000 support (DTS, clk, reset, pinctrl, dma-buf heaps, V4L2, DRM, codec,
  accel, mailbox) and for reference drivers of comparable hardware.
- Public ecosystem sources were cloned and read (section 7). Web pages on github.com, milkv.io and lore.kernel.org
  were not reachable from this environment; `git clone` was. Claims that rest only on search snippets are marked.
- Board device trees are **not** in either kernel repository (they live in the SDK `build` repository). A copy of
  the vendor SoC and board DTS was obtained from a community Debian build repository and is used in Appendix A.

---

## 3. Baseline facts

### 3.1 Mainline 7.3 state for SG2000

Present (file evidence in `mainline_linux`):

| Area | Mainline 7.3 | Evidence |
|---|---|---|
| SoC DTS | `cv180x.dtsi`, `cv181x.dtsi`, `sg2000.dtsi`, `sg2000-milkv-duo-s.dts`, `sg2002-*.dts`; `soc { dma-noncoherent; }` | `arch/riscv/boot/dts/sophgo/sg2000.dtsi:21` |
| Clocks | Full VIP/VC tree: `CLK_DISP_VIP`, `CLK_DSI_MAC_VIP`, `CLK_DSI_ESC`, `CLK_ISP_TOP_VIP`, `CLK_CSI_*`, `CLK_SC_*`, `CLK_IMG_*`, `CLK_DWA_VIP`, `CLK_IVE_VIP`, `CLK_RAW_VIP`, `CLK_H264C/H265C/JPEG`, `CLK_TPU`, PLLs | `drivers/clk/sophgo/clk-cv1800.c:627-850`, `include/dt-bindings/clock/sophgo,cv1800.h` |
| Resets | `reset-simple` for `sophgo,cv1800b-reset` (4 KiB block, so indices up to 32767 are accepted); macros for VIPSYS, VCSYS, H264C, H265C, JPEG, TPU; no macros for CSIPHY/DSIPHY (68-73) | `drivers/reset/reset-simple.c:154,183`; `arch/riscv/boot/dts/sophgo/cv18xx-reset.h` |
| Pinctrl | `pinctrl-sg2000.c` with all MIPI TX/RX and VIVO pads | `drivers/pinctrl/sophgo/pinctrl-sg2000.c:198-228,401-408` |
| Non-coherent DMA | T-Head CMO errata ops for C906 | `arch/riscv/errata/thead/errata.c:74-125`; `ERRATA_THEAD` has no default and is **not** in `arch/riscv/configs/defconfig` |
| Memory | dma-buf heaps: `system` and `cma` only; one heap per `shared-dma-pool` + `reusable` reserved-memory node; no in-kernel heap allocation API | `drivers/dma-buf/heaps/cma_heap.c:397-421`; `include/linux/dma-heap.h:43-47` |
| Peripherals | DMA mux, AXI DMA, I2C/SPI/UART/GPIO (DesignWare), SDHCI, dwmac, USB PHY and dwc2, RTC, SARADC, mailbox, I2S/codec | `cv180x.dtsi`; `drivers/mailbox/cv1800-mailbox.c`; `sound/soc/sophgo/` |
| Generic ISP uAPI | Versioned block-based params/stats framing shared by rkisp1, rcar-isp, rppx1 | `include/uapi/linux/media/v4l2-isp.h`, `Documentation/driver-api/media/v4l2-isp.rst` |
| DRM helpers | `drm_gem_dma`, `drm_fbdev_dma`, `drm_mipi_dsi` host API, 106 DSI panel drivers, LT8912B bridge | `drivers/gpu/drm/drm_fbdev_dma.c`, `drivers/gpu/drm/bridge/lontium-lt8912b.c` |

Missing: any driver, binding or DTS node for VI/ISP, CIF/CSI-2 RX, VPSS, VO/DISP, MIPI DSI TX, D-PHY (RX or TX),
video codec, JPEG, TPU, IVE, DWA/LDC, RGN; the VIP_SYS sub-controller (see 3.3); reserved-memory/CMA pools;
a mailbox DTS node. `grep -rli 'sophgo\|cvitek\|cv18' drivers/gpu drivers/media drivers/staging` is empty.

On the Milk-V Duo S mainline DTS the MIPI TX pads are muxed to `sdhci1` (SDIO WiFi), so DSI bring-up on that
board needs a different pin configuration or a board with a free DSI connector
(`sg2000-milkv-duo-s.dts:116-133`).

### 3.2 What `osdrv` is

`interdrv/v2` is a Hi3516-style "MPP" driver set: every module is a misc or char device with a private ioctl
ABI, no V4L2 and no DRM anywhere. Buffers are identified by **physical address** in the uAPI; `VB_BLK` handles
returned to userspace are raw kernel pointers (`base/vb.c:495,1094`). Modules are coupled through exported
symbols and a callback bus (`include/common/kapi/base_cb.h`), giving this build order
(`KBUILD_EXTRA_SYMBOLS` in the module Makefiles):

```
sys -> base -> { vpss, cif, rgn, dwa, ive }
vo   <- base + vpss + sys          fb <- base + vpss
vi   <- base + sys                 vcodec -> jpeg -> cvi_vc_drv
rtos_cmdqu -> fast_image
```

| Module | Compiled LOC (approx.) | Kernel interface | Notes |
|---|---|---|---|
| sys | 1.5k | `/dev/cvi-sys`, 11 ioctls | ION wrapper, cache ops by physical address, bind graph |
| base | 5.8k | `/dev/cvi-base`, VB ioctls, mmap | VB pools, job queues, callback bus, **VIP_SYS register owner** |
| cif + snsr_i2c | 9.7k (3.4k register headers) | `/dev/cvi-mipi-rx`, 17 live ioctls | CSI-2 RX, no DMA |
| vi | 24k (+28k generated register headers) | `/dev/cvi-vi`, 53 + 38 sub-ids, mmap, poll | ISP; 3A in userspace |
| vpss | 23.4k | `/dev/cvi-vpss`, 49 ioctls | scalers **and the whole display HAL** (72 exports) |
| vo + mipi_tx + fb | 6.9k + 0.9k | `/dev/cvi-vo`, `/dev/cvi-mipi-tx`, `/dev/fb0` | own no registers (see 4.1) |
| vcodec + jpeg + cvi_vc_drv | 55.7k | 81 + 17 + 16 ioctls | ~70% Chips&Media host API and sample code inside the kernel |
| dwa, rgn, ive, tpu, mon, rtos_cmdqu, fast_image | 4k, 3.5k, 21k, 4.2k, 1.7k, 1.1k, 1.2k | misc/cdev | see section 6 |
| pwm, rtc, wdt, saradc, clock_cooling, wiegand | small | frameworks with vendor compatibles | mostly replaceable by mainline |

The snapshot is two years behind Sophgo's `sg200x-dev` branch (weekly releases continued through 2026-08-24
with VO/VPSS/VI fixes); any port should rebase first.

### 3.3 Vendor-kernel-only dependencies

Exactly ten vendor-kernel exports are used by `osdrv`:

| Vendor facility (5.10) | Used by | Mainline 7.3 state | Replacement |
|---|---|---|---|
| `cvi_ion_alloc/_nofd`, `cvi_ion_free/_nofd`, `ion_buf_begin/end_cpu_access`, `struct ion_buffer.paddr/vaddr/name` (vendor ION fork, carveout heap `cvitek,carveout` on reserved-memory `ion-region`) | `sys.c` (15 sites, the only allocator path), plus direct use in `tpu`, `fast_image`, `dwa/ldc_test.c`, `vcodec vdi.h`, `jpeg jdi.h` | ION removed in 5.11; no carveout heap; no in-kernel heap allocation API | CMA reserved pool + `dma_buf_export` inside `sys.c`, keeping the exported `sys_ion_*` signatures (48 external call sites unchanged); `MODULE_IMPORT_NS("DMA_BUF")` |
| `arch_sync_dma_for_device()` | `sys.c:378-410`, `fb/cvifb.c`, `tpu`, `vpss/scaler.c` (10 sites) | Defined but **not exported** (`arch/riscv/mm/dma-noncoherent.c`) | `dma_sync_single_for_device/for_cpu` on a `dma-noncoherent` platform device; note the vendor "invalidate" is clean+invalidate while mainline's `for_device(FROM_DEVICE)` only cleans |
| `cvi_efuse_read_buf/write/read_from_shadow` | `base.c` (sysfs debug attributes, UID), `saradc` | No Sophgo nvmem driver upstream (Armbian carries one) | Drop sysfs attributes or use nvmem |
| `sched_setscheduler()` export, `MAX_USER_RT_PRIO`, `CONFIG_SCHED_CVITEK` auto RT boost of `cvitask_*` threads | `vi`, `vo`, `vpss`, `dwa`, `cvi_venc` (14 sites) | Only `sched_set_fifo*` exported | `sched_set_fifo()`; re-measure frame-drop behaviour |
| `i2c_adapter.i2c_idx`, `I2C_M_WRSTOP` | `snsr_i2c` | Absent | `adap->nr`, `I2C_M_STOP` (DesignWare splits transfers at STOP) |
| Vendor headers `pinctrl-cv181x.h` (direct FMUX writes), `tee_cv_private.h`, `streamline_annotate.h`, `cv180x_efuse.h`, ION headers via `-I$(srctree)/drivers/{staging/android,tee,pinctrl/cvitek}` | `cif`, `vo`, `base`, `tpu`, several | Absent | Remove; convert pinmux to DT pinctrl states |
| vermagic/modversions ignored under `CONFIG_ARCH_CVITEK` | module loading | Enforced | Rebuild per kernel |

The **VIP_SYS** block at 0x0A0C8000 (inferred from `cif.c:2341` and `scaler.c:4632`; the vendor DTS in Appendix A
gives the authoritative value) holds the per-block VIP resets (isp_top, img_d/v, sc_*, disp, bt, dsi_mac,
csi_mac0-2, ldc, dsi_phy, csi_phy0, csi_be, ive), clock gates, dividers and AXI real-time/offline switches.
`base.ko` maps it and exports raw accessors (`vip_toggle_reset`, `vip_sys_reg_write_mask`), used from `vi`,
`vpss`, `cif`, `dwa`, `ive`. Mainline has no provider for this block (the clock binding allows only the
0x03002000 range). Any strategy needs either `base.ko` as owner or a new syscon + reset/clock-gate provider.

Direct pokes into blocks that mainline drivers now own must be removed: 38 `ioremap(0x03002xxx)` sites in
`cif.c` (cam PLL power/dividers, only compiled when `CONFIG_COMMON_CLK_CVITEK` is unset, which is the case on
mainline), `0x03002840` DISPPLL bit in `scaler.c:3615`, FMUX writes to 0x03001000 (52 in `vo.c`, 19 in `cif.c`,
parallel/BT/TTL modes only), `0x08004544` DDR controller patch, `0x03000220/0x03005144` VBAT registers.

### 3.4 5.10 to 7.3 API breakage (compile-only view)

Full inventory in the api-delta report (44 rows). The items that matter:

| Class | Sites | Severity | Fix |
|---|---|---|---|
| ION allocator + physical-address ABI | 15 direct + 119 wrapper lines in 11 modules | blocker (design) | shim in `sys.c`, 2-4 weeks |
| `arch_sync_dma_for_device` not exported | 10 | high | centralise in `sys_cache_*`, 0.5-1 week |
| `strncpy()` **removed from the kernel** (`Documentation/process/deprecated.rst:134-137`) | 171 | high (volume) | per-site `strscpy`/`strscpy_pad`/`strtomem_pad`, 1-2 weeks |
| `-Wall -Wextra -Werror` in 18 Makefiles vs stricter 7.3 default warnings (`-Wmissing-prototypes` etc.) | up to ~2000 non-static functions | high (volume) | drop `-Werror` for bring-up; restoring it 2-4 weeks |
| `platform_driver.remove` returns `void` | 26 | trivial | |
| `class_create(THIS_MODULE, ...)` | 13 (+2 extdrv) | trivial | |
| `<linux/of_gpio.h>` removed; legacy integer GPIO API deprecated; GPIO numbers arrive from userspace in `vo`/`vi` | 4 lookups + ~20 legacy calls | moderate | gpiod from DT |
| `MODULE_IMPORT_NS("DMA_BUF")` required | 0 present | trivial | |
| `sched_setscheduler`/`MAX_USER_RT_PRIO`, timers (`del_timer_sync`, `from_timer`), `vm_flags_set`, `DEFINE_SEMAPHORE`, thermal 5-arg register, `FBINFO_DEFAULT`, `PDE_DATA`, `compat_ptr_ioctl` redefinition (COMPAT is now on for rv64) | ~60 | trivial | |
| PWM chip API rewritten | 1 module | moderate | drop module; mainline `pwm-sophgo-sg2042.c` has an identical register map (needs a cv18xx compatible, Armbian carries the patch) |
| In-kernel firmware file reads (`filp_open` of `/usr/share/fw_vcodec/*.bin`) | codec | moderate | `request_firmware()` |

Compile-only estimate for **all** of `interdrv/v2` against 7.3 with `-Werror` removed: **8-15 engineer-weeks**
(no hardware work). `extdrv/wireless` (1751 files of third-party WiFi drivers) is out of scope; use in-tree drivers.

---

## 4. Display path

### 4.1 Hardware and vendor driver structure (verified)

- **DISP** timing generator and 3-plane DMA read engine at SCL_TOP + 0x8000 (0x0A088000): shadowed registers with
  a force-update bit (clean double-buffered mode sets), input/output CSC, 65-node gamma LUT, four solid "cover"
  windows (`vpss/chip/cv181x/scaler.c:2249-3340`).
- **DISP GOP** overlay: 2 layers x 8 windows, ARGB8888/4444/1555 and 8/4-bit LUT formats, colour key, optional 2x
  upscale (`scaler.c:2688-2777`). Layer 0 is used by RGN overlays, layer 1 window 0 by the vendor framebuffer.
- **Output mux**: DSI, BT.601/656/1120, LVDS, I80/MCU, parallel RGB (`scaler.c:3601-3611`). The I80 path is
  incomplete in the vendor code.
- **MIPI DSI MAC**: CVITEK-specific (not DesignWare), small register set at SCL_TOP + 0xA000, **video burst mode
  only**, 1/2/4 lanes, ECC and CRC16 computed in software, 16-byte LPDT FIFO, 4-byte reads
  (`scaler.c:3714-4126`; `vo/chip/cv181x/vo_mipi_tx.c:95-108` rejects other modes).
- **D-PHY** at 0x0A0D1000: lane role remap and P/N swap, in-driver PLL math, HS timing, LVDS mode; its PLL code
  also writes a VIP_SYS clock-control bit (`vpss/chip/cv181x/dsi_phy.c:118-135,237-334`).
- Single device/layer/channel; interlaced rejected; rotation goes through the DWA engine.
- **`vo`, `mipi_tx` and `fb` own no registers and no interrupt.** All register code is in `vpss` and exported
  (59 `sclr_*` + 13 `dphy_*` symbols). `vpss` owns the ioremaps and the shared `sc` IRQ, whose single status
  register mixes scaler and display-vblank bits; display IRQs reach `vo` through a callback
  (`vpss_core.c:1332-1363,1421-1442`; `vo_core.c:211-220` is `#if 0`). The display-enable bit shares a register
  and spinlock with the scaler enables (`scaler.c:708-745`).
- **Panel initialisation is entirely userspace**: `/dev/cvi-mipi-tx` ioctls carry lane map, timing, pixel clock
  and raw DCS packets; the kernel holds no panel tables (`vo_mipi_tx.c:148-276`). Frames reach DISP as physical
  addresses of VB blocks (`vo_sdk_layer.c:374-420`). `cvifb` is a thin fbdev over GOP layer 1 backed by a
  reserved-memory region and uses the non-exported `arch_sync_dma_for_device` and the removed `FBINFO_DEFAULT`.
- Public documentation exists: the SG2000 TRM documents VDP DISP (0x0A088000), OSD (0x0A088800), MIPI TX
  control (0x0A08A000) and PHY (0x0A0D1000) at register level (`sophgo-doc/SG200X/TRM/contents/en/video/`).

### 4.2 Mapping to mainline

The display engine maps cleanly onto DRM/KMS: one CRTC (DISP timing generator, vblank from the `disp_frame_end`
bit, atomic flush via the shadow force-update bit), a primary plane (planar/semi-planar YUV, packed YUV, RGB888),
2-4 overlay planes (GOP windows), a DSI encoder implementing `mipi_dsi_host_ops.transfer` exactly like
`sun6i_mipi_dsi.c` (which also computes ECC/CRC in software), the D-PHY as a `drivers/phy` driver, and
`drm_gem_dma` + `drm_fbdev_dma` for `/dev/fb0`. Panels come from the 106 upstream DSI panel drivers
(`hx8394` and `ili9881c`, which the vendor timing tables reference, already exist); HDMI via the upstream LT8912B
or LT9611 bridges. Closest references: `ingenic-drm-drv.c` (1.7k lines, planes), `sun6i_mipi_dsi.c` (1.3k),
`sprd_dsi.c` (1.1k, in-driver PHY PLL), `mxsfb` (2.3k). Expected driver size 2-3.5k lines.

Structural problems to solve before upstreaming:
1. SCL_TOP registers and the `sc` IRQ are shared with the scaler: needs a syscon regmap (or a parent MFD) and an
   IRQ-status demux or a single owner.
2. The DISPPLL bit at 0x03002840 and the VIP_SYS clock bits must become clk framework calls or a VIP_SYS syscon.
3. The "disp_from_sc" online path (scaler feeding DISP without DRAM) has no DRM analogue; drop it.
4. fbdev is deprecated for new drivers; `drm_fbdev_dma` provides `/dev/fb0` but in XRGB8888, not the vendor
   default 16-bit ARGB4444 or 8-bit LUT modes.
5. Boot-logo handoff ("smooth" mode reading back bootloader state) is not handled unless implemented.

### 4.3 Options and effort

| Option | Scope | Effort (engineer-weeks) | Upstreamable | Keeps vendor VO apps |
|---|---|---|---|---|
| D1 Forward-port `vo`+`mipi_tx`+`fb` on top of ported `sys`/`base`/`vpss` | fix ~12 API classes, pinmux to DT or copied header, DTS nodes | 3-6 (vpss reader) to 6-12 (vo reader) **after** the foundation is ported | no | yes |
| D2 Minimal DRM/KMS: 1 CRTC, primary plane, DSI host, D-PHY phy driver, fbdev emulation, bindings | DSI video-mode panel or bridge only | 8-14 to a reviewed driver; first picture 3-6; +3-6 for upstream review rounds | yes | no |
| D3 Full DRM: D2 + GOP overlay planes, gamma, BT.601/656/1120 and LVDS encoders, VBAT handling | all vendor outputs except I80 | 18-30 incl. review | yes | no |
| D4 "Ugly then clean": copy `scaler.c:2249-4300` + `dsi_phy.c` verbatim into a thin DRM driver first, refactor later | fastest first light | 4-8 to first picture, converging into D2's total | eventually | no |

Assumptions: one engineer experienced in DRM atomic and DSI; a board with free DSI pads and a known panel or an
LT8912B/LT9611 bridge; no documentation beyond TRM + vendor code; a logic analyser for D-PHY debugging.

### 4.4 Display risks and unknowns

- D-PHY PLL and HS-timing constants are reverse-engineered from vendor code; per-panel tuning may be needed.
- Register 0x03002840 bit 1 and the VIP_SYS CLK_CTRL0 bits written by `dsi_phy.c` are not explained by any
  source read; bit 3 is modelled in mainline as the DISPPLL synthesizer enable (`clk-cv1800.c:124-126`).
- Whether the DSI MAC can do non-burst or command mode is unknown (restriction may be software only).
- Sophgo's CV186x DRM driver (`linux-common` 6.12.y, `drivers/gpu/drm/cvitek`, 5.3k lines) shares no register
  macro names with the CV181x headers; it is a structural template only, each write must be re-validated.

---

## 5. Camera path

### 5.1 CIF (MIPI CSI-2 receiver) and `snsr_i2c` (verified)

- Pure bridge with **no DMA**: zero address/stride fields in its register maps; data goes on-chip to the ISP's
  CSI bridge, which `vi` handles (`cif/chip/cv181x/drv/inc/reg_fields_csi_*.h`; `vi/chip/cv181x/vip/vi_drv.c:281-303`).
  Its two IRQs only count ECC/CRC/WC/HDR/FIFO errors.
- 3 MACs, 2 MIPI-capable (MAC0 on a 4-lane PHY, MAC1 on a 2-lane PHY); 6 physical D-PHY lanes freely routable
  to logical CLK/D0-D3 with P/N swap; sub-LVDS, HiSPi, BT.601/656/1120, TTL inputs; VC/DT/DOL/manual HDR
  (`cif_drv.c:1263-1363`; `cif.c:456-581`).
- Userspace pushes the entire receiver configuration (lane map, hs_settle, HDR mode, data types) in one ioctl and
  streaming starts immediately (`cif.c:464-640`). Sensor init over I2C is done by userspace (likely via
  `/dev/i2c-N`; the middleware is not in the repos, so this is unverified).
- `snsr_i2c` exists only to fire ISP-computed exposure/gain register groups synchronously with a target frame,
  optionally as one burst transfer with inter-message STOPs; it depends on two vendor I2C core patches.
- Uses the reset framework (`phy0`, `phy-apb0`, `phy1`, `phy-apb1` = vendor IDs 70-73, usable as raw numbers on
  the mainline reset controller), 6 clocks (all with mainline IDs), sensor reset GPIOs, plus VIP_SYS MAC dividers
  and direct PLL pokes.
- Mainline fit: a V4L2 sub-device with media-controller and fwnode endpoints, like `rkisp1-csi.c` (518 lines),
  `cdns-csi2rx.c` (1.1k), `sun6i-mipi-csi2` (0.8k), with the PHY wrapper as a `drivers/phy` driver; `snsr_i2c`
  becomes unnecessary because sensors are in-kernel V4L2 sub-devices. Sub-LVDS/HiSPi have no V4L2 bus type.
  Common CVITEK/Milk-V sensors (gc2053, gc2083, gc4653, SmartSens sc*, os04a10) have **no** mainline driver;
  imx219/imx290/imx335/imx415/ov5647/ov5640 do.
- The TRM documents MIPI RX (D-PHY 0x0A0D0000, CSI 0x0A0C2400/0x0A0C4400) and VI top at register level.

### 5.2 VI / ISP (verified)

- Pipeline: 3 pre-raw front-ends (FE0 4 ch, FE1/FE2 2 ch) with CSI bridges, one pre-raw back-end, then
  RAWTOP/RGBTOP/YUVTOP post stage; 143 register blocks in a 512 KiB window at 0x0A000000; default single-sensor
  flow is FE -> BE -> DRAM -> post (multi-pass, memory-to-memory-like), with an optional online handshake into
  the VPSS scaler (`include/chip/cv181x/uapi/linux/isp_reg.h`; `vi.c` scene control).
- Single `isp` IRQ, a hi-tasklet and four SCHED_FIFO kthreads.
- **All 3A and tuning run in userspace.** The kernel only DMAs statistics (AF, GMS, AE histograms, AWB, DCI,
  edge histogram, motion map) into memblocks whose physical addresses it publishes, and applies double-buffered
  tuning nodes written by userspace. The three `VI_IOCTL_AE_CFG/AWB_CFG/AF_CFG` ids have no handler
  (`vi.c:5004-5040,5389-5460,7172-7190`). Tuning nodes (post 47 KB, BE 16.6 KB, FE 104 B per node) live in
  kzalloc'd memory exposed to userspace by **physical address plus kernel virtual pointers**
  (`vi_tun_ip_ctrl.c:246-298`, `vi.c:5627-5643`), which requires `/dev/mem` and breaks under
  `STRICT_DEVMEM`.
- The ISP working pool is allocated by userspace from ION and handed over as paddr/size
  (`vi.c:5322-5340`); output frames come from the VB pool. VI's own ION use is a 128 KiB CMDQ buffer per pipe.
- Sensor drivers are userspace libraries; exposure/gain register lists are queued per frame and fired by the
  kernel through `snsr_i2c`.
- LOC split: register programming 9.0k lines in `vip/*_ip_ctrl.c` with essentially no kernel API use plus 28.3k
  lines of generated register headers (portable as-is); kernel glue 11.8k lines and 3.1k uAPI header lines are
  what a mainline rewrite replaces. The 10.6k-line `vi_vreg_blocks.h` shadow-register image is dead code on CV181x.
- Mainline fit: a V4L2 media-controller ISP driver with params (`META_OUTPUT`) and stats (`META_CAPTURE`)
  queues using the generic `v4l2-isp.h` framing. The vendor design (DMA'd stats, double-buffered per-block
  tuning with update flags) maps directly onto that model. References: rkisp1 (12.3k lines), mali-c55 (5.5k),
  c3-isp (4.1k), pisp_be (1.8k, memory-to-memory scheduling). libcamera has **no** Sophgo pipeline handler, and
  the vendor 3A is closed, so a native stack needs new AE/AWB/AF and tuning. The ISP registers are documented
  only by GPL-2.0 `osdrv` headers (not by the TRM), so an in-kernel ISP driver would be a GPL derivative, not a
  clean-room design.

### 5.3 VPSS (verified)

- IMG_IN x2 (display and video paths) + 4 scaler cores (SC_D, SC_V1-3; one read DMA shared by three outputs) +
  4 write DMAs + per-scaler GOP, privacy mask, border, slice-buffer handshake to the encoder; online (ISP -> VPSS
  without DRAM) and offline modes; 16 groups x 3-4 channels software model with a kthread scheduler; 96 call sites
  into the base/sys VB framework. It also hosts the display HAL (section 4).
- Mainline fit: offline scaling as a V4L2 mem2mem device modelled on Rockchip RGA (2.3k lines) or sun8i-rotate,
  with the 1-in/3-out hardware forced into 1:1 jobs; online mode only makes sense as resizer sub-devices inside
  an ISP media graph. GOP/privacy/slice-buffer features have no V4L2 equivalent.

### 5.4 Options and effort

| Option | Scope | Effort (engineer-weeks) | Upstreamable | Keeps vendor camera apps and tuning |
|---|---|---|---|---|
| C1 Forward-port `cif`+`snsr_i2c`+`vi`+`vpss` on top of ported `sys`/`base` | API fixes, CCF-only clock path, pinctrl states or MIPI-only, DTS nodes, tuning buffers moved from `/dev/mem` to an mmap offset or dma-buf, ISP pool via CMA/dma-buf | cif 2-4, vi 6-10, vpss 3-6 (plus foundation, section 3) | no | yes (ISP pool allocation path in the middleware needs an ION-compatible path or a small userspace change) |
| C2 V4L2 CSI-2 RX sub-device + D-PHY driver + VI raw/YUV capture (no ISP processing), usable with libcamera "simple" | bindings, DT, one supported sensor | 6-12 (cif 4-8 + capture node) | yes | no |
| C3 Native V4L2 ISP driver + VPSS as m2m/resizers | reuse `vip/*_ip_ctrl.c` behind a rkisp1/pisp_be-style shell; params/stats via `v4l2-isp.h`; sensors as sub-devices | 20-35 kernel (+4-8 VPSS m2m, +1-3 per sensor driver) | yes (long review) | no |
| C4 libcamera pipeline handler + IPA with new 3A and tuning | required for usable images with C3 | 15-30 (readers), up to 20-40 (ecosystem reader) | yes | no |
| C5 Staged: C1 first, then wrap VI capture/stats/params in V4L2 nodes carrying the vendor structs | transition path | C1 + 10-16 | no (vendor structs in META formats) | partially |

Assumptions: single RGB sensor, offline post-to-DRAM path, no HDR/3DNR parity, no FreeRTOS fast-boot, no
ISP -> VPSS online mode for the first milestone; a board with a sensor that has a mainline driver for the V4L2
options.

### 5.5 Camera risks and unknowns

- Coherency: the middleware flushes/invalidates by physical address through vendor ION ioctls; on the
  `dma-noncoherent` C906 any mismatch yields stale statistics or corrupted frames. The kernel used must enable
  `CONFIG_ERRATA_THEAD` (not in the stock riscv defconfig).
- hs_settle, deskew phases and clock-lane direction rules exist only as heuristics in `cif.c`; wrong values give
  silent link failures.
- Loss of the vendor kernel's automatic RT-priority boost for `cvitask_*` threads may change frame-drop behaviour.
- The real middleware's dependence on `/dev/ion`, `/dev/mem` and on `VB_BLK` being a stable kernel pointer is
  unverified (the middleware is not in the repositories).
- No open 3A exists for this ISP; image quality of a native stack is an open research item, not an engineering task.

---

## 6. Other modules

| Module | Verdict | Reason / mainline home |
|---|---|---|
| vcodec + jpeg + cvi_vc_drv | Defer; forward-port 6-12 weeks, native V4L2 stateful codec 30-55 weeks | Chips&Media CODA980 (H.264, 0x9800) + WAVE420L (HEVC, 0x4201) + CODAJ12-class JPEG. Mainline `wave5` supports only WAVE5xx; `coda` supports CODA960 not 980; no CODAJ12 driver. Firmware is read by the kernel from `/usr/share/fw_vcodec`; redistribution terms unclear. |
| tpu | **Already ported** to mainline 7.2/7.3 by a Cvitek engineer in Armbian (ION -> dma-buf import from the CMA heap, paddr ioctl, `dma_sync_sgtable_*`, binary-compatible uAPI) | `armbian/build` patch `0064-drivers-soc-sophgo-add-CV181x-SG200x-TPU-driver.patch`; the proven ION-removal recipe for this plan |
| dwa (LDC/dewarp) | Forward-port 2-3 weeks or V4L2 m2m later (NXP dw100, 1.7k lines, as reference) | depends on base VIP_SYS helpers and VPSS callbacks |
| rgn | Follows VPSS/VO; software-only OSD manager over ION canvases | would become DRM planes / V4L2 controls |
| ive | Forward-port 2-4 weeks; no mainline framework | custom misc or accel driver |
| rtos_cmdqu + fast_image | Replace transport with mainline `cv1800-mailbox.c` (identical register map) + a small hwspinlock; drop fast_image unless FreeRTOS fast-boot ISP is required | Armbian carries C906L remoteproc + mailbox DTS |
| rtc, saradc, wdt, pwm | **Drop**: mainline `rtc-cv1800`, `sophgo-cv1800b-adc`, `dw_wdt` (needs DTS node), `pwm-sophgo-sg2042` (needs cv18xx compatible; Armbian patch exists) | check SARADC eFuse trim and RTC power-on features if needed |
| mon, clock_cooling | Drop (debug profiler; no upstream thermal zone for cv18xx yet, Armbian carries a thermal driver) | |
| wiegand, extdrv tp/gyro | trivial fixes | |
| extdrv/wireless | Out of scope; use in-tree brcmfmac/mt76/rtw drivers | |

---

## 7. Prior art and ecosystem (as of 2026-10-06)

| Source | What it tells us | Access / confidence |
|---|---|---|
| `sophgo/linux` wiki, "Peripherals Status" (revision 2026-09-02) | DRM, Media and TPU rows: **Not Started**. Base peripherals upstream (clk 6.10, pinctrl 6.12, reset 6.17, mailbox 6.16, RTC 6.16, ...). Under review: eFuse, timer, watchdog, PWM (v8), thermal (v5), remoteproc C906L (v2), I2S (v4). Duo S minimal DTS in 7.3. | cloned wiki repo; verified |
| Public patch series for CV18xx camera/display | None found. | lore/patchwork unreachable; search snippets only; likely |
| Armbian `sophgo-sg200x` family | Duo S on mainline 7.2 ("edge") and 7.3 ("bleedingedge") with ~60 patches: thermal, MDIO mux, timer, watchdog, PWM, C906L remoteproc + mailbox DTS, eFuse nvmem, I2S, TPU driver, 128 MiB CMA pool. No video drivers. Natural integration base for a port. | cloned repo; verified |
| Armbian TPU port (PR 10760, patch 0064, author Wellken Chen, Cvitek) | Vendor misc driver re-hosted on mainline: ION -> dma-buf import from CMA heap, `GET_DMABUF_PADDR`/`RELEASE_DMABUF` ioctls, `dma_sync_sgtable_*`, TEE dropped, uAPI binary-compatible so `libcviruntime` works with a small userspace allocator change. Merged in `armbian/build` main. | patch files read; verified |
| NixVegas/BadgeOS (mainline on Duo Module 01) | Documents that VO/DSI, ISP, codec, TPU were the only vendor-5.10-only subsystems; wants a minimal DRM/KMS driver for DISP + DSI host to drive an LT8912B HDMI bridge (detected over I2C); no mainline display output achieved. | repo cloned; issue text from search snippet; likely |
| scpcom / Fishwaldo Debian images | Vendor 5.10 + osdrv; DSI panels, SPI LCD, LT9611 DSI -> HDMI (part of panel init done in the bootloader); cameras via osdrv. Contains the vendor SoC/board DTS used in Appendix A. | repo cloned; verified |
| Sophgo `linux-common` 6.12.y (BM1688/CV186 line) | GPL DRM/KMS driver `drivers/gpu/drm/cvitek` (disp, dsi, lvds, dw_hdmi, MIPI PLL; 5.3k lines) for the sibling CV186x display engine. Structural template; register names do not match CV181x. No vendor V4L2 camera driver in that tree. | repo cloned; verified |
| Sophgo `osdrv` `bm1688` branch | Same driver family adapted to 6.x (31 `KERNEL_VERSION(6,0,0)` guards, up to 6.12), still ION-based. Catalogue of API fixes Sophgo already made. | repo cloned; verified |
| Sophgo SDK manifests (`sophpi`) | SG200x SDKs stay on `linux_5.10`; osdrv weekly releases continued through 2026-08-24. Only BM1688/CV186 uses 6.12. | verified |
| SG2000 TRM (`sophgo-doc`) | Register-level docs for VI top, VDP DISP/OSD, MIPI RX, MIPI TX. **Not** for ISP, VPSS, codec, JPEG, TPU, IVE, DWA. | verified |
| libcamera (2026-09-18) | No Sophgo/Cvitek pipeline handler. | mirror cloned; verified |

---

## 8. Strategies compared

_(filled from the Phase 2 design and judging pass)_

## 9. Recommended plan

_(filled from the Phase 2 design and judging pass)_

## 10. Risk register

_(filled from the Phase 2 design and judging pass)_

## 11. Open questions to resolve before committing

_(filled from the Phase 2 design and judging pass)_

## Appendix A: device-tree contract

_(filled from the Phase 2 DT extraction)_

## Appendix B: evidence index

Phase 1 reader reports (file:line evidence for every statement above) are archived with the assessment:
`sys-base`, `cif-snsr`, `vi-isp`, `vpss`, `vo-mipitx-fb`, `codec-others`, `api-delta`, `vendor-deps`,
`mainline-infra`, `ecosystem`.
