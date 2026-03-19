# Sonix4 RFS — Kernel Migration Plan
## Kernel 4.14 → Linux 6.12 LTS + PREEMPT_RT

---

## Hardware

| Item | Details |
|---|---|
| **Module** | Enclustra Mars XU3 |
| **SoC** | AMD Zynq UltraScale+ MPSoC (ZU2CG / ZU2EG / ZU3EG) |
| **APU** | 4x ARM Cortex-A53 (ARM64), up to 1.5 GHz |
| **RPU** | 2x ARM Cortex-R5F (500 MHz, hard real-time capable) |
| **PL** | Xilinx FPGA fabric with custom IP |
| **Carrier board** | Custom (Sonix4) |
| **Build system** | Buildroot |

---

## Current BSP (4.14 era)

Provided by an external company. Components:

| File | Description |
|---|---|
| `fsbl.elf` | First Stage Boot Loader |
| `bl31.elf` | ARM Trusted Firmware (ATF) |
| `upfw.elf` | PMU firmware |
| `bitfile.bit` | FPGA bitstream for PL |
| `.dts` | Device Tree Source for the full system |

Boot image is assembled with Bootgen using a `.bif` file:

```
BOOT.BIN
  ├── fsbl.elf
  ├── upfw.elf (PMU firmware)
  ├── bitfile.bit      ← PL configured here, before Linux
  ├── bl31.elf (ATF)
  └── u-boot.elf
```

PL is fully configured before Linux starts — no runtime fpga-manager needed.

---

## Custom PL IP

- Custom AXI IP in the FPGA fabric
- **No custom kernel driver** — IP is accessed from userspace via **UIO (Userspace I/O)**
- Kernel config required: `CONFIG_UIO=y`, `CONFIG_UIO_PDRV_GENIRQ=y`
- DTS nodes use `compatible = "generic-uio"`

---

## Target: Linux 6.12 LTS + PREEMPT_RT

- Linux 6.12 is the current LTS release
- **PREEMPT_RT is fully mainlined in 6.12** — no out-of-tree RT patch series needed
- Base: Xilinx kernel fork `xlnx_rebase_v6.6` or mainline 6.12
- RT config fragment: `PREEMPT_RT=y`, `HZ_1000=y`, `NO_HZ_FULL=y`

---

## Migration Strategy

### Boot Components

| Component | Action |
|---|---|
| `bitfile.bit` | **Reuse as-is** — bitfile is kernel-independent |
| `fsbl.elf` | **Replace** with Enclustra BSP version for Mars XU3 |
| `bl31.elf` | **Replace** with updated ATF from Enclustra BSP |
| `upfw.elf` | **Replace** with updated PMU firmware from Enclustra BSP |
| `u-boot.elf` | **Rebuild** from Enclustra BSP (`xlnx_rebase_v2024.x`) |
| `.bif` | **Reuse / minor update** — format unchanged |

Enclustra BSP reference: `github.com/enclustra/buildroot-bsp`

### Device Tree

The vendor-provided `.dts` is the starting point. Needs audit for 6.x DT binding changes:

- PS peripheral bindings (UART, I2C, SPI, GEM Ethernet, etc.)
- PL IP nodes — update to `compatible = "generic-uio"` where needed
- Clock and interrupt controller bindings

DT hierarchy:
```
zynqmp.dtsi              ← AMD/Xilinx upstream (do not modify)
  └── mars-xu3.dtsi      ← Enclustra module layer (from Enclustra BSP)
        └── sonix4.dts   ← Custom carrier + PL IP (based on vendor DTS)
```

### Buildroot

- Update `BR2_LINUX_KERNEL_VERSION` to `6.12.x`
- Add RT kernel config fragment
- Use Enclustra XU3 board support as base
- Enable `BR2_PACKAGE_UIO_PRIV` or equivalent for userspace UIO tools if needed

---

## Planned Repository Structure

```
Sonix4RFS/
├── buildroot/
│   ├── configs/
│   │   └── sonix4_defconfig          ← Buildroot top-level config
│   └── board/sonix4/
│       ├── post-build.sh             ← copy kernel modules, DTB
│       └── rootfs-overlay/           ← custom files in RFS
│
├── kernel/
│   ├── configs/
│   │   └── sonix4_rt_fragment.config ← RT config fragment
│   │       (PREEMPT_RT, HZ_1000, UIO, CPU isolation)
│   └── dts/
│       └── sonix4.dts                ← custom carrier DT (from vendor, updated for 6.x)
│
├── boot/
│   ├── sonix4.bif                    ← Bootgen input file
│   └── README-bootbin.md             ← how to rebuild BOOT.BIN
│
└── docs/
    └── migration-4.14-to-6x-rt.md   ← step-by-step migration guide
```

---

## Work Phases

### Phase 1 — Baseline (Enclustra BSP)
- Clone `enclustra/buildroot-bsp`
- Verify Mars XU3 boots with Enclustra FSBL + new kernel (minimal rootfs)
- Confirm BOOT.BIN assembly with existing bitfile

### Phase 2 — Device Tree Audit
- Take vendor `.dts` as base
- Audit all bindings against Linux 6.12 DT documentation
- Update PL IP nodes to `generic-uio`
- Compile and boot-test updated DTB

### Phase 3 — Kernel 6.12 + PREEMPT_RT
- Configure Buildroot with 6.12 LTS
- Apply RT config fragment
- Verify UIO devices appear correctly (`/dev/uio*`)
- Boot full system

### Phase 4 — RT Validation
- Run `cyclictest` on APU under system load
- Define pass/fail latency criteria
- Tune: CPU isolation (`isolcpus`), tickless (`nohz_full`), IRQ affinity

### Phase 5 — Full RFS Integration
- Complete Buildroot package set
- Startup services (systemd or BusyBox init)
- Full regression test of all subsystems

---

## Outstanding Items (Next Steps)

1. **Share vendor DTS** — needed to audit PL IP nodes and carrier board peripherals
2. **Confirm RT latency target** — determines tuning approach
3. **Confirm RPU usage** — is the R5F used for any tasks today?
4. **Clone Enclustra BSP** and identify Mars XU3 support status for 6.x

---

## Key References

- Enclustra Buildroot BSP: `https://github.com/enclustra/buildroot-bsp`
- Xilinx kernel: `https://github.com/Xilinx/linux-xlnx`
- Xilinx ATF: `https://github.com/Xilinx/arm-trusted-firmware`
- Xilinx U-Boot: `https://github.com/Xilinx/u-boot-xlnx`
- ZU+ PREEMPT_RT: `https://wiki.linuxfoundation.org/realtime/start`
- UIO driver docs: `https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html`
- Mars XU3 product page: `https://www.enclustra.com/en/products/system-on-chip-modules/mars-xu3/`
