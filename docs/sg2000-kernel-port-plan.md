# SG2000 (CV181x) multimedia drivers on a modern kernel: feasibility assessment and porting plan

Status: assessment, 2026-10-06. Inputs: this `osdrv` snapshot (branch `sg200x-dev`, last commit 2024-07-31),
the vendor Linux 5.10.4 tree (`sg2000_linux_5.10`), mainline Linux 7.3-rc6 (`mainline_linux`, 2026-10-06),
and the public sources listed in section 7. Evidence is cited as `path:line` in the repository named by
context. Effort figures are engineer-week ranges with stated assumptions; they are estimates, not measurements.
Nothing in this document was compiled or run on hardware. Statements that could not be verified are marked.
Assumption: Linux runs on the SG2000's RISC-V C906 core (riscv64); all cache-coherency, errata, Kconfig and
toolchain statements are for that core. Running Linux on the SG2000's Arm Cortex-A53 was not assessed (the Duo
Module 01 EVB mainline device tree is arm64-only).

The assessment was produced in three passes: ten independent code readers (per subsystem), three adversarial
verifiers on the load-bearing claims, three independent strategy planners and two judges. The verifiers'
corrections are folded into the text; the raw reports are archived in `docs/port-assessment-evidence/`.

---

## 1. Bottom line

**Porting is feasible. Display should be done as a new DRM/KMS driver rather than a port of the vendor
display modules. The camera can be ported quickly only if the vendor middleware is kept; a native V4L2 and
libcamera camera is a multi-quarter programme.**

| Question | Answer | Confidence |
|---|---|---|
| Can the display path (VO, MIPI DSI TX, framebuffer) work on mainline 7.x? | Yes. Fact: the display register code is a separable (not isolated) layer inside `vpss`, about 2.5k lines that share one register page, one interrupt line, 13 exports and three static tables with the scaler (4.1); the display blocks are documented at register level in the public SG2000 TRM. Interpretation: a DRM/KMS driver is upstreamable, the vendor fbdev/ioctl stack is not. Recommendation: build a **new DRM/KMS driver** reusing the vendor register sequences: first picture 3-8 weeks, reviewed minimal driver (DSI, one plane) 9-17 weeks including the shared-register groundwork (the recommended Stage 2 spends 12-26 because it adds the register-map study, overlay planes and gamma), full-featured 18-30 weeks. The vendor `vo`/`fb` modules (fbdev, ioctl-driven, panel init in userspace) can also be forward-ported in 3-12 weeks but are not upstreamable. | medium-high |
| Can the camera path (CSI-2 RX, ISP, scaler) work on mainline 7.x? | Fact: the kernel contains no 3A; all image intelligence is in the vendor userspace (5.2). Recommendation: **yes, as a forward-port that keeps the vendor middleware** (`cif`, `snsr_i2c`, `vi`, `vpss` on a ported `sys`/`base`): about 21-39 weeks to a validated 1080p30 stream including the shared infrastructure. Hard conditions: `/dev/mem` must stay available (the middleware maps every buffer through it), the ION allocator is replaced by a CMA/dma-buf exporter behind the existing kernel API, and cache semantics must be validated on the non-coherent C906. **A native V4L2 + libcamera camera** is feasible but large: CSI-2 receiver and raw capture 8-16 weeks, ISP driver 20-35 weeks (a second reader estimated 10-20; the higher figure is used, see 5.4), 3A/IPA 4-30 weeks depending on how much of the vendor 3A may be reused (licence question), plus VPSS scaler 4-8 and sensor drivers 0-8: 36-97 in total (9.2, Stage 3). | forward-port: medium; native: medium-low |
| Is the vendor camera intelligence available? | Partly. The ISP middleware framework source is public (`github.com/sophgo/cvi_mpi`), but the AE/AWB/AF cores and the tuning-interpolation library ship only as prebuilt objects, with no licence file and "All rights reserved" headers. The kernel contains no 3A at all. | high |
| Does mainline already have the SoC plumbing? | Yes. Clocks (every vendor clock name has a mainline ID), top-level resets, pinctrl, Duo S device tree, DMA, I2C/SPI/UART, USB, SD/eMMC, Ethernet, RTC, ADC, mailbox and audio are upstream. Nothing exists for VI/ISP, CSI-RX, VPSS, VO/DSI, codec, JPEG, TPU, IVE, DWA, nor for the VIP_SYS sub-controller. | high |
| Is anyone else doing this? | No upstream or public RFC work for CV18xx display or camera was found (Sophgo's own upstream status wiki lists DRM, Media and TPU as "Not Started" as of 2026-09-02). Armbian runs the Duo S on mainline 7.2/7.3 with ~60 patches including a Cvitek-authored TPU port. Sophgo wrote a GPL DRM driver for the sibling CV186x on a 6.12 vendor kernel, usable as a template. | medium (mailing-list archives were not reachable) |
| Biggest technical blockers? | (1) The Android ION allocator and physical-address buffer ABI used by every module; the ION API calls are confined to `sys.c` (plus `tpu`), so the allocator shim is bounded work (3-6 weeks), but ION headers leak through `sys.h` into nine modules and 12 Makefiles and must be cleaned up in the compile pass. (2) The VIP_SYS register block (per-block resets, clock gates, dividers) has no mainline provider and is shared by display and camera. (3) The display and scaler share one 4 KiB register page and one interrupt line. | high |
| Biggest non-technical blocker? | Licensing: the vendor 3A cores are binary-only without a licence; the ISP and VPSS registers are documented only by `osdrv` headers that carry "All rights reserved" and no SPDX tag. Any native ISP work and any reuse of the 3A binaries need Legal review. | high |

Recommended approach (section 9): a staged hybrid. First settle the product question (vendor SDK on a modern
kernel, or standard Linux APIs), then (0) a compile-only forward-port of the nine display/camera modules to
measure real breakage (3-6 weeks), (1) a vendor-ABI camera port with the middleware retained (18-33 weeks),
(2) a new DRM/KMS display driver developed in parallel from the foundation milestone onward (12-26 weeks),
and (3) an optional, separately gated native V4L2/libcamera camera. Running code answering "feasible for both"
(camera at G1 plus display first picture at G2) costs roughly 28-54 engineer-weeks; with two engineers working in
parallel the calendar time is close to the camera chain alone (21-39 weeks). A maintainable result (G3) is 33-65
engineer-weeks.

---

## Terms

- **MPP**: HiSilicon-style media-processing driver model (misc devices plus private ioctls) that the vendor code follows.
- **VIP / VIP_SYS**: the vendor's video subsystem (camera input, ISP, scalers, display, dewarp, vision engine) and
  its shared control register block at 0x0A0C8000 (resets, clock gates, dividers).
- **VB**: the vendor's video-buffer pool framework (in the `base` module) on top of the Android ION allocator.
- **GOP**: the vendor's graphics overlay engine (2 layers x 8 windows) inside the display controller and each scaler.
- **SC_TOP / SCL_TOP**: the scaler-top register page at 0x0A080000 that also holds the display enable, output mux and
  the shared interrupt registers. **SC / SCL**: scaler. **DISP / VDP**: the display controller (TRM name VDP).
- **CIF**: vendor name for the camera interface (CSI-2 receiver, sub-LVDS, parallel inputs). **VI**: vendor name for the
  camera input and ISP driver. **VPSS**: vendor name for the scaler subsystem. **VO**: vendor name for video output.
- **FE / BE**: ISP pre-raw front end / back end. **3A**: auto exposure, auto white balance, auto focus.
- **HAL**: hardware abstraction (register programming) layer. **TRM**: Technical Reference Manual.
- **CMA**: Linux Contiguous Memory Allocator; **dma-heap**: its userspace dma-buf interface. **CCF**: common clock
  framework. **m2m**: V4L2 memory-to-memory device. **IPA**: libcamera image-processing-algorithm module.
  **PRIME**: DRM buffer sharing through dma-buf file descriptors. **igt**: the IGT GPU test suite.
- **FMUX**: pin-function multiplexer registers at 0x03001000. **W1C**: write-1-to-clear interrupt status semantics.
- **DOL / HiSPi**: Sony digital-overlap HDR and Aptina high-speed serial sensor interfaces.
- **ION**: the Android memory allocator removed from mainline in 5.11; the vendor kernel carries a fork of it.

---

## 2. What was examined and how

- All 24 module directories of `interdrv/v2` (170 `.c`, 340 `.h`) were read, with deep reads of `sys`, `base`,
  `cif`, `snsr_i2c`, `vi`, `vpss`, `vo`, `fb`, and a structural survey of `vcodec`/`jpeg`/`cvi_vc_drv`, `rgn`,
  `dwa`, `ive`, `tpu`, `mon`, `rtos_cmdqu`, `fast_image`, `pwm`, `rtc`, `wdt`, `saradc`, `clock_cooling`, `wiegand`.
- Every kernel symbol or header the modules take from the vendor 5.10 kernel was traced to its vendor source and
  checked against mainline 7.3.
- Mainline 7.3 was surveyed for SG2000 support (DTS, clk, reset, pinctrl, dma-buf heaps, V4L2, DRM, codec,
  accel, mailbox) and for reference drivers of comparable hardware.
- Public ecosystem sources were cloned and read (section 7). Web pages on github.com, milkv.io and
  lore.kernel.org were not reachable from this environment; `git clone` was. Claims that rest only on search
  snippets are marked.
- Board device trees are **not** in either kernel repository (they live in the SDK `build` repository). The
  vendor SoC and board DTS were obtained from a community Debian build repository and are used in Appendix A.
- The vendor middleware (`cvi_mpi`, branch `sg200x-dev`, HEAD 693a72a1 of 2026-08-24) was cloned in the
  verification pass to settle how userspace allocates and maps buffers.

---

## 3. Baseline facts

### 3.1 Mainline 7.3 state for SG2000

| Area | Mainline 7.3 | Evidence (`mainline_linux`) |
|---|---|---|
| SoC DTS | `cv180x.dtsi`, `cv181x.dtsi`, `sg2000.dtsi`, `sg2000-milkv-duo-s.dts`, `sg2002-*.dts`; `soc { dma-noncoherent; }` | `arch/riscv/boot/dts/sophgo/sg2000.dtsi:21` |
| Clocks | Full VIP/VC tree: `CLK_DISP_VIP`, `CLK_DSI_MAC_VIP`, `CLK_DSI_ESC`, `CLK_ISP_TOP_VIP`, `CLK_CSI_*`, `CLK_SC_*`, `CLK_IMG_*`, `CLK_DWA_VIP`, `CLK_IVE_VIP`, `CLK_RAW_VIP`, `CLK_H264C/H265C/JPEG`, `CLK_TPU`, PLLs | `drivers/clk/sophgo/clk-cv1800.c:627-850`, `include/dt-bindings/clock/sophgo,cv1800.h` |
| Resets | `reset-simple` for `sophgo,cv1800b-reset` (4 KiB block, indices up to 32767 accepted); macros for VIPSYS, VCSYS, H264C, H265C, JPEG, TPU; none for CSIPHY/DSIPHY (68-73), raw numbers work | `drivers/reset/reset-simple.c:154,183`; `arch/riscv/boot/dts/sophgo/cv18xx-reset.h` |
| Pinctrl | `pinctrl-sg2000.c` with all MIPI TX/RX and VIVO pads | `drivers/pinctrl/sophgo/pinctrl-sg2000.c:198-228,401-408` |
| Non-coherent DMA | T-Head CMO errata ops for C906 | `arch/riscv/errata/thead/errata.c:74-125`; `ERRATA_THEAD` has no default and is **not** in `arch/riscv/configs/defconfig` (Armbian enables it) |
| `/dev/mem` policy | riscv selects `GENERIC_LIB_DEVMEM_IS_ALLOWED`, so `STRICT_DEVMEM` is selectable; default off; unset in the Armbian configs | `arch/riscv/Kconfig:125`; `lib/Kconfig.debug:1919-1923` |
| Memory | dma-buf heaps `system` and `cma` only; one heap per `shared-dma-pool` + `reusable` reserved-memory node; **no in-kernel heap allocation API** | `drivers/dma-buf/heaps/cma_heap.c:397-421`; `include/linux/dma-heap.h:43-47` |
| Peripherals | DMA mux, AXI DMA, I2C/SPI/UART/GPIO (DesignWare), SDHCI, dwmac, USB PHY and dwc2, RTC, SARADC, mailbox, I2S/codec | `cv180x.dtsi`; `drivers/mailbox/cv1800-mailbox.c`; `sound/soc/sophgo/` |
| Generic ISP uAPI | Versioned block-based params/stats framing shared by rkisp1, rcar-isp, rppx1 | `include/uapi/linux/media/v4l2-isp.h`, `Documentation/driver-api/media/v4l2-isp.rst` |
| DRM helpers | `drm_gem_dma`, `drm_fbdev_dma`, `drm_mipi_dsi` host API, 106 DSI panel drivers, LT8912B and LT9611 bridges | `drivers/gpu/drm/drm_fbdev_dma.c`, `drivers/gpu/drm/bridge/lontium-lt8912b.c` |

Missing: any driver, binding or DTS node for VI/ISP, CIF/CSI-2 RX, VPSS, VO/DISP, MIPI DSI TX, D-PHY (RX or TX),
video codec, JPEG, TPU, IVE, DWA/LDC, RGN; the VIP_SYS sub-controller (3.3); reserved-memory/CMA pools; a
mailbox DTS node. `grep -rli 'sophgo\|cvitek\|cv18' drivers/gpu drivers/media drivers/staging` is empty.

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
| vo + mipi_tx + fb | 6.9k + 0.9k | `/dev/cvi-vo`, `/dev/cvi-mipi-tx`, `/dev/fb0` | own no display registers (4.1) |
| vcodec + jpeg + cvi_vc_drv | 55.7k | 81 + 17 + 16 ioctls | ~70% Chips&Media host API and sample code inside the kernel |
| dwa, rgn, ive, tpu, mon, rtos_cmdqu, fast_image | 4k, 3.5k, 21k, 4.2k, 1.7k, 1.1k, 1.2k | misc/cdev | section 6 |
| pwm, rtc, wdt, saradc, clock_cooling, wiegand | small | frameworks with vendor compatibles | mostly replaceable by mainline |

The snapshot is two years behind Sophgo's `sg200x-dev` branch (weekly releases continued through 2026-08-24
with VO/VPSS/VI fixes); any port should rebase first.

### 3.3 Vendor-kernel-only dependencies

Exactly ten vendor-only exported symbols are used by `osdrv` (the six ION functions, `arch_sync_dma_for_device`
and three eFuse functions, rows 1-3); the remaining rows are vendor-kernel behaviours and headers that also
disappear on mainline:

| Vendor facility (5.10) | Used by | Mainline 7.3 state | Replacement |
|---|---|---|---|
| `cvi_ion_alloc/_nofd`, `cvi_ion_free/_nofd`, `ion_buf_begin/end_cpu_access`, `struct ion_buffer.paddr/vaddr/name` (vendor ION fork, carveout heap on reserved-memory `ion-region`) | ION **API** calls only in `sys.c` (15 sites) and `tpu` (`cvi_tpu_interface.c:302,320-331`); `dwa/ldc_test.c` is compiled out. ION **headers** are also pulled in by `sys/common/uapi/sys.h:9-10` (and therefore by vo, vi, vpss, rgn, ive, base, fast_image, vcodec, jpeg), by `fast_image.c`, `vdi.h`, `jdi.h`, and 12 Makefiles add `-I$(srctree)/drivers/staging/android` | ION removed in 5.11; no carveout heap; no in-kernel heap allocation API | Allocator + dma-buf **exporter** inside `sys.c` (CMA reserved pool, `cma_alloc`/`dma_alloc_attrs`, `dma_buf_export` with cached/write-combine mmap per `is_cached`), keeping the exported `sys_ion_*` signatures (48 external call sites unchanged); remove the header propagation; `MODULE_IMPORT_NS("DMA_BUF")`. Note: the Armbian TPU patch is an **importer** (userspace allocates from the CMA heap); it is a reference for cache maintenance, not for the allocator |
| `arch_sync_dma_for_device()` | `sys.c:378-410`, `fb/cvifb.c`, `tpu`, `vpss/scaler.c` (10 sites) | Defined but **not exported** (`arch/riscv/mm/dma-noncoherent.c`) | `dma_sync_single_for_device/for_cpu` on a `dma-noncoherent` platform device, accepting physical ranges that `sys` did not allocate (the ISP working pool is supplied by userspace). Semantics differ: the vendor "invalidate" is clean+invalidate, mainline `for_device(FROM_DEVICE)` only cleans. The middleware calls `SYS_CACHE_INVLD` right after every cached mmap (`cvi_mpi modules/sys/src/cvi_sys.c:612-615`), so this must be an explicit test case |
| `cvi_efuse_read_buf/write/read_from_shadow` | `base.c` (sysfs debug attributes, UID), `saradc` | No Sophgo nvmem driver upstream (Armbian carries one) | Drop sysfs attributes or use nvmem |
| `sched_setscheduler()` export, `MAX_USER_RT_PRIO`, `CONFIG_SCHED_CVITEK` auto RT boost of `cvitask_*` threads | `vi`, `vo`, `vpss`, `dwa`, `cvi_venc` (14 sites) | Only `sched_set_fifo*` exported | `sched_set_fifo()`; re-measure frame-drop behaviour |
| `i2c_adapter.i2c_idx`, `I2C_M_WRSTOP` | `snsr_i2c` | Absent | `adap->nr`, `I2C_M_STOP` (DesignWare splits transfers at STOP) |
| Vendor headers `pinctrl-cv181x.h` (direct FMUX writes), `tee_cv_private.h`, `streamline_annotate.h`, `cv180x_efuse.h`; `-I$(srctree)/drivers/{staging/android,tee,pinctrl/cvitek}` | `cif`, `vo`, `base`, `tpu`, several | Absent | Remove; convert pinmux to DT pinctrl states |
| vermagic/modversions ignored under `CONFIG_ARCH_CVITEK` | module loading | Enforced | Rebuild per kernel |

**VIP_SYS.** The block at 0x0A0C8000 (confirmed by the vendor DTS, Appendix A) holds the per-block VIP resets
(isp_top, img_d/v, sc_*, disp, bt, dsi_mac, csi_mac0-2, ldc, dsi_phy, csi_phy0, csi_be, ive), clock gates and
dividers (`CLK_LP` 0x14, `CLK_CTRL0` 0x18, `CLK_CTRL1` 0x1c, `CLK_RATIO` 0x30-0x48) and AXI real-time/offline
switches. `base.ko` maps it and exports raw accessors (`vip_toggle_reset`, `vip_sys_reg_write_mask`) used from
`vi`, `vpss`, `cif`, `dwa`, `ive`. `CLK_CTRL0` is shared between display and camera: it holds the DSI/BT clock
select written by the display code **and** the CSI0/CSI1 RX and VI clock selects (`base/chip/cv181x/vip_common.h:74-88`);
`vo` overwrites the whole register with a 0xFFFFFFFF mask (`vo.c:433-472,1794-1833` via `dsi_phy.c:336-340`).
Mainline has no provider for this block (the clock binding allows only the 0x03002000 range). Any strategy
needs either `base.ko` as owner or a new syscon with reset and clock-gate cells, and bit-field writes.

**Direct pokes into blocks that mainline drivers own** must be removed: 38 fixed-address `ioremap` sites in `cif.c` (30 into the clock
controller 0x03002xxx for cam PLL power/dividers, compiled in on mainline because `CONFIG_COMMON_CLK_CVITEK` is
unset; 4 into VIP_SYS `CLK_CTRL0` and 4 into the display block at 0x0A0880F8, which belong to the VIP_SYS/SC_TOP
owner rather than the clock framework),
`0x03002840` DISPPLL bit in `scaler.c:3613-3625` and `0x3002008` in `scaler.c:2394`, FMUX writes to 0x03001000
(52 in `vo.c`, 19 in `cif.c`; parallel/BT/TTL modes only), the DDR controller patch `0x08004544` and VIP AXI
priority `0x0A0C8070` called from `cvifb.c`, and the VBAT registers `0x03000220/0x03005144` in `vo_mipi_tx.c`
(dead code: the vendor DTS wires no interrupt to `mipi_tx`).

### 3.4 5.10 to 7.3 API breakage (compile-only view)

Full inventory in the api-delta report (44 rows). The items that matter:

| Class | Sites | Severity | Fix |
|---|---|---|---|
| ION allocator + physical-address ABI | 15 direct ION calls in `sys.c`; 119 lines in 11 modules reference the `sys_ion_*`/`sys_cache_*` wrappers (48 of them allocation/free calls); header propagation via `sys.h` | blocker (design) | shim in `sys.c`, 3-6 weeks incl. cache ops and header cleanup |
| `arch_sync_dma_for_device` not exported | 10 | high | centralise in `sys_cache_*` |
| `strncpy()` **removed from the kernel** (`Documentation/process/deprecated.rst:134-137`) | 171 (vpss 52, vo 89, base 4, sys 4, rgn 5, rest codec/dwa/mon; vi and cif 0) | high (volume) | per-site `strscpy`/`strscpy_pad`/`strtomem_pad`, 1-2 weeks |
| `-Wall -Wextra -Werror` in 18 Makefiles (plus `-Werror` alone in `jpeg` and `vcodec`; 20 to edit) vs stricter 7.3 default warnings (`-Wmissing-prototypes` etc.) | up to ~2000 non-static functions | high (volume) | drop `-Werror` for bring-up; restoring it 2-4 weeks |
| `platform_driver.remove` returns `void` | 26 | trivial | |
| `class_create(THIS_MODULE, ...)` | 13 (+2 extdrv) | trivial | |
| `<linux/of_gpio.h>` removed; legacy integer GPIO API deprecated; GPIO numbers arrive from userspace in `vo`/`vi` | 4 lookups + ~20 legacy calls | moderate | gpiod from DT |
| `MODULE_IMPORT_NS("DMA_BUF")` required | 0 present | trivial | |
| `sched_setscheduler`/`MAX_USER_RT_PRIO`, timers (`del_timer_sync`, `from_timer`), `vm_flags_set`, `DEFINE_SEMAPHORE`, thermal 5-arg register, `FBINFO_DEFAULT`, `PDE_DATA`, `compat_ptr_ioctl` redefinition (COMPAT is on for rv64) | ~60 | trivial | |
| PWM chip API rewritten | 1 module | moderate | drop module; mainline `pwm-sophgo-sg2042.c` has an identical register map (needs a cv18xx compatible; Armbian carries the patch) |
| In-kernel firmware file reads (`filp_open` of `/usr/share/fw_vcodec/*.bin`) | codec | moderate | `request_firmware()` |

Compile-only estimate for **all** of `interdrv/v2` against 7.3 with `-Werror` removed: **8-15 engineer-weeks**;
for the nine display/camera modules only (~78k LOC, codec/PWM/misc excluded, ION stubbed): **3-6 weeks**.
`extdrv/wireless` (1751 files of third-party WiFi drivers) is out of scope; use in-tree drivers.

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
- **MIPI DSI MAC**: CVITEK-specific (not DesignWare), small register set at SCL_TOP + 0xA000, 1/2/4 lanes, ECC and
  CRC16 computed in software, 16-byte LPDT FIFO, 4-byte reads (`scaler.c:3714-4126`). The vendor driver accepts
  **video burst mode only** (`vo/chip/cv181x/vo_mipi_tx.c:95-108`); the TRM states the hardware also supports
  command and non-burst modes (`sophgo-doc .../video/mipi_tx.rst:22-30`), so the restriction is software (silicon
  behaviour unverified).
- **D-PHY** at 0x0A0D1000: lane role remap and P/N swap, in-driver PLL math, HS timing, LVDS mode; its PLL code
  also writes a VIP_SYS `CLK_CTRL0` bit (`vpss/chip/cv181x/dsi_phy.c:118-135,237-334,255-259`).
- Single device/layer/channel; interlaced rejected; rotation goes through the DWA engine.
- **`vo`, `mipi_tx` and `fb` own no display registers and no display interrupt.** Their register and IRQ setup
  code is `#if 0` (`vo_core.c:168-220`). All display register code is in `vpss` and exported (59 `sclr_*` +
  13 `dphy_*` symbols; `vo`/`fb` consume 68 of them). `vpss` owns the ioremaps of 0x0A080000 and 0x0A0D1000 and the
  shared `sc` IRQ, whose single status register mixes scaler and display-vblank bits; display IRQs reach `vo`
  through a callback (`vpss_core.c:1332-1363,1421-1442`).
- The display code is **wider than one contiguous range**: besides `scaler.c:2249-4300` and `dsi_phy.c`, display
  functions live at `scaler.c:708-739` (top config, display enable in the same word and spinlock as the scaler
  enables), `875-912` (BT enable, VO mux, MCU), `1374-1431` (interrupt mask/status/clear), `4607-4621` (display
  source), `4627-4665` (AXI/DDR patches), and they write SC_TOP-page registers `BT_CFG` (+0x0C), `LVDSTX` (+0x50),
  `BT_ENC/BT_SYNC_CODE` (+0x60/0x64) and `VO_MUX0-7` (+0x90..0xAC). 13 of the 68 consumed exports are shared with
  the scaler path (GOP programming, top config, interrupt mask, `sclr_ctrl_set_disp_src` which reprograms
  scaler SC_D), and the statics `g_top_cfg`/`g_disp_cfg`/`g_gop_cfg` are initialised by the `vpss` probe.
- Direct pokes by the display modules: 52 FMUX writes in `vo.c:67-205` (BT/parallel only), 10 full-register writes
  of VIP_SYS `CLK_CTRL0` (see 3.3), DISPPLL `0x03002840` and `0x3002008` from `scaler.c`, DDR controller
  `0x08004544` and AXI priority `0x0A0C8070` from `cvifb.c:91,795,825`.
- **Panel initialisation is entirely userspace**: `/dev/cvi-mipi-tx` ioctls carry lane map, timing, pixel clock
  and raw DCS packets; the kernel holds no panel tables (`vo_mipi_tx.c:148-276`). Frames reach DISP as physical
  addresses of VB blocks (`vo_sdk_layer.c:374-420`). `cvifb` is a thin fbdev over GOP layer 1 backed by a
  reserved-memory region and uses the non-exported `arch_sync_dma_for_device` and the removed `FBINFO_DEFAULT`.
  The vendor init script does not even load `mipi_tx`; `vo` alone drives DSI (Appendix A).
- Public documentation exists: the SG2000 TRM documents VDP DISP (0x0A088000), OSD (0x0A088800), MIPI TX
  control (0x0A08A000) and PHY (0x0A0D1000) at register level (`sophgo-doc/SG200X/TRM/contents/en/video/`).
  It has no description of the SC_TOP interrupt status register, so whether that register is write-1-to-clear per
  bit (needed for two drivers to share the IRQ) is **unverified**; the vendor code is consistent with it
  (`scaler.c:1413-1416`) and the vendor driver already requests the line with `IRQF_SHARED` (`vpss_core.c:1664`),
  although no second consumer exists in the vendor tree. Write-1-to-clear (W1C) means writing a 1 to a status bit
  clears only that bit, which lets two drivers acknowledge their own interrupts independently.

### 4.2 Mapping to mainline

The display engine maps cleanly onto DRM/KMS: one CRTC (DISP timing generator, vblank from the `disp_frame_end`
bit, atomic flush via the shadow force-update bit), a primary plane (planar/semi-planar YUV, packed YUV, RGB888),
2-4 overlay planes (GOP windows), a DSI encoder implementing `mipi_dsi_host_ops.transfer` exactly like
`sun6i_mipi_dsi.c` (which also computes ECC/CRC in software), the D-PHY as a `drivers/phy` driver, and
`drm_gem_dma` + `drm_fbdev_dma` for `/dev/fb0`. Panels come from the 106 upstream DSI panel drivers
(`hx8394` and `ili9881c`, which the vendor timing tables reference, already exist); HDMI via the upstream LT8912B
or LT9611 bridges. Closest references: `ingenic-drm-drv.c` (1.7k lines, planes), `sun6i_mipi_dsi.c` (1.3k),
`sprd_dsi.c` (1.1k, in-driver PHY PLL), `mxsfb` (2.3k). Sophgo's CV186x DRM driver (`linux-common` 6.12.y,
`drivers/gpu/drm/cvitek`, 5.3k lines of C, 6.5k with headers, in 15 files: disp, dsi, lvds, dw_hdmi, MIPI PLL) is a structural template; it shares
no register macro names with the CV181x headers, so each write must be re-validated (the D-PHY PLL function
`_cal_pll_reg` is the same algorithm in both). Expected driver size 2-3.5k lines.

Structural problems to solve before coding:
1. **SC_TOP is shared page-wide** (config, interrupt mask/status/enable, BT/LVDS/VO-mux registers): needs a syscon
   regmap or an MFD parent shared by the DRM driver and any scaler driver, and either a bench-confirmed
   write-1-to-clear status register with `IRQF_SHARED` or a small interrupt demultiplexer owning +0x30..0x38.
   While no scaler driver exists on mainline, the DRM driver can own the page and the IRQ alone.
2. **VIP_SYS `CLK_CTRL0`** must be written as bit-fields through a syscon; the display must not clobber the camera
   clock selects. The DISPPLL bit at 0x03002840 becomes a clk framework call (bit 3 is modelled upstream; bit 1 is
   unexplained).
3. The 13 scaler-shared exports and the shared statics must be factored before the display HAL can be lifted out.
4. The "disp_from_sc" online path (scaler feeding DISP without DRAM) has no DRM analogue; drop it (default off).
5. fbdev is deprecated for new drivers; `drm_fbdev_dma` provides `/dev/fb0` in XRGB8888, not the vendor default
   16-bit ARGB4444 or 8-bit LUT modes.
6. Boot-logo handoff ("smooth" mode reading back bootloader state) must be implemented or replaced by a full reset.

### 4.3 Options and effort

| Option | Scope | Effort (engineer-weeks) | Upstreamable | Keeps vendor VO apps |
|---|---|---|---|---|
| D1 Forward-port `vo`+`mipi_tx`+`fb` on top of ported `sys`/`base`/`vpss` | fix ~12 API classes, pinmux to DT or disabled, DTS nodes | 3-6 (vpss reader) to 6-12 (vo reader) **after** the foundation is ported | no | yes |
| D2 Minimal DRM/KMS: 1 CRTC, primary plane, DSI host, D-PHY phy driver, fbdev emulation, bindings | DSI video-mode panel or bridge only | 8-14 to a reviewed driver, plus 1-3 for the SC_TOP/VIP_SYS syscon and IRQ groundwork; first picture 3-6 (reader estimate; 4-8 via the D4 verbatim-HAL route); +3-6 for upstream review rounds | yes | no |
| D3 Full DRM: D2 + GOP overlay planes, gamma, BT.601/656/1120 and LVDS encoders | all vendor outputs except I80 | 18-30 incl. review | yes | no |
| D4 "Ugly then clean": copy the display HAL verbatim into a thin DRM driver first, refactor later | fastest first light | 4-8 to first picture, converging into D2's total | eventually | no |

Assumptions: one engineer experienced in DRM atomic and DSI; a board with free DSI pads and a known panel or an
LT8912B/LT9611 bridge; no documentation beyond TRM + vendor code; a logic analyser for D-PHY debugging.

### 4.4 Display risks and unknowns

- D-PHY PLL and HS-timing constants are reverse-engineered from vendor code; per-panel tuning may be needed.
- Register `0x03002840` bit 1 and VIP_SYS `CLK_CTRL0` bit 4 are not explained by any source read.
- SC_TOP interrupt status write-1-to-clear semantics are unverified (bench item).
- Board: the Duo S has no free DSI pads without giving up SDIO WiFi; the LicheeRV Nano is an SG2002 (VIP identity
  with SG2000 assumed, unverified); the Duo Module 01 EVB is reported to carry an LT8912B-class DSI-to-HDMI bridge
  (search snippet only, unverified).

---

## 5. Camera path

### 5.1 CIF (MIPI CSI-2 receiver) and `snsr_i2c` (verified)

- Pure bridge with **no DMA**: zero address/stride fields in its register maps; data goes on-chip to the ISP's
  CSI bridge, which `vi` handles (`cif/chip/cv181x/drv/inc/reg_fields_csi_*.h`; `vi/chip/cv181x/vip/vi_drv.c:281-303`).
  Its two IRQs (vendor hwirq 26/27) only count ECC/CRC/WC/HDR/FIFO errors.
- 3 MACs, 2 MIPI-capable (MAC0 on a 4-lane PHY, MAC1 on a 2-lane PHY); 6 physical D-PHY lanes freely routable
  to logical CLK/D0-D3 with P/N swap; sub-LVDS, HiSPi, BT.601/656/1120, TTL inputs; VC/DT/DOL/manual HDR
  (`cif_drv.c:1263-1363`; `cif.c:456-581`).
- Userspace pushes the entire receiver configuration (lane map, hs_settle, HDR mode, data types) in one ioctl and
  streaming starts immediately (`cif.c:464-640`). Sensor init over I2C is done by userspace.
- `snsr_i2c` exists only to fire ISP-computed exposure/gain register groups synchronously with a target frame,
  optionally as one burst transfer with inter-message STOPs; it depends on two vendor I2C core patches.
- Uses the reset framework (`phy0`, `phy-apb0`, `phy1`, `phy-apb1` = vendor IDs 70-73, usable as raw numbers on
  the mainline reset controller), 6 clocks (all with mainline IDs), sensor reset GPIOs, plus VIP_SYS MAC dividers
  and direct PLL pokes. The vendor DTS passes 5 register entries; due to an indexing quirk the fifth (`pad_ctrl`,
  inside the pinctrl block) is never used (`cif.c:2820-2846`).
- Mainline fit: a V4L2 sub-device with media-controller and fwnode endpoints, like `rkisp1-csi.c` (518 lines),
  `cdns-csi2rx.c` (1.1k), `sun6i-mipi-csi2` (0.8k), with the PHY wrapper as a `drivers/phy` driver; `snsr_i2c`
  becomes unnecessary because sensors are in-kernel V4L2 sub-devices. Sub-LVDS/HiSPi have no V4L2 bus type.
  Common CVITEK/Milk-V sensors (gc2053, gc2083, gc4653, SmartSens sc*, os04a10) have **no** mainline driver;
  imx219/imx290/imx335/imx415/ov5647/ov5640 do.
- The TRM documents MIPI RX (D-PHY 0x0A0D0000, CSI 0x0A0C2400/0x0A0C4400) and VI top at register level.

### 5.2 VI / ISP (verified)

- Pipeline: 3 pre-raw front-ends (FE0 4 ch, FE1/FE2 2 ch) with CSI bridges, one pre-raw back-end, then
  RAWTOP/RGBTOP/YUVTOP post stage; 143 register blocks in a 512 KiB window at 0x0A000000; default single-sensor
  flow is FE -> BE -> DRAM -> post (multi-pass), with an optional online handshake into the VPSS scaler
  (`include/chip/cv181x/uapi/linux/isp_reg.h`; `vi.c` scene control). Single `isp` IRQ (vendor hwirq 24), a
  hi-tasklet and four SCHED_FIFO kthreads.
- **All 3A and tuning run in userspace.** The kernel only DMAs statistics into memblocks whose physical addresses
  it publishes (`vi.c:5428-5460`), applies double-buffered tuning nodes written by userspace at frame boundaries
  (`vi.c:7172-7190`), derives motion/DCI levels from statistics for VPSS/codec hints (`vi.c:6080-6132`), reads the HDR (FSWDR)
  hardware report (`vi.c:5520-5534`), exports the lens-shading buffer address (`vi.c:5462-5484`), applies a black
  Y-curve until userspace tuning arrives (`vi.c:45-51,1666-1669`) and unity white-balance gains (`vi.c:1586-1595`),
  and relays per-frame sensor register lists to I2C. A native driver must place these non-3A functions in params,
  statistics or controls. The three `VI_IOCTL_AE_CFG/AWB_CFG/AF_CFG` ids have no handler.
- Tuning nodes (post 47 KB, BE 16.6 KB, FE 104 B per node) live in kzalloc'd memory exposed to userspace by
  **physical address plus kernel virtual pointers** (`vi_tun_ip_ctrl.c:246-298`, `vi.c:5627-5643`). The VI mmap
  maps only an 8 KiB shared context (`vi.c:5875-5903`). **Verified in the middleware source**: userspace maps
  tuning and statistics buffers through `/dev/mem` (`cvi_mpi modules/isp/cv181x/isp/src/isp_tun_buf_ctrl.c:80,167`
  -> `modules/sys/src/cvi_sys.c:599,612` -> `modules/sys/src/devmem.c:25`). The ISP working pool is allocated by
  the middleware through the `sys` ioctl `SYS_ION_ALLOC` (`modules/vi/src/cvi_vi.c:369` -> `cvi_sys.c:630`), not
  through `/dev/ion`; no `ION_IOC_*` ioctl appears in any `cvi_mpi` C file, `/dev/ion` is only opened
  (`cvi_sys.c:831`). Consequence: a kernel-side allocator shim is sufficient for allocation, but `/dev/mem` access to
  kernel RAM is a **hard dependency** of the vendor camera stack (`CONFIG_STRICT_DEVMEM` and kernel lockdown must
  stay off, or the VI driver must gain an mmap-offset/dma-buf path for these buffers).
- Sensor drivers are userspace libraries; exposure/gain register lists are queued per frame and fired by the
  kernel through `snsr_i2c`.
- **Vendor middleware availability (verified):** the ISP middleware framework is public source in
  `github.com/sophgo/cvi_mpi` (branch `sg200x-dev`; `isp_mgr`, `isp_3a` plugin dispatch, ~35 per-block control
  files, stats/tuning buffer handling). The AE/AWB/AF algorithm cores and the tuning-interpolation library are
  **prebuilt objects only** (e.g. `modules/isp/algo/ae/obj/*.riscv64-unknown-linux-gnu.cv181x.o`; 56 riscv64
  objects, zero `.c` files under `modules/isp/algo/{ae,awb,af}`). The repository has **no licence file** and the
  sources carry "Copyright (C) Cvitek Co., Ltd. ... All rights reserved." with no SPDX tag (`isp_3a.c:1-7`). The
  3A plugin interface is a documented C API (`include/isp/cv181x/cvi_comm_3a.h:430-448`,
  `ISP_AE_EXP_FUNC_S` etc.) with source samples for replacement algorithms (`custom_ae/awb/af`). Sophgo also has
  a V4L2-flavoured adapter of this middleware for CV186x (`github.com/sophgo/isp`, `cv186x/v4l2_adapter`; kernel
  side not found, unverified).
- LOC split: register programming 9.0k lines in `vip/*_ip_ctrl.c` with essentially no kernel API use plus 28.3k
  lines of generated register headers (portable as-is); kernel glue 11.8k lines and 3.1k uAPI header lines are
  what a mainline rewrite replaces. The 10.6k-line `vi_vreg_blocks.h` shadow-register image is dead code on CV181x.
- Mainline fit: a V4L2 media-controller ISP driver with params (`META_OUTPUT`) and stats (`META_CAPTURE`)
  queues using the generic `v4l2-isp.h` framing. The vendor design (DMA'd stats, double-buffered per-block
  tuning with update flags) maps directly onto that model. References: rkisp1 (11-12.6k lines depending on whether the 1.7k-line
  uAPI header is counted), mali-c55 (5.5k), c3-isp (4.1k), pisp_be (1.8k, memory-to-memory scheduling). libcamera has **no** Sophgo pipeline handler.
  The ISP registers are documented only by `osdrv` headers (not by the TRM); those headers carry "All rights
  reserved" and no SPDX tag (`include/chip/cv181x/uapi/linux/vi_reg_fields.h:1-5`) although the modules declare
  `MODULE_LICENSE("GPL")`, so an in-kernel ISP driver is a derivative work whose licence basis needs Legal review.

### 5.3 VPSS (verified)

- IMG_IN x2 (display and video paths) + 4 scaler cores (SC_D, SC_V1-3; one read DMA shared by three outputs) +
  4 write DMAs + per-scaler GOP, privacy mask, border, slice-buffer handshake to the encoder; online (ISP -> VPSS
  without DRAM) and offline modes; 16 groups x 3-4 channels software model with a kthread scheduler; 96 call sites
  into the base/sys VB framework. It also hosts the display HAL (section 4).
- Mainline fit: offline scaling as a V4L2 mem2mem device modelled on Rockchip RGA (2.3k lines), with the
  1-in/3-out hardware forced into 1:1 jobs; online mode only makes sense as resizer sub-devices inside an ISP
  media graph. GOP/privacy/slice-buffer features have no V4L2 equivalent.

### 5.4 Options and effort

| Option | Scope | Effort (engineer-weeks) | Upstreamable | Keeps vendor camera apps and tuning |
|---|---|---|---|---|
| C1 Forward-port `cif`+`snsr_i2c`+`vi`+`vpss` on top of ported `sys`/`base` | API fixes, CCF-only clock path, pinctrl states or MIPI-only, DTS nodes, `/dev/mem` kept (or VI mmap offset for tuning buffers), ISP pool via the `sys` shim | cif 2-4, vi 6-10, vpss 3-5; with sys/base 3-6, DT/clocks 2-4 and integration 2-4: 18-33 | no | yes (unchanged middleware; `/dev/mem` required) |
| C2 V4L2 CSI-2 RX sub-device + D-PHY driver + VI raw/YUV capture (no ISP processing), usable with libcamera "simple" + software ISP | bindings, DT, one mainline sensor | 8-16 | yes | no |
| C3 Native V4L2 ISP driver (+ VPSS as m2m/resizers) | reuse `vip/*_ip_ctrl.c` behind a rkisp1/pisp_be-style shell; params/stats via `v4l2-isp.h`; sensors as sub-devices | 20-35 kernel (+4-8 VPSS m2m, +2-4 per missing sensor driver) | yes (long review) | no |
| C4 libcamera pipeline handler + IPA | (a) shim linking the published prebuilt 3A objects behind the documented plugin API: 4-8, out-of-tree only, redistribution rights unverified; (b) open AE/AWB/AF written against the documented plugin contract and stats layouts: 10-20; (c) from scratch: 15-30 | 4-30 | (b)/(c) yes | no |
| C5 Staged: C1 first, then wrap VI capture/stats/params in V4L2 nodes carrying the vendor structs | transition path | C1 + 10-16 | no (vendor structs in META formats) | partially |

Assumptions: single RGB sensor, offline post-to-DRAM path, no HDR/3DNR parity, no FreeRTOS fast-boot, no
ISP -> VPSS online mode for the first milestone; a board with a sensor that has a mainline driver for the V4L2
options. Image-quality parity with the vendor tuning should not be assumed for C3/C4(b)/(c). The C2 raw-capture
figure assumes the ISP front end can write RAW/YUV to DRAM without programming the back-end and post stages; that
minimal sequence is unverified, and if the full pipeline must be programmed C2 absorbs part of C3. New sensor
drivers (2-4 weeks each) assume register-level datasheets are obtainable; vendor mode tables live in closed
`libsns_*.so` and some sensor datasheets are NDA-only. The ISP driver figure 20-35 (vi-isp reader) is used instead
of the infrastructure reader's 10-20 because the 143 register blocks and the 47 KB parameter node imply a
multi-round uAPI review.

### 5.5 Camera risks and unknowns

- Coherency: the middleware flushes/invalidates by physical address; on the `dma-noncoherent` C906 any mismatch
  yields stale statistics or corrupted frames. The kernel must enable `CONFIG_ERRATA_THEAD`. The invalidate
  semantics change (clean+invalidate -> invalidate) hits every cached allocation at creation time.
- `/dev/mem` dependency (verified) versus any hardened kernel configuration.
- hs_settle, deskew phases and clock-lane direction rules exist only as heuristics in `cif.c`.
- Loss of the vendor kernel's automatic RT-priority boost for `cvitask_*` threads may change frame-drop behaviour.
- No open 3A exists for this ISP; the prebuilt cores have no stated licence.

---

## 6. Other modules

| Module | Verdict | Reason / mainline home |
|---|---|---|
| vcodec + jpeg + cvi_vc_drv | Defer; forward-port 6-12 weeks, native V4L2 stateful codec 30-55 weeks | Chips&Media CODA980 (H.264, 0x9800) + WAVE420L (HEVC, 0x4201) + CODAJ12-class JPEG. Mainline `wave5` supports only WAVE5xx; `coda` supports CODA960 not 980; no CODAJ12 driver. Firmware is read by the kernel from `/usr/share/fw_vcodec`; redistribution terms unclear. |
| tpu | **Already ported** to mainline 7.2/7.3 by a Cvitek engineer in Armbian (dma-buf import from the CMA heap, paddr ioctl, `dma_sync_sgtable_*`, binary-compatible uAPI with a small `libcviruntime` change) | `armbian/build` patch `0064-drivers-soc-sophgo-add-CV181x-SG200x-TPU-driver.patch` |
| dwa (LDC/dewarp) | Forward-port 2-3 weeks or V4L2 m2m later (NXP dw100, 1.7k lines, as reference) | depends on base VIP_SYS helpers and VPSS callbacks |
| rgn | Follows VPSS/VO; software-only OSD manager over ION canvases | would become DRM planes / V4L2 controls |
| ive | Forward-port 2-4 weeks; no mainline framework | custom misc or accel driver |
| rtos_cmdqu + fast_image | Replace transport with mainline `cv1800-mailbox.c` (identical register map) + a small hwspinlock; drop fast_image unless FreeRTOS fast-boot ISP is required | Armbian carries C906L remoteproc + mailbox DTS |
| rtc, saradc, wdt, pwm | **Drop**: mainline `rtc-cv1800`, `sophgo-cv1800b-adc`, `dw_wdt` (needs DTS node), `pwm-sophgo-sg2042` (needs cv18xx compatible; Armbian patch exists) | check SARADC eFuse trim and RTC power-on features if needed; the vendor `wdt` binds to `snps,dw-wdt` and would double-bind with mainline |
| mon, clock_cooling | Drop (debug profiler; the vendor cooling compatible does not even match the sg200x DTS; Armbian carries a thermal driver) | |
| wiegand, extdrv tp/gyro | trivial fixes | |
| extdrv/wireless | Out of scope; use in-tree brcmfmac/mt76/rtw drivers | |

---

## 7. Prior art and ecosystem (as of 2026-10-06)

| Source | What it tells us | Access / confidence |
|---|---|---|
| `sophgo/linux` wiki, "Peripherals Status" (revision 2026-09-02) | DRM, Media and TPU rows: **Not Started**. Base peripherals upstream (clk 6.10, pinctrl 6.12, reset 6.17, mailbox 6.16, RTC 6.16, ...). Under review: eFuse, timer, watchdog, PWM (v8), thermal (v5), remoteproc C906L (v2), I2S (v4). Duo S minimal DTS in 7.3. | cloned wiki repo; verified |
| Public patch series for CV18xx camera/display | None found. | lore/patchwork unreachable; search snippets only; likely |
| Armbian `sophgo-sg200x` family | Duo S on mainline 7.2 ("edge") and 7.3 ("bleedingedge") with ~60 patches: thermal, MDIO mux, timer, watchdog, PWM, C906L remoteproc + mailbox DTS, eFuse nvmem, I2S, TPU driver, 128 MiB CMA pool; config enables `ERRATA_THEAD`, CMA, `DMABUF_HEAPS_CMA`, `MEDIA_SUPPORT`. No video drivers. Natural integration base for a port. | cloned repo; verified |
| Armbian TPU port (PR 10760, patch 0064, author Wellken Chen, Cvitek) | Vendor misc driver re-hosted on mainline as a dma-buf **importer** from the CMA heap, `GET_DMABUF_PADDR`/`RELEASE_DMABUF` ioctls, `dma_sync_sgtable_*`, TEE dropped; needed a small userspace allocator change. | patch files read; verified |
| `sophgo/cvi_mpi` (branch `sg200x-dev`, 2026-08-24) | Public middleware source; ISP framework open, 3A cores prebuilt, no licence file; buffer allocation via `SYS_ION_ALLOC`, all mappings via `/dev/mem`. | cloned; verified (section 5.2) |
| `sophgo/isp` | CV186x ISP middleware with a V4L2 adapter layer (`cv186x/v4l2_adapter`); prebuilt `libae/libawb/libaf`. | cloned; kernel counterpart unverified |
| NixVegas/BadgeOS (mainline on Duo Module 01) | Documents that VO/DSI, ISP, codec, TPU were the only vendor-5.10-only subsystems; wants a minimal DRM/KMS driver for DISP + DSI host to drive an LT8912B HDMI bridge; no mainline display output achieved. | repo cloned; issue text from search snippet; likely |
| scpcom / Fishwaldo Debian images | Vendor 5.10 + osdrv; DSI panels, SPI LCD, LT9611 DSI -> HDMI (part of panel init done in the bootloader); cameras via osdrv. Contains the vendor SoC/board DTS used in Appendix A. | repo cloned; verified |
| Sophgo `linux-common` 6.12.y (BM1688/CV186 line) | GPL DRM/KMS driver `drivers/gpu/drm/cvitek` (disp, dsi, lvds, dw_hdmi, MIPI PLL; 5.3k lines of C, 6.5k with headers, 15 files; no SPDX tags) for the sibling CV186x. Structural template; register names do not match CV181x. No vendor V4L2 camera driver in that tree. | repo cloned; verified |
| Sophgo `osdrv` `bm1688` branch | Same driver family adapted to 6.x (31 `KERNEL_VERSION(6,0,0)` guards, up to 6.12), still ION-based. Catalogue of API fixes Sophgo already made. | repo cloned; verified |
| Sophgo SDK manifests (`sophpi`) | SG200x SDKs stay on `linux_5.10`; osdrv weekly releases continued through 2026-08-24. Only BM1688/CV186 uses 6.12. | verified |
| SG2000 TRM (`sophgo-doc`) | Register-level docs for VI top, VDP DISP/OSD, MIPI RX, MIPI TX. **Not** for ISP, VPSS, codec, JPEG, TPU, IVE, DWA, nor the SC_TOP interrupt register. | verified |
| libcamera (2026-09-18) | No Sophgo/Cvitek pipeline handler. | mirror cloned; verified |

---

## 8. Strategies compared

Three plans were developed independently and scored by two judges (evidence grounding, estimate realism, risk
handling, completeness, fit to the question; 10 points each, 50 maximum). Sums below are reconciled against the
per-workstream ranges (the planners' own totals contained arithmetic slips).

| | A. Forward-port everything (vendor ABI) | B. Upstream-native rewrite | C. Staged hybrid |
|---|---|---|---|
| Idea | Rebuild `sys/base/vpss/vo/mipi_tx/fb/cif/snsr_i2c/vi` (+rgn/dwa) as out-of-tree modules on the Armbian 7.3 kernel; ION shim in `sys.c`; keep every `/dev/cvi-*` ioctl; vendor middleware unchanged | New DRM/KMS display, V4L2 media-controller camera (CSI-2 RX, VI raw capture, ISP with `v4l2-isp.h`, VPSS m2m), libcamera pipeline handler + IPA; VIP_SYS syscon and bindings; vendor middleware dropped | Compile-only measurement, then vendor-ABI camera (as A), display as new DRM driver (as B, display part), optional native camera later |
| Display result | vendor fbdev + ioctl stack on 7.x | DRM/KMS, upstreamable | DRM/KMS, upstreamable |
| Camera result | vendor stack with vendor 3A | native, new 3A, no codec | vendor stack first; native optional |
| Effort to display + camera running | 30.5-56 engineer-weeks (workstream sum 27.5-51 plus 3-5 for rgn/dwa; the planner stated 30-53) | display MVP 12-24, full display +6-12, raw camera +8-16, ISP/VPSS/IPA +39-73, sensors +2-8, userspace guidance +1-3, upstream review +8-16 counted as effort: 76-152 (planner stated 75-150; +6-12 for online VPSS resizers) | Stage 0: 3-6; Stage 1 camera: 18-33 (cumulative 21-39); Stage 2 display: 12-26 (cumulative 33-65); Stage 3 optional 36-97 as scoped in 9.2 (+7-15 for HDR streams and online VPSS; spread dominated by the 4-30 IPA option) |
| Upstream value | none | high | display yes, camera no (until Stage 3) |
| Hardware codec | available via the ported `cvi_vc_drv` later (6-12) | none (CODA980/WAVE420L unsupported) | as A |
| Maintenance | permanent API chasing (estimate 0.5-3 engineer-weeks per major kernel bump; planners gave 0.5-2 and 1-3) | in-tree after merge; libcamera IPA ownership | A's cost for camera, B's benefit for display |
| Key risk | middleware memory/ABI assumptions; `/dev/mem`; coherency | ISP effort spread, 3A quality, header licensing, review calendar | two owners of SC_TOP/VIP_SYS/IRQ; "ugly" DRM may stall |
| Judge scores (realism / decision-fit) | 38 / 36 | 38 / 38 | **40 / 39** |

Both judges recommend **C**, with these grafts: A's fully specified `sys.c` work package and go/no-go tests;
B's register-map step (vendor HAL vs TRM vs CV186x template), Legal gate and measurable quality targets; the
verifiers' corrections (middleware facts, `/dev/mem`, VIP_SYS bit-field writes, display HAL extent, ION header
propagation, cache-invalidate test). The decision-fit judge also notes what all three plans missed: a product-requirements
decision before any engineering (vendor SDK vs standard Linux APIs decides A vs B vs C).

Where each strategy is honestly weaker: A delivers only "vendor SDK on a modern kernel" and nothing upstream;
B is the slowest route to any camera and cannot reach vendor picture quality without the closed 3A; C carries
non-upstreamable patches in `vpss` (SC_TOP regmap, masked IRQ) that are thrown away if Stage 3 happens, and its
first gate (compile-only) measures the cheapest kind of breakage while the real risk surfaces at the allocator
and camera gates.

---

## 9. Recommended plan (staged hybrid, corrected)

### 9.1 Decision gate before engineering (0 weeks of engineering; elapsed time depends on Legal and product turnaround, assumed 1-2 weeks, not evidenced)

1. **Product requirement**: is the target userspace the vendor SDK (`cvi_mpi`, hardware codec, vendor 3A) or
   standard Linux APIs (DRM, V4L2, libcamera, no codec)? Vendor SDK -> Stages 0-2; standard APIs -> Stages 0, 2
   and 3 (camera gate G3 in B's sense). Both -> full plan. Also confirm which core runs Linux (RISC-V C906 assumed throughout; the Arm
   Cortex-A53 option was not assessed).
2. **Legal review request** (Dentsply Sirona Legal/Compliance): (a) licence basis of in-kernel code derived from
   `osdrv` register headers ("All rights reserved", no SPDX, modules `MODULE_LICENSE("GPL")`); (b) use and
   redistribution of `cvi_mpi` binaries, prebuilt 3A objects, ISP tuning files and codec firmware in a product
   image; (c) whether a libcamera IPA may link the prebuilt 3A objects. TRM-documented blocks (DISP/OSD/MIPI
   TX/RX/VI top) can be written from the public TRM.
3. **Read `cvi_mpi`** (`modules/sys/src/cvi_sys.c`, `devmem.c`, `modules/vi/src/cvi_vi.c`,
   `modules/isp/cv181x/isp/src/isp_tun_buf_ctrl.c`) to confirm the verification-pass findings against the SDK
   version actually in use, and whether any caller fails when `open("/dev/ion")` fails (`cvi_sys.c:831`).
4. **Manual mailing-list search** (lore.kernel.org `sophgo/`, dri-devel, linux-media) for in-flight CV18xx
   CSI/ISP/DRM/DSI series from an unrestricted network.

### 9.2 Workstreams

| Stage | WS | Deliverable | Depends on | Effort (eng-weeks) |
|---|---|---|---|---|
| 0 | WS0 Baseline | osdrv rebased to vendor `sg200x-dev` HEAD; vendor 5.10 reference image with recorded frame checksums and fps; Armbian 7.3 kernel booting on the bench board with `ERRATA_THEAD`, CMA, `DMABUF_HEAPS_CMA`, DRM, MEDIA; vendor DTS imported (Appendix A); hardware kit on the bench | - | 1-2 |
| 0 | WS1 Compile-only measurement | `sys, base, cif, snsr_i2c, vi, vpss, vo, fb, rgn` build and modpost-link against 7.3: `-Werror/-Wextra` removed, ION stubbed, ION header propagation removed (`sys.h`, `fast_image.c`, `vdi.h`, `jdi.h`, 12 Makefiles), temporary `strncpy` compat header, efuse/annotate stubs; per-module breakage report vs the inventory; list of every `ioremap(0x...)` and `PINMUX_CONFIG` site | WS0 | 2-4 |
| 1 | WS2 `sys`/`base` runtime | `sys.c` allocator on a reserved-memory CMA pool with `dma_buf_export` (cached vs write-combine per `is_cached`; kref'd objects replacing the 100-entry table; no stray fds in the ioctl caller, documented); `sys_cache_*` and `SYS_CACHE_*` via `dma_sync_single_*` on the `cvi-sys` device, accepting any pool paddr incl. the userspace-supplied ISP pool, with an A/B switch for clean+invalidate; `MODULE_IMPORT_NS("DMA_BUF")`; `base` trivial fixes and review of the 8 `strncpy` sites in `sys`/`base` deferred by the compat header; `base` keeps VIP_SYS via its DT reg (size 0x100) with **bit-field** accessors; chip-id via syscon regmap; efuse sysfs dropped or nvmem (Armbian patch) | WS1 | 3-6 |
| 1 | WS3 DT, clocks, resets, pinmux policy | `sg2000-vip.dtsi` from the vendor DTS with clock ids translated by name to `sophgo,cv1800.h`, resets 70-73 raw, IRQs isp 24 / sc 25 / dwa 28 (vendor numbering = `SOC_PERIPHERAL_IRQ(n-16)`), CMA pool(s) sized per product (vendor Duo S ION 74 MiB; Armbian uses 128 MiB; another source quotes 170 MiB), board fragment; `cif.c` CCF-only (remove the `CONFIG_COMMON_CLK_CVITEK` ifdef and the 30 clock-controller pokes; its 8 VIP_SYS/display-block pokes go through the VIP_SYS/SC_TOP owner), `scaler.c` DISPPLL via `clk_prepare_enable(CLK_DISPPLL)` with bit 1 bench-checked, `PINMUX_CONFIG` sites disabled (MIPI-only) with MIPIRX pad bias as a pinctrl state | WS0 | 2-4 |
| 1 | WS4 `cif` + `snsr_i2c` | gpiod, `devm_reset_control_get_optional_exclusive` + `IS_ERR` fixes, void remove, `adap->nr`, `I2C_M_STOP`, pad_ctrl entry dropped, VIP_SYS dividers via `base` | WS2, WS3 | 2-4 |
| 1 | WS5 `vi` | trivial fixes (class_create, remove, `sched_set_fifo`, timers, `compat_ptr_ioctl`, headers, `do_exit`); CMDQ buffer via the shim; **tuning/stats buffers**: either keep `/dev/mem` (document `STRICT_DEVMEM=n` and no lockdown as a hard precondition) or add a VI mmap offset / dma-buf export (recommended, keeps `VI_IOCTL_GET_TUN_ADDR` returning paddr); bandwidth-limiter ioremaps to DT/syscon; single-sensor offline path | WS2, WS3, WS4 | 6-10 |
| 1 | WS6 `vpss` (offline scaler, display HAL dormant) | `pde_data`, void remove, ION via shim (6 sites), 52 `strncpy` sites reviewed, DDR/AXI pokes dropped or moved to `base`; online mode off | WS2, WS3 | 3-5 |
| 1 | WS7 Camera integration, coherence validation (G1) | end-to-end camera with the vendor middleware; cache-coherence checksum suite incl. the invalidate-after-cached-mmap case; performance vs the 5.10 baseline; G1 report | WS4, WS5, WS6 | 2-4 |
| 2 | WS8 Display register map and bench facts | field-by-field map of the display HAL (full extent per 4.1) vs TRM VDP/MIPI-TX tables vs the CV186x template; bench tests on the vendor 5.10 image: SC_TOP `INTR_STATUS` write-1-to-clear, `0x03002840` bit 1, `CLK_CTRL0` bit 4, whether pure MIPI needs any FMUX write, DSI non-burst capability | WS0 | 1-3 |
| 2 | WS9 SC_TOP / VIP_SYS ownership | `vip-sys` syscon (regmap + reset/clock-gate cells over `VIP_RESETS/RESETS1/CLK_*`) and `sc-top` syscon (page-wide: CFG0/1, INTR_*, BT/LVDS/VO_MUX); `base` accessors re-implemented on the regmap (signatures unchanged); the DRM driver owns SC_TOP display bits and the `sc` IRQ alone for first light; the `vpss` split (regmap field writes that preserve the vendor's whole-register fields such as the QoS-enable and debug bits, masked IRQ bits, callback removal, 13 shared exports and shared statics factored) is done only when both must coexist (G3), with a single-owner + notifier fallback if `INTR_STATUS` is not per-bit write-1-to-clear | WS3, WS8 | 2-4 (+1-2 fallback) |
| 2 | WS10 DRM/KMS first picture (G2) | `drivers/gpu/drm/sophgo`: one CRTC, primary plane, DSI host (`transfer` with software ECC/CRC, attach rejects unsupported modes until non-burst is bench-verified), D-PHY programming from `dsi_phy.c` with statics removed, `drm_gem_dma` + `drm_fbdev_dma`, panel/bridge via `drm_of_find_panel_or_bridge`; board DTS with free MIPI_TX pads | WS3, WS9 | 4-8 |
| 2 | WS11 DRM clean-up toward upstream shape | D-PHY as `drivers/phy/sophgo`, regmap helpers, `atomic_check` for alignment/window limits, 2-4 GOP overlay planes, gamma LUT, YAML bindings; BT/LVDS/I80 excluded | WS10 | 4-8 (+3-6 review rounds, optional) |
| 2 | WS12 Coexistence and camera-to-display sample (G3) | ported `vpss` + DRM loaded together; sample exporting VB blocks as dma-buf fds (the `SYS_ION_ALLOC` fd) and page-flipping via PRIME; concurrent soak | WS7, WS11 | 1-3 |
| 3 (optional) | WS13 CSI-2 RX V4L2 sub-device + D-PHY RX driver | linear mode first; VC/DT HDR via streams later | WS7 or WS3 | 4-8 (+1-3) |
| 3 | WS14 VI raw capture, then full ISP V4L2 driver | raw/YUV capture node first (libcamera "simple" + software ISP; assumes a front-end-only DMA path to DRAM, unverified: if the back-end/post stages must be programmed this absorbs part of the ISP driver effort); then FE/BE/post scheduler, params/stats nodes on `v4l2-isp.h`, placement of the non-3A kernel-side functions (motion/DCI levels, FSWDR readout, LSC buffer export, black Y-curve and unity WB defaults), uAPI RFC | WS13 | 4-8, then 20-35 |
| 3 | WS15 VPSS as V4L2 m2m / resizer sub-devices | offline m2m first | WS9 | 4-8 (+6-12 online) |
| 3 | WS16 libcamera pipeline handler + IPA | option (a) prebuilt-core shim 4-8 (licence permitting, out-of-tree), (b) open 3A on the documented plugin contract 10-20, (c) from scratch 15-30 | WS14 | 4-30 |
| 3 | WS17 Sensor drivers | mainline sensor for bring-up; 2-4 per CVITEK-shipped sensor, assuming register-level datasheets are obtainable (vendor mode tables live in closed `libsns_*.so`; some datasheets are NDA-only) | WS13 | 0-8 |

### 9.3 Milestones, gates and totals

| Gate | Cumulative effort (eng-weeks) | Exit criteria | No-go trigger |
|---|---|---|---|
| G0 compile measurement | 3-6 | nine modules build and link; unknown symbols limited to the stub set; measured breakage within +/-20% of the inventory | structural ABI problem (e.g. packed ioctl structs with pointers under COMPAT) or breakage > 2x inventory |
| G0.5 allocator and cache (WS0-WS3) | 8-16 | VB pools on CMA; `SYS_ION_ALLOC` returns stable paddr + mmap-able fd; DMA round-trip bit-exact over 10,000 iterations on cached and uncached buffers incl. invalidate-after-mmap; no leak over 1,000 cycles; middleware allocation path confirmed | middleware needs behaviour the shim cannot offer without middleware changes |
| G1 camera on 7.x | 21-39 | vendor sample streams 1080p at >= 30 fps for 10 min with 3A converging; static-scene checksums stable; VPSS offline outputs bit-exact vs 5.10; dmesg clean | coherence corruption persists after 2 weeks of focused debugging, or sustained fps < 80% of baseline without cause |
| G2 display first picture (WS8-WS10; runs in parallel with WS4-WS7 when a second engineer is available) | 15-30 (G0.5 8-16 plus 7-15) | `modetest` pattern on panel or HDMI bridge; `/dev/fb0` console; vblank within 1%, pixel clock within 2%; write-1-to-clear bench result recorded | no HS link after 4 weeks of D-PHY work |
| G3 coexistence and clean DRM | 33-65 | camera + display together for 10 min with matching IRQ/frame counters; no SC_TOP clobber (read-back checks); PRIME preview >= 25 fps; igt basic tests pass; no fixed-address ioremap in the DRM driver | - |
| G4 Stage 3 funding | - | product requirement for V4L2/libcamera exists; Legal cleared the 3A option and header-derived code; quality metrics agreed | otherwise stop at G3 and budget recurring maintenance |

Critical path: WS0 -> WS1 -> WS2 (the single largest design item) -> WS5 (`vi`, largest module) -> WS7; WS3 runs
in parallel with WS1/WS2 and must finish before WS4-WS6. Display: WS8 can start after WS0, WS9/WS10 wait for WS3,
and the display chain rejoins at WS12. With one engineer everything is serial; with two (camera, display) calendar
time approaches the camera chain alone plus the merge, but hardware debugging on one board set is a shared
bottleneck.

Calendar translation (assumption: two engineers, one board set, no procurement or Legal delay): G0 at week 3-6,
G0.5 at week 8-16, G2 (display first picture) at week 15-30 on the display engineer, G1 at week 21-39, G3 at week
24-45, i.e. roughly 6-11 months to G3; one engineer serial: 33-65 weeks (8-15 months). Stage 3 adds roughly 9-24
months with the same team. Upstream review adds 6-18 months of overlapping calendar time, not effort.

Team and hardware assumptions: one engineer experienced in DMA-API/dma-buf and out-of-tree module porting; one
experienced in DRM atomic/DSI/PHY; a Milk-V Duo S for camera; for display a Duo S DTS variant that re-muxes the
MIPI_TX pads (losing SDIO WiFi), a LicheeRV Nano (SG2002) with a DSI panel, or a Duo Module 01 EVB with its
reported LT8912B bridge (unverified); a sensor supported by the vendor middleware (GC2083/GC2053 class) and, for Stage 3, one with a
mainline driver (imx219/ov5647 class); a scope or logic analyser. Effort excludes procurement, Legal turnaround
and upstream review latency.

### 9.4 What changes for userspace

- Stage 1: the vendor middleware runs unchanged against the preserved `/dev/cvi-*` ABIs. Visible changes: buffers
  come from a CMA pool instead of a fixed carveout (physical addresses still stable per buffer, no longer at a
  predictable base; `DEFAULT_MESH_PADDR 0x80000000` in `vo_sdk_layer.c` must be checked); no stray ION fds are
  installed into the ioctl caller for kernel-internal pool allocations; `SYS_CACHE_INVLD` semantics change
  (validated at G0.5); `/dev/mem` remains required unless WS5's mmap-offset option is taken; thread priorities
  differ; modules must be rebuilt per kernel (vermagic enforced); `saradc/rtc/wdt/pwm` move to IIO/sysfs and
  standard class devices; codec/IVE/RTOS paths unavailable until ported.
- Stage 2: `libvo`, `/dev/cvi-mipi-tx` and the `VO_SDK_*` ioctls stop working; panel init moves into DT and panel
  drivers; `/dev/fb0` comes from `drm_fbdev_dma` in XRGB8888; VPSS -> VO kernel binding is replaced by dma-buf
  export + DRM PRIME page flips in userspace; RGN overlays become DRM planes.
- Stage 3: camera userspace moves to libcamera; vendor sensor libraries are replaced by in-kernel sensor drivers.

### 9.5 Maintenance outlook

The Stage 1 camera modules stay out-of-tree with the vendor ABI indefinitely; between 5.10 and 7.3 the inventory
counted 44 breakage classes including three hard ones, so budget roughly 0.5-3 engineer-weeks per major kernel
bump plus a hardware regression run (planners' estimates 0.5-2 and 1-3), tracking the Armbian family and mining Sophgo's `bm1688` osdrv
branch for the vendor's own 6.x fixes. The DRM driver, PHY driver and syscon bindings become normal in-tree
maintenance once merged (review calendar months). The hybrid's `vpss` patches (regmap SC_TOP, masked IRQ) are
throwaway if Stage 3 replaces `vpss`.

---

## 10. Risk register

| # | Risk | Likelihood | Impact | Mitigation / where resolved |
|---|---|---|---|---|
| 1 | Middleware memory assumptions: `/dev/mem` mapping of CMA RAM and kernel-kzalloc'd tuning buffers (verified), `VB_BLK` handles as kernel pointers, cached/write-combine attributes | high (verified dependency) | high | State `STRICT_DEVMEM=n`/no lockdown as a precondition or implement the VI mmap-offset path in WS5; G0.5 tests |
| 2 | Cache coherence on the non-coherent C906: invalidate semantics change; userspace-driven cache ops by paddr; `ERRATA_THEAD` must be on | medium-high | high (silent corruption) | WS2 A/B switch, checksum soak at G0.5/G1, invalidate-after-mmap test case |
| 3 | Display register semantics known only from vendor code (D-PHY PLL constants, `0x03002840` bit 1, `CLK_CTRL0` bit 4, FIFO/shadow bits, bootloader pre-init) | medium | medium-high (first picture slips) | WS8 register map and bench plan before coding; HAL reused verbatim first; scope on D-PHY |
| 4 | SC_TOP page and `sc` IRQ shared between display and scaler; write-1-to-clear unverified; `CLK_CTRL0` clobbering of camera clock selects | medium | medium | WS9 syscons with bit-field writes; DRM sole owner first; fallback single-owner + notifier (+1-2 weeks) |
| 5 | Licensing: `osdrv` headers "All rights reserved" (no SPDX) as basis for in-kernel ISP/VPSS code; `cvi_mpi` no OSS licence; prebuilt 3A redistribution; codec firmware | medium | high for Stage 3 and for product images | Legal request at the decision gate; restrict early stages to TRM-documented blocks and unchanged vendor modules |
| 6 | Board availability: Duo S MIPI_TX pads used by SDIO; SG2002 VIP identity with SG2000 assumed | high (verified for Duo S) | medium (delay) | procure LicheeRV Nano or Duo Module 01 + bridge, or a Duo S display DTS variant |
| 7 | Stage 1 underestimated: `vi` is 24k LOC with 4 RT kthreads; loss of RT boost may cause frame drops | medium | medium | measure at G1 against baseline; `sched_set_fifo`, IRQ affinity |
| 8 | Volume of mechanical churn larger than inventoried (`-Wmissing-prototypes`, rebase diff of two years) | medium | low-medium | `-Werror` off first; size the rebase diff in WS0 |
| 9 | Perpetual API churn for out-of-tree camera modules; vermagic enforced | high | medium (recurring) | CI against linux-next; compat headers; pin to Armbian families |
| 10 | No open 3A for the ISP; image quality of a native stack below vendor tuning | high (Stage 3) | high (Stage 3 only) | three costed IPA options; measurable non-parity quality targets; do not fund Stage 3 to answer feasibility |
| 11 | Security posture of the retained ABI (paddrs, kernel pointers, raw GPIO numbers, `/dev/mem`) | high | low in a closed embedded image, high if exposed | document the trust model; restrict device permissions; do not ship on multi-user systems |
| 12 | Phase-1 evidence is from a two-year-stale snapshot; "no upstream series" rests on search snippets | medium | low-medium | rebase and re-grep in WS0; manual lore search |

---

## 11. Open questions to resolve before committing

Bench items (cheap on the vendor 5.10 image, decide display design):
- Is SC_TOP `INTR_STATUS` (0x0A080034) write-1-to-clear per bit?
- What do `0x03002840` bit 1 and VIP_SYS `CLK_CTRL0` bit 4 do? Does `clk-cv1800` expose enough control for the
  CAM0/CAM1 PLL power-down and the five MCLK rates (24/25/26/27/37.125 MHz)?
- Does pure MIPI (DSI TX, CSI RX) need any FMUX write, or are the pinmux pokes only for parallel/BT/TTL modes?
- Can the DSI MAC run non-burst or command mode (TRM says yes; vendor driver says no)?
- Does the vendor U-Boot/fsbl pre-initialise DISP/DSI (boot logo) on the target board?

Middleware and product:
- Which SDK/`cvi_mpi` version will be used; does it match the verification-pass findings (allocation via
  `SYS_ION_ALLOC`, `/dev/mem` mappings, `SYS_CACHE_INVLD` after cached mmap)?
- Is `/dev/mem` (no `STRICT_DEVMEM`, no lockdown) acceptable for the product's security posture?
- Does any `cvi_mpi` caller fail when `open("/dev/ion")` fails (`cvi_sys.c:831`)? If so, a no-op `/dev/ion` node
  (about half a week) is needed in Stage 1.
- Which core runs Linux: the RISC-V C906 (assumed throughout) or the Arm Cortex-A53 (not assessed)?
- Which sensor(s) and panel/bridge are the actual targets; do they have mainline drivers?
- Is the FreeRTOS fast-boot camera, ISP -> VPSS online mode, HDR or multi-sensor needed?
- Per-product CMA pool size (74 MiB vendor Duo S ION, 128 MiB Armbian default, 170 MiB quoted elsewhere); an
  RTOS-shared carveout cannot be `reusable` CMA.

Legal:
- Licence basis for header-derived in-kernel ISP/VPSS code; use/redistribution of `cvi_mpi`, prebuilt 3A, tuning
  files and codec firmware; linking prebuilt 3A into a libcamera IPA.

Ecosystem:
- Manual lore.kernel.org search for in-flight CV18xx CSI/ISP/DRM/DSI series.
- Whether the SG2002 (LicheeRV Nano) VIP blocks are identical to SG2000.
- Is the CSI-2 RX controller / D-PHY a licensed IP block (Cadence, Synopsys) whose mainline sub-device could be
  adapted, or CVITEK-proprietary? (`cif` carries no vendor identifiers; compare the TRM register names.)

---

## Appendix A: device-tree contract (from the vendor DTS)

Source: `scpcom/sophgo-sg200x-debian` `configs/common/dts/sg200x/sg200x_base.dtsi` (byte-identical to
`cv181x_base.dtsi`), `sg200x_riscv/sg200x_base_riscv.dtsi`, `sg200x_default_memmap.dtsi`, `configs/duos/dts/*.dts`,
`configs/licheervnano/dts/*.dts`, `memmap.py`, `settings.mk`. Caveat: the board files include `soph_*.dtsi` names
and labels (`&vo`, `&cvi_vo`, `&cvi_fb`) not defined in the repository copy of the base; the production base in the
vendor kernel branch may differ slightly (likely only labels). Clock IDs differ numerically between the vendor
header and mainline for almost every clock; translate by **name**. All reset indices used by the DTS are
numerically identical in both headers; 68-73 (DSIPHY, CSIPHY) have no mainline macro but work as raw numbers.

| Node (compatible) | reg | IRQ (vendor hwirq = `SOC_PERIPHERAL_IRQ(n-16)`) | clocks (vendor names -> mainline ids) | resets / other | Probe-code mismatches |
|---|---|---|---|---|---|
| `cvitek,sys` | none | - | - | - | none; forward-port adds `memory-region` |
| `cvitek,base` | 0x0A0C8000 size **0x20** ("vip_sys") | - | - | - | drivers use offsets to 0xe0 (CLK_RATIO 0x30-0x48, AXI 0x70, RESETS1 0xc0); `vo` declares the same block as 0xa0; use 0x100 |
| `cvitek,cif` | MAC0 0x0A0C2000/0x2000, WRAP 0x0A0D0000/0x1000, MAC1 0x0A0C4000/0x2000, MAC2 0x0A0C6000/0x2000, pad_ctrl 0x03001C30/0x30 | 26, 27 ("csi0", "csi1") | clk_cam0/1 -> CLK_CAM0/1 (147/148), clk_sys_2 -> CLK_SRC_VIP_SYS_2 (102), clk_mipimpll (3), clk_disppll (5), clk_fpll (2) | resets phy0/phy1/phy-apb0/phy-apb1 = 70/72/71/73; `snsr-reset` GPIOs (3 x `porta 2` on Duo S; one entry on LicheeRV Nano) | pad_ctrl entry never used (index quirk) and lies in the mainline pinctrl block |
| `cvitek,vi` | 0x0A000000/0x80000 | 24 ("isp") | clk_sys_0..3 -> CLK_SRC_VIP_SYS_0..3 (100-103), clk_axi -> 99, clk_csi_be -> 105, clk_raw -> 134, clk_isp_top -> 111, clk_csi_mac0/1/2 -> 106/107/108 (all mandatory) | `clock-freq-vip-sys1 = 300000000` (unused by `vi`) | none |
| `cvitek,vpss` | 0x0A080000/0x10000 (SCL_TOP + IMG + SC + DISP + BT + DSI MAC + CMDQ), 0x0A0D1000/0x100 (DSI D-PHY) | 25 ("sc") | clk_sys_0/1/2, clk_img_d/v -> 112/113, clk_sc_top -> 114, clk_sc_d/v1/v2/v3 -> 115-118 (optional) | `clock-freq-vip-sys1` | one `reg-names` entry for two regions (ignored) |
| `cvitek,vo` | 0x0A080000/0x10000, 0x0A0C8000/0xa0, 0x0A0D1000/0x100 | none | clk_disp -> 121, clk_dsi -> 122, clk_bt -> 120 (optional) | `reset-gpio porte 2`, `pwm-gpio porte 0`, `power-ct-gpio porte 1` (used by U-Boot) | all three regs are dead (`#if 0`) and duplicate `vpss`/`base` |
| `cvitek,mipi_tx` | none | none (VBAT IRQ code dead) | clk_disp, clk_dsi | same three GPIOs (Linux side); not loaded by the vendor init script | none |
| `cvitek,fb` | 0x0A088000/0x1000 (informational) | - | - | `memory-region = <&fb_reserved>` (5632 KiB) | none |
| `cvitek,dwa` | 0x0A0C0000/0x1000 | 28 ("dwa") | clk_sys_0..4 -> 100-104, clk_dwa -> 119 | | none |
| `cvitek,ive` | 0x0A0A0000/0x3100 | 97 | none (VIP_SYS bits via `base`) | | none |
| `cvitek,tpu` | tdma 0x0C100000/0x1000, tiu 0x0C101000/0x1000 | 75, 76 | clk_tpu_axi -> CLK_TPU (11), clk_tpu_fab (12) | RST_TDMA 7, RST_TPU 8, RST_TPUSYS 9 | Armbian already carries a mainline node |
| `cvitek,asic-vcodec` | h265 0x0B020000/0x10000, h264 0x0B010000/0x10000, vc_ctrl 0x0B030000/0x100, vc_sbm 0x0B058000/0x100, vc_addr_remap 0x0B050000/0x400 | 22, 21, 23 ("h265", "h264", "sbm") | 9 clocks, all with mainline ids (137-144, 154) | no resets (mainline has RST_H264C/H265C/VCSYS) | none |
| `cvitek,asic-jpeg` | 0x0B000000/0x300, vc_ctrl, vc_sbm | 20 | 7 clocks (145, 146, ...) | RST_JPEG 4 | IRQ fetched via `platform_get_resource(IORESOURCE_IRQ)` (deprecated; whether mainline still populates IRQ resources for OF platform devices was not verified) |
| `cvitek,rtos_cmdqu` | mailbox 0x01900000/0x1000 | 101 | - | | matches mainline `cv1800-mailbox` block |
| `cvitek,cvitek-ion` + `cvitek,carveout` | - | - | - | `memory-region = <&ion_reserved>` (size only, dynamically placed; 74 MiB Duo S, 63 MiB Nano) | replaced by a `shared-dma-pool` + `reusable` node |
| rtc, saradc, wdt (`snps,dw-wdt`), pwm x4, wiegand x3, mon, cooling | various | | | | all replaceable by mainline drivers; cooling compatible mismatch (`sophgo,cooling` vs `cvitek,cv181x-cooling`) |

Memory map (Duo S, 512 MiB): Linux `memory@80000000` size 0x1FE00000 (510 MiB), FreeRTOS 2 MiB at the top
(`no-map`), remoteproc vrings/buffer fixed at 0x8F528000-0x8F630000 (`no-map`), ION 74 MiB (dynamically placed by
Linux; bootloader/RTOS assume 0x9B400000 with codec bitstream and ISP fast-boot buffers inside), framebuffer
5632 KiB at 0x9AE80000. LicheeRV Nano (256 MiB): ION 63 MiB, same layout scaled. The vendor DT describes **no**
panel, sensor, CSI/DSI lane topology or pinmux (panel timing from build variables and middleware, sensors from
userspace libraries, pinmux from U-Boot and raw writes); a mainline port must author these from board schematics.

Sketch of a mainline-style `sg2000-vip.dtsi` (full version in `docs/port-assessment-evidence/phase2/dt-contract.txt`;
`sophgo,*` compatibles are placeholders, no such bindings exist yet):

```dts
reserved-memory {
	vip_cma: vip-pool { compatible = "shared-dma-pool"; reusable; size = <0x4a00000>; alignment = <0x400000>; };
	rtos_fw: rproc@9fe00000 { reg = <0x9fe00000 0x200000>; no-map; };   /* only if the C906L RTOS is used */
};
soc {
	vip_syscon: syscon@a0c8000 {           /* VIP_SYS: per-block resets @0x0/0xc0, clock gates/dividers @0x14-0x48 */
		compatible = "sophgo,sg2000-vip-syscon", "syscon", "simple-mfd";
		reg = <0x0a0c8000 0x100>;
		vip_rst: reset-controller { #reset-cells = <1>; };
		vip_clk: clock-controller { #clock-cells = <1>; };
	};
	isp: isp@a000000 { reg = <0x0a000000 0x80000>; interrupts = <SOC_PERIPHERAL_IRQ(8) IRQ_TYPE_LEVEL_HIGH>;
		clocks = <&clk CLK_SRC_VIP_SYS_0>, ... <&clk CLK_CSI_MAC2_VIP>; /* 11 clocks, names as vendor */
		memory-region = <&vip_cma>; };
	csi_phy: phy@a0d0000 { reg = <0x0a0d0000 0x1000>; resets = <&rst 70>, <&rst 71>, <&rst 72>, <&rst 73>;
		reset-names = "phy0", "phy-apb0", "phy1", "phy-apb1"; #phy-cells = <1>; };
	csi0: csi@a0c2000 { reg = <0x0a0c2000 0x2000>; interrupts = <SOC_PERIPHERAL_IRQ(10) IRQ_TYPE_LEVEL_HIGH>;
		clocks = <&clk CLK_CSI_MAC0_VIP>; phys = <&csi_phy 0>; ports { /* sensor in, isp out */ }; };
	vpss: scaler@a080000 { reg = <0x0a080000 0x8000>; interrupts = <SOC_PERIPHERAL_IRQ(9) IRQ_TYPE_LEVEL_HIGH>;
		clocks = /* 10 scaler clocks */; memory-region = <&vip_cma>; };
	disp: display-controller@a088000 { reg = <0x0a088000 0x2000>; interrupts = <SOC_PERIPHERAL_IRQ(9) IRQ_TYPE_LEVEL_HIGH>;
		clocks = <&clk CLK_DISP_VIP>, <&clk CLK_BT_VIP>, <&clk CLK_SC_TOP_VIP>; port { disp_out: endpoint { remote-endpoint = <&dsi_in>; }; }; };
	dsi: dsi@a08a000 { reg = <0x0a08a000 0x1000>; clocks = <&clk CLK_DSI_MAC_VIP>, <&clk CLK_DSI_ESC>;
		phys = <&dsi_phy>; ports { /* disp in, panel/bridge out */ }; };
	dsi_phy: phy@a0d1000 { reg = <0x0a0d1000 0x100>; resets = <&rst 68>; sophgo,vip-syscon = <&vip_syscon>; #phy-cells = <0>; };
	dwa: dewarp@a0c0000 { reg = <0x0a0c0000 0x1000>; interrupts = <SOC_PERIPHERAL_IRQ(12) IRQ_TYPE_LEVEL_HIGH>; };
};
```

For a forward-port the vendor compatibles are kept instead (`cvitek,base` with reg 0x0A0C8000/0x100, `cvitek,sys`
with `memory-region`, `cvitek,vi`, `cvitek,vpss` with both regions, `cvitek,cif` with four regions, `cvitek,dwa`,
`cvitek,rgn`), with the clock phandles renumbered to `sophgo,cv1800.h`. The SC_TOP interrupt sharing between
`disp` and `vpss` in the sketch is a placeholder for the WS9 design decision.

## Appendix B: evidence index and corrections log

Phase 1 reader reports (`docs/port-assessment-evidence/*.txt`): `sys-base`, `cif-snsr`, `vi-isp`, `vpss`,
`vo-mipitx-fb`, `codec-others`, `api-delta`, `vendor-deps`, `mainline-infra`, `ecosystem`; condensed in
`DIGEST.md`. Phase 2 outputs (`docs/port-assessment-evidence/phase2/`): three adversarial verdicts
(`verdict-isp-3a-userspace`, `verdict-ion-confined`, `verdict-vo-owns-nothing`), the device-tree contract
(`dt-contract`), three plans (`plan-forward-port`, `plan-upstream-native`, `plan-staged-hybrid`) and two
judgements (`judge-engineering-realism`, `judge-decision-fit`).

Corrections the verification pass made to the first-pass findings (all reflected above):
- "Vendor 3A is closed and in no repository" -> the middleware framework is public source (`cvi_mpi`); only the
  3A cores and tuning library are prebuilt, without a licence; a documented plugin API and sample algorithms exist.
- "Middleware allocates the ISP pool via `/dev/ion`" -> it uses the `sys` ioctl `SYS_ION_ALLOC`; `/dev/ion` is
  only opened (`cvi_sys.c:831`). An ION-compatibility device is unnecessary for allocation; whether callers tolerate
  the open() failing is unverified (bench item).
- "How userspace maps tuning buffers is unknown" -> verified: `/dev/mem`, for every buffer type; and the
  middleware invalidates immediately after each cached mmap.
- "Mainline riscv cannot block `/dev/mem`" (one planner) -> wrong: `STRICT_DEVMEM` is selectable on riscv,
  default off.
- "ION is confined to `sys.c`" -> API use is, but ION headers propagate through `sys.h` into nine modules and 12
  Makefiles; `tpu` dereferences ION structures.
- "The Armbian TPU patch is the recipe for the `sys.c` allocator" -> it is an importer; `sys.c` needs an exporter.
- "The only direct register pokes in `vo` are the VBAT registers" -> plus 52 pinmux writes and 10 full-register
  VIP_SYS `CLK_CTRL0` overwrites; `fb` pokes the DDR controller and VIP AXI priority; the VBAT IRQ is unwired.
- "Display HAL = `scaler.c:2249-4300` + `dsi_phy.c`" -> wider (section 4.1); 13 of 68 consumed exports and three
  statics are shared with the scaler path.
- "DSI MAC is burst-only" -> software restriction; the TRM lists command and non-burst modes.
- Plan arithmetic: Stage 1 is 18-33 (not 16-29), Stage 2 is 12-26 (not 12-24); cumulative G1 21-39, G3 33-65.
