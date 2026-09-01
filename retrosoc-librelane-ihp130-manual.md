# 🛠️ retroSoC Full-Chip Implementation Manual
## LibreLane 3.0.5 + IHP SG13G2 (130 nm SiGe BiCMOS)

> **Repository:** `github.com/retroSoC/retroSoC` (fork: `chumnarn/retroSoC`)
> **Target PDK:** IHP SG13G2 — 130 nm SiGe BiCMOS, open-source preview
> **Flow:** Single-level pad-ring Chip flow (LibreLane 3.0.5, manual-PDK mode)
> **ผู้เขียนคู่มือ:** อาจารย์ผู้สอน ASIC Design Workshop
> **ฉบับ:** v1.0 — Step-by-step, ready-to-run
> **อัปเดตล่าสุด:** 2026-09-01

---

## 📑 สารบัญ

| # | Section | เนื้อหา |
|---|---------|---------|
| 1 | [Overview & Goal](#1-overview--goal) | ทำไมต้อง retroSoC + LibreLane + IHP SG13G2 |
| 2 | [Prerequisites](#2-prerequisites) | HW / OS / Network / Knowledge self-check |
| 3 | [Architecture Deep Dive](#3-architecture-deep-dive) | Hazard3, peripherals, memory map, pin map |
| 4 | [Environment Setup](#4-environment-setup) | Docker / Nix / Manual bootstrap |
| 5 | [Project Initialization](#5-project-initialization) | clone, `make setup`, `make doctor` |
| 6 | [Configuration Generation](#6-configuration-generation) | `config.json` + `retrosoc_asic.sdc` อัตโนมัติ |
| 7 | [Run LibreLane Flow](#7-run-librelane-flow) | `librelane-doctor` → `librelane-chip` → `librelane-package` |
| 8 | [Visualization](#8-visualization) | OpenROAD + KLayout viewers |
| 9 | [Output Structure & Artifacts](#9-output-structure--artifacts) | `runs/`, `final/`, `evidence/`, archive |
| 10 | [Verification — DRC, LVS, STA, Antenna](#10-verification--drc-lvs-sta-antenna) | อ่าน reports + ตีความผล |
| 11 | [Debug & Optimization Playbook](#11-debug--optimization-playbook) | แก้ปัญหาที่พบบ่อย |
| 12 | [Sign-off Checklist](#12-sign-off-checklist) | ก่อนส่ง MPW / Tape-out |
| 13 | [Troubleshooting Cookbook](#13-troubleshooting-cookbook) | per-error fix |
| 14 | [Resources & Next Steps](#14-resources--next-steps) | ลิงก์อ้างอิง, แหล่งเรียนรู้ |

---

## 1. Overview & Goal

### 1.1 retroSoC คืออะไร

**retroSoC** เป็น ASIC framework แบบ open-source ที่รวม **Hazard3 RISC-V CPU** (32-bit, RV32IM) เข้ากับ peripherals ครบชุด:

| Block | รายละเอียด |
|-------|-----------|
| **CPU Core** | Hazard3 RV32IM (Wren6991) + Debug Module |
| **Memory** | OPI/QSPI PSRAM controller, SDRAM controller, opipsram |
| **Connectivity** | USB2 (ULPI PHY), SDIO host, SPI-SD, I²C, I²S, UART |
| **Multimedia** | DVP camera input, WS2812 LED, Crypto (AES/SHA/RSA) |
| **System** | CLINT timer, DMA, SystemCtrl, GPIO 32-bit |
| **Buses** | AXI4-Lite, APB4, RIB, AHB-Lite bridge |

### 1.2 ทำไมต้อง IHP SG13G2 + LibreLane

| Tool/PDK | เหตุผล |
|----------|--------|
| **IHP SG13G2** | 130 nm SiGe BiCMOS — open-source PDK จาก IHP Microelectronics (Germany) |
| | preview/non-commercial license, มี LibreLane config ในตัว |
| | มี IO cells, stdcell, SRAM macros ครบ |
| **LibreLane 3.0.5** | hardened open-source RTL→GDSII flow (FOSSi Foundation) |
| | รองรับ IHP SG13G2 ผ่าน manual-PDK mode |
| | ใช้ Yosys (synthesis) + OpenROAD (PnR) + KLayout (DRC) + Netgen (LVS) |
| **Single-level pad-ring** | ไม่แยก core macro + chip-top — เป็น flat chip run |

### 1.3 What you'll get เมื่อจบ flow

```
✅ synthesizable netlist  (Yosys → gate-level Verilog)
✅ DEF + placed layout    (OpenROAD floorplan/place/CTS/route)
✅ GDSII                  (KLayout stream-out, 8000 µm × 8000 µm die)
✅ DRC-clean report       (IHP SG13G2 KLayout ruleset)
✅ LVS-clean report       (Netgen vs schematic)
✅ STA report             (OpenROAD timing across corners)
✅ Sign-off archive       (SHA256-verified tar.gz)
```

### 1.4 Runtime budget (เครื่อง 8-core, 32 GB RAM)

| Phase | Expected time |
|-------|---------------|
| `make setup` (first time, ดาวน์โหลด PDK + tools) | 15-30 นาที |
| `make librelane-doctor` | < 1 นาที |
| `make librelane-input` (export RTL+SDC) | 1-2 นาที |
| `make librelane-chip` (RTL→GDS) | **30 นาที – 2 ชั่วโมง** |
| `make librelane-package` | < 30 วินาที |

---

## 2. Prerequisites

### 2.1 Hardware

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | x86_64 4-core | x86_64 8-core+ (Apple Silicon ใช้ emulation ได้แต่ช้า) |
| RAM | 16 GB | 32 GB+ (chip run ใช้ memory มาก) |
| Disk | 30 GB free | 50 GB+ SSD (PDK + toolchain + build artifacts) |
| Network | broadband | ไม่จำกัด — ดาวน์โหลด PDK + tools |

> ⚠️ **Apple Silicon (M1/M2/M3):** ต้องใช้ `--platform linux/amd64` Docker emulation — จะช้ากว่า ~30-50%

### 2.2 Operating System

รองรับ Linux x86_64 เท่านั้น (Ubuntu 22.04 LTS เป็นหลัก):

| OS | วิธีที่แนะนำ |
|----|-------------|
| **Ubuntu 22.04 / 24.04** (native) | manual bootstrap |
| **Windows 11 + WSL2 (Ubuntu 22.04)** | manual หรือ Docker Desktop |
| **macOS 13+ (Intel)** | manual (ต้องติดตั้งเอง) |
| **macOS 13+ (Apple Silicon)** | Docker with `--platform linux/amd64` |
| **Docker / Podman บน Linux** | ตามสะดวก |

> ❌ **Windows native (ไม่ใช่ WSL2):** ไม่รองรับ — EDA tools เป็น ELF binaries

### 2.3 Knowledge Prerequisites

ก่อนเริ่ม ควรเข้าใจ:

- [ ] **SystemVerilog RTL** — module/port/always_ff/process
- [ ] **ASIC flow พื้นฐาน** — synth → floorplan → place → CTS → route → sign-off
- [ ] **LibreLane** — อ่าน Section 4 ของ workshop handbook ก่อน (ดู INDEX.md)
- [ ] **IHP SG13G2 PDK** — I/O, stdcell, SRAM macro, ข้อจำกัด
- [ ] **RISC-V RV32IM ISA** — โดยเฉพาะ CSR, IRQ, Debug

ถ้ายังไม่มั่นใจ → ดู **Section 14 Resources** ก่อนเริ่ม

### 2.4 Network Requirements

- เข้าถึง `github.com`, `hub.docker.com`, `github.com/retroSoC/...`
- เข้าถึง `github.com/IHP-GmbH/IHP-Open-PDK` (PDK)
- HTTPS เปิดใช้งาน (ไม่ต้อง SSH key — ใช้ HTTPS clone)

### 2.5 Pre-flight Check

```bash
# Linux x86_64?
uname -m          # ต้องได้: x86_64
uname -s          # ต้องได้: Linux

# Disk free
df -h .           # ต้องเหลือ ≥ 30 GB

# Memory
free -h           # ต้องเหลือ ≥ 8 GB (สำหรับ chip run)

# Tools พื้นฐาน
git --version     # ≥ 2.25
make --version    # ≥ 4.0
python3 --version # ≥ 3.10
curl --version
```

ถ้าทุกข้อผ่าน → ไป Section 4 (Environment Setup)

---

## 3. Architecture Deep Dive

### 3.1 SoC Block Diagram

```
                    +-------------------------------+
                    |        retrosoc_asic          |  ← top-level (synth target)
                    |   (pad-ring + power/IO pads) |
                    +---------------+---------------+
                                    |
        +-----------+-----+---------+--------+-------+----------+
        |           |     |         |        |       |          |
    +---+---+  +----+----+|  +------+------+ +-+-----+--+ +-----+-----+
    | RCU   |  |  GPIO  ||  |  retrosoc   | | USB2 ULPI| | Bondpads  |
    |       |  |  32-bit||  |  Hazard3 +  | | 12 pads  | | 70×70 µm  |
    +---+---+  +----+----+|  |  AXI/APB/   | +----+-----+ +-----------+
        |                |  |  RIB bus    |      |
        |                |  |  fabric     |      |
        |                |  +--+----+-----+      |
        |                |     |    |            |
        |                |     |    |            |
    +---+---+    +-------+--+ ++----+-------+   |
    | PLL   |    |  APB4   |  |  AXI4      |   |
    | (opt) |    |  Periph |  |  Memory    |   |
    +-------+    +---------+  |  Subsystem |   |
                             +------------+   |
                                              |
        +------------+-----------+------------+----------+----------+
        |            |           |            |          |          |
    +---+---+   +----+---+  +----+---+   +----+---+ +----+---+ +----+-----+
    | PSRAM  |   | SDRAM  |  |  DMA   |   |  UART  | |  I2C   | | SDIO1    |
    | Ctrl   |   |  Ctrl  |  |        |   |        | |        | |          |
    +--------+   +--------+  +--------+   +--------+ +--------+ +----------+
```

### 3.2 Top-level Module — `retrosoc_asic`

ไฟล์: `rtl/mini/top/retrosoc_asic.sv` (155 บรรทัด)

```systemverilog
module retrosoc_asic (
    `include "retrosoc_asic_ports.svh"   // 109 signal pads + clock + reset
);

  // 1. Internal clocks & resets
  logic s_ext_clk, s_aud_clk, s_sys_clk, s_aud_clk_buf;
  logic s_ext_rst_n, s_sys_rst_n, s_aud_rst_n;
  // ...

  // 2. Peripherals interfaces
  gpio_if          u_gpio_if();
  xpi_if           u_xpi_if();
  sdram_if         u_sdram_if();
  sdio_if          u_sdio1_if();
  usb2_ulpi_if     u_usb2_ulpi_if();
  pll_ctrl_if      u_pll_ctrl_if();

  // 3. Generated pad bindings (port ↔ internal signal)
  `include "retrosoc_asic_pad_bindings.svh"

  // 4. Reset & Clock Unit
  rcu u_rcu (
      .ext_clk_i      (s_ext_clk),
      .aud_clk_i      (s_aud_clk),
      .ext_rst_n_i    (s_ext_rst_n),
      .sys_clk_o      (s_sys_clk),
      .sys_rst_n_o    (s_sys_rst_n),
      // ...
  );

  // 5. SoC core
  retrosoc u_retrosoc (
      .clk_i          (s_sys_clk),
      .rst_n_i        (s_sys_rst_n),
      .clk_aud_i      (s_aud_clk_buf),
      // ...
      .gpio           (u_gpio_if),
      .sdram          (u_sdram_if),
      .sdio1          (u_sdio1_if),
      .usb2           (u_usb2_ulpi_if),
      // ...
  );
endmodule
```

**Key facts:**
- Top รับ **109 signal pads** (ext_clk, reset, GPIO×32, SDIO×6, USB2×12, JTAG×5, etc.)
- Pad ring มี **80 power/ground pads** (24 VDD + 24 VSS + 16 IOVDD + 16 IOVSS) + 4 corners
- ใช้ **`SYNTH_HIERARCHY_MODE = deferred_flatten`** เพื่อให้ Yosys flatten top เป็น single netlist

### 3.3 Pad-Ring Layout (Single-Level)

Pad ring แบ่งเป็น 4 ด้าน (รวม 189 instances):

| Side | Signal Pads | Power Pads | Total |
|------|-------------|-----------|-------|
| **South** | 30 | 30 (12 VDD/VSS + 8 IOVDD/IOVSS) | 30 |
| **East**  | 52 (GPIO) | 30 | 52 |
| **North** | 48 (SDIO/USB2/XPI) | 30 | 48 |
| **West**  | 59 (SDRAM) | 30 | 59 |

Side assignment logic (`generate_chip_config.py:51-59`):
```python
def signal_side(pad: Pad) -> str:
    if name.startswith("sdram_"): return "west"
    if name.startswith("gpio_"):  return "east"
    if name.startswith(("sdio1_", "usb2_", "xpi_")): return "north"
    return "south"   # clocks, reset, JTAG, ...
```

### 3.4 Memory Map (canonical)

ดูใน `rtl/mini/integration/` และ generated filelists. ตัวอย่าง:

| Region | Address | Size | Block |
|--------|---------|------|-------|
| Boot ROM | `0x0000_0000` | 256 KB | onchip (HAVE_SRAM_IF) |
| Main RAM | `0x8000_0000` | varies | PSRAM/SDRAM |
| APB4 peripherals | `0x4000_0000+` | 64 KB | UART, I2C, GPIO, ... |
| SystemCtrl | `0x4000_0000` | 4 KB | Reset, PLL, IRQ routing |
| CLINT | `0x4000_1000` | 4 KB | Timer + Software IRQ |

> 📖 ดู memory map เต็มใน `tests/test_memory_map.py` และ `docs/engineering.md`

### 3.5 Clock Domains

5 independent clock domains — ต้องแยกใน SDC:

| Domain | Source | Period (default) |
|--------|--------|------------------|
| `clk_external` | extclk_i_pad | 13.888 ns (72 MHz) |
| `clk_system`   | derived (÷1) | 13.888 ns |
| `clk_audio`    | audclk_i_pad | 54.253 ns (18.432 MHz) |
| `clk_jtag`     | jtag_tck_i_pad | 100 ns (10 MHz) |
| `clk_dvp`      | gpio_10_io_pad (ALT1) | 41.667 ns (24 MHz) |
| `clk_usb2_ulpi`| usb2_ulpi_clk_i_pad | 16.667 ns (60 MHz) |

Async groups ใช้ `set_clock_groups -asynchronous` ใน SDC

### 3.6 ทำไม Single-level Chip flow (ไม่ใช่ Classic)

จาก `physical/librelane/README.md`:

> This directory owns the open-source, single-level IHP130 pad-ring implementation for `retrosoc_asic`. It does **not** harden or deliver a separate digital-core macro.

**เหตุผล:**
1. IHP SG13G2 SRAM macros (`RM_IHPSG13_1P_4096x16_c3_bm_bist`) ฝังอยู่ใน single chip run
2. Pad ring ขนาด 8000×8000 µm ต้องการ full-chip integration
3. Classic flow (`Classic` flow ของ LibreLane) เหมาะกับ hardened core + แยก chip-top

---

## 4. Environment Setup

เลือกวิธีใดวิธีหนึ่ง — ทั้ง 3 วิธีใช้ lock-pinned versions เดียวกัน (`dependencies/dependencies.lock.json`)

### 4.1 Option A: Docker (แนะนำ — สะดวกที่สุด)

**ข้อดี:** idempotent, reproducible, ไม่กระทบ host system
**ข้อเสีย:** ต้องมี Docker, บน macOS ช้ากว่าเล็กน้อย

```bash
# 1. Clone retroSoC
cd ~
git clone https://github.com/chumnarn/retroSoC.git
cd retroSoC

# 2. Build Docker image (ใช้เวลา 5-10 นาที)
docker build --tag retrosoc-dev --file docker/Dockerfile .

# 3. Run container + mount current directory
docker run --rm --init --platform linux/amd64 \
  --user "$(id -u):$(id -g)" \
  -v "$PWD:/workspace/retrosoc" \
  -it retrosoc-dev bash

# ตอนนี้คุณอยู่ใน container shell
cd /workspace/retrosoc
ls    # ควรเห็น Makefile, rtl/, physical/, ...
```

**Apple Silicon note:** ใช้ `--platform linux/amd64` เสมอ (ตามตัวอย่างข้างบน)

### 4.2 Option B: Nix (Linux x86_64 only)

**ข้อดี:** pure functional, lock-pinned nixpkgs
**ข้อเสีย:** ต้องติดตั้ง Nix ก่อน, Linux x86_64 เท่านั้น

```bash
# 1. ติดตั้ง Nix (single-user, deterministic)
sh <(curl -L https://nixos.org/nix/install) --no-daemon
. ~/.nix-profile/etc/profile.d/nix.sh

# 2. Clone retroSoC
cd ~
git clone https://github.com/chumnarn/retroSoC.git
cd retroSoC

# 3. เข้า Nix shell (จะ bootstrap tools อัตโนมัติ — ใช้เวลา 5-15 นาทีแรก)
nix run .#dev
```

หลังจากนี้ ทุกครั้งที่จะใช้งาน:
```bash
cd ~/retroSoC
nix run .#dev -- make CONFIG=configs/ci/ihp130.mk librelane-doctor
```

### 4.3 Option C: Manual (Ubuntu 22.04 / WSL2)

**ข้อดี:** เข้าใจทุกขั้นตอน, ไม่ต้อง Docker
**ข้อเสีย:** ต้องติดตั้งเอง, ใช้เวลานาน

```bash
# 1. ติดตั้ง system packages
sudo apt-get update
sudo apt-get install --no-install-recommends -y \
    build-essential git make python3 python3-pip python3-venv \
    bzip2 ca-certificates ccache curl g++ libfl2 \
    libgoogle-perftools4 libunwind8 mold numactl \
    xz-utils zlib1g

# 2. Clone retroSoC
cd ~
git clone https://github.com/chumnarn/retroSoC.git
cd retroSoC

# 3. Bootstrap development environment (ดาวน์โหลด + verify tools)
python3 scripts/development_environment.py --root "$PWD" bootstrap

# 4. Activate environment (ทุกครั้งที่เปิด shell ใหม่)
source .cache/retrosoc/development/activate.sh

# 5. Verify
python3 scripts/development_environment.py check
```

### 4.4 เปรียบเทียบ 3 วิธี

| Item | Docker | Nix | Manual |
|------|--------|-----|--------|
| Setup time (first) | 5-10 นาที | 5-15 นาที | 15-30 นาที |
| Disk usage | 5-8 GB image | 3-5 GB store | 3-5 GB ใน `~/.cache/retrosoc` |
| ต้องติดตั้ง system packages ไหม | ไม่ | Nix daemon | ใช่ (apt) |
| Reproducible | ✅ (OCI digest) | ✅ (flake.lock) | ✅ (lock + checksums) |
| ใช้ได้บน macOS | ✅ (ช้า) | ❌ | ⚠️ (ต้องติดตั้งเอง) |
| ใช้ได้บน Windows | ✅ (Docker Desktop + WSL2) | ❌ | ✅ (WSL2 only) |

**คำแนะนำ:**
- **ผู้เริ่มต้น / Windows:** ใช้ Docker
- **Linux native + ชอบ reproducible:** ใช้ Nix
- **CI / production / production-class run:** ใช้ Manual

---

## 5. Project Initialization

### 5.1 Clone และ Verify

```bash
cd ~
git clone https://github.com/chumnarn/retroSoC.git
cd retroSoC

# ตรวจสอบ lock digest (ถ้าใช้ Docker/Nix ไม่ต้อง verify)
git log --oneline -5
cat dependencies/dependencies.lock.json | python3 -m json.tool | head -20
```

### 5.2 Install External Dependencies (PDK, MPW, IP)

```bash
# ใช้เวลา 5-15 นาที (ดาวน์โหลด ~500 MB)
# - IHP-Open-PDK (recursive, 130 nm PDK)
# - mini-ver-mpw (Hazard3 + managed IP)
# - archinfo, common, crc, pwm, rtc, rng, wdg (Cluster IPs)
make setup
```

Expected output (ตัวอย่าง):
```
[setup-mpw]   cloning https://github.com/retroSoC/mini-ver-mpw @ e63d5cd7...
[setup-clusterip]  installing archinfo, common, crc, pwm, ps2, rng, rtc, wdg
[setup-ip]    installing self-owned IP wrappers
[setup-pdk]   cloning https://github.com/IHP-GmbH/IHP-Open-PDK @ 970a7688...
[setup-app]   installing application (bringup, ci_smoke, coremark, ...)
```

### 5.3 Environment Doctor

```bash
# ตรวจสอบ tools + paths + configuration
make CONFIG=configs/ci/ihp130.mk doctor
```

Expected output (success):
```
[PASS] verilator 5.046 at /opt/retrosoc/.../bin/verilator
[PASS] yosys 0.67 at ...
[PASS] opensta 2.2.0-1767-ga93b51ec at ...
[PASS] iverilog 13.0 at ...
[PASS] sv2v 0.0.13-9-gd381209 at ...
[PASS] sby 0.67 at ...
[PASS] bitwuzla 0.9.1 at ...
[PASS] riscv32-unknown-elf-gcc at ...
[PASS] IHP-Open-PDK @ 970a7688e7dcce2a...
```

ถ้า fail → ดู Section 13 Troubleshooting

### 5.4 ดู Configuration ที่เลือก

```bash
make CONFIG=configs/ci/ihp130.mk config
```

Expected output:
```
ROOT_PATH         /workspace/retrosoc
CONFIG            /workspace/retrosoc/configs/ci/ihp130.mk
BUILD_TIMESTAMP   2026-09-01-14-30
VARIANT_ID        ihp130-2026-09-01-14-30-abc12345
SOC               MINI        MGMT_CORE    HAZARD3
SIMU              VCS         SYNTH        NONE
PDK               IHP130      HAVE_PLL     NO
HAVE_SRAM_IF      NO          HAVE_SRAM_MACRO  NO
EXT_CLK_HZ        72000000    AUD_CLK_HZ   18432000
ISA               RV32IM      APP          bringup
LINK_TYPE         ld2_psram
```

### 5.5 (Optional) ทดสอบ RTL ก่อน Chip flow

```bash
# Compile + simulate bringup firmware (ใช้เวลา 3-5 นาที)
make CONFIG=configs/ci/ihp130.mk SIMU=IVERILOG sim
```

Expected: `SIM_TEST_PASS` ใน log

---

## 6. Configuration Generation

retroSoC generate **2 ไฟล์สำคัญ** ให้อัตโนมัติ:

1. `config.json` — LibreLane chip configuration
2. `retrosoc_asic.sdc` — pad-aware multi-clock SDC

### 6.1 เมื่อไหร่ต้อง regenerate

- เปลี่ยน `pin_map.json` (เพิ่ม/ลด pads)
- เปลี่ยน EXT_CLK_HZ หรือ AUD_CLK_HZ
- เปลี่ยน PDK macros (SRAM instances)
- เปลี่ยน HAVE_PLL flag

**ไม่ต้อง regenerate** ถ้าแค่:
- เปลี่ยน synthesis options
- เปลี่ยน PnR density
- เปลี่ยน flow steps

### 6.2 Generate อัตโนมัติ (ผ่าน Makefile)

```bash
# Generate ทั้ง config.json + SDC + bondpad GDS
make CONFIG=configs/ci/ihp130.mk librelane-input
```

จะสร้าง:
```
build/ihp130-2026-09-01-14-30-abc12345/physical/librelane/chip/
├── input/
│   ├── retrosoc_asic_sources.sv     ← flattened SystemVerilog
│   ├── retrosoc_asic.sdc            ← SDC พร้อม 5 clock domains
│   ├── bondpad_70x70.gds            ← 70µm bondpad
│   └── ...
└── config.json                      ← LibreLane config
```

### 6.3 Generate แบบ manual (debug)

ถ้าอยากรู้ว่า generator ทำอะไรบ้าง:

```bash
# ดู generate_chip_config.py
cat physical/librelane/scripts/generate_chip_config.py | head -100

# ดู generate_sdc.py
cat physical/librelane/scripts/generate_sdc.py | head -80
```

### 6.4 ทำความเข้าใจ `config.json` (ตัวอย่าง output)

```json
{
  "meta": {"version": 3, "flow": "Chip"},
  "DESIGN_NAME": "retrosoc_asic",
  "VERILOG_FILES": ["dir::input/retrosoc_asic_sources.sv"],
  "USE_SLANG": true,                       ← ใช้ Slang (เร็วกว่า Surelog)
  "SLANG_ARGUMENTS": ["--keep-hierarchy"],
  "SYNTH_HIERARCHY_MODE": "deferred_flatten",
  "PAD_SOUTH": ["vdd_pads[0].vdd_pad", "u_extclk_i_pad.u_sg13g2_IOPadIn", ...],
  "PAD_EAST":  ["u_gpio_00_io_pad.u_sg13g2_IOPadInOut4mA", ...],
  "PAD_NORTH": ["u_sdio1_clk_o_pad.u_sg13g2_IOPadOut4mA", ...],
  "PAD_WEST":  ["u_sdram_dq15_io_pad.u_sg13g2_IOPadInOut4mA", ...],
  "DIE_AREA": [0, 0, 8000, 8000],          ← 8 × 8 mm
  "CORE_AREA": [365, 365, 7635, 7635],     ← 365 µm seal-ring offset
  "PL_TARGET_DENSITY_PCT": 30,
  "CLOCK_PORT": ["extclk_i_pad", "audclk_i_pad", "jtag_tck_i_pad",
                 "gpio_10_io_pad", "usb2_ulpi_clk_i_pad"],
  "CLOCK_PERIOD": 13.888888888889,         ← 1/72 MHz
  "VDD_NETS": ["VDD", "IOVDD"],
  "GND_NETS": ["VSS", "IOVSS"],
  "MACROS": {
    "RM_IHPSG13_1P_4096x16_c3_bm_bist": {  ← USB packet SRAM
      "gds": [...], "lef": [...], "vh": [...],
      "instances": { ... }
    }
  }
}
```

### 6.5 ทำความเข้าใจ SDC (ตัวอย่าง output)

```tcl
# Generated by physical/librelane/scripts/generate_sdc.py; do not edit.
current_design $::env(DESIGN_NAME)
set_units -time ns

# 1. Create 5 clock domains
set clk_external_pin [require_pins "clock external" {u_extclk_i_pad.u_sg13g2_IOPadIn/p2c}]
create_clock -name clk_external -period 13.888888888889 $clk_external_pin

set clk_audio_pin [require_pins "clock audio" {u_audclk_i_pad.u_sg13g2_IOPadIn/p2c}]
create_clock -name clk_audio -period 54.253472222222 $clk_audio_pin

# 2. Generated clock (system = external ÷1)
set clk_system_pin [require_pins "clock system" {u_rcu.s_sys_clk}]
create_generated_clock -name clk_system -source $clk_external_pin \
    -divide_by 1 $clk_system_pin

# 3. Async clock groups
set_clock_groups -name retrosoc_async -asynchronous \
  -group [get_clocks {clk_external clk_system}] \
  -group [get_clocks {clk_audio}] \
  -group [get_clocks {clk_jtag}] \
  -group [get_clocks {clk_dvp}] \
  -group [get_clocks {clk_usb2_ulpi}]

# 4. I/O delays (per-domain, 20% of period)
set_input_delay -min 0     -clock clk_external {ext_rst_n_i_pad jtag_trst_n_i_pad ...}
set_input_delay -max 2.778 -clock clk_external {ext_rst_n_i_pad jtag_trst_n_i_pad ...}

# 5. Reset false paths
set_false_path -from $reset_ext_rst_n_i_pad
set_false_path -from $reset_jtag_trst_n_i_pad
```

---

## 7. Run LibreLane Flow

### 7.1 Workflow Overview

```
make librelane-doctor    → ตรวจ env + config (1 นาที)
make librelane-input     → generate config.json + SDC + bondpad (1-2 นาที)
make librelane-chip      → RTL → GDS (30 นาที - 2 ชั่วโมง) ⭐ MAIN
make librelane-openroad  → เปิด OpenROAD GUI (optional)
make librelane-klayout   → เปิด KLayout (optional)
make librelane-package   → สร้าง deliverable tar.gz
```

### 7.2 Step 1: Doctor (validate environment)

```bash
make CONFIG=configs/ci/ihp130.mk librelane-doctor
```

Expected output (success):
```
[librelane-check-pdk]  PDK=IHP130 OK
[generate-sdc]         wrote .../input/retrosoc_asic.sdc
[generate-chip-config] wrote .../config.json
[bondpad-gds]          wrote .../input/bondpad_70x70.gds
[librelane-doctor]     loading LibreLane v3.0.5...
[librelane-doctor]     config DESIGN_NAME = retrosoc_asic
[librelane-doctor]     KLAYOUT_DRC_RUNSET = .../ihp-sg13g2.drc
[librelane-doctor]     NETGEN_SETUP = .../netgen_setup.tcl
[librelane-doctor]     PAD_SPICE_MODELS = [.../sg13g2_io.spice]
[librelane-doctor]     189 placed PADs verified
[librelane-doctor]     PASSED ✓
```

ถ้า fail:
- `LibreLane v3.x not found` → ติดตั้ง LibreLane 3.0.5 (`pip install librelane==3.0.5`)
- `IHP PDK checkout missing` → รัน `make setup-pdk`
- `KLayout not found` → `apt install klayout` (Ubuntu) หรือ activate environment

### 7.3 Step 2: Generate Inputs (optional — chip step จะทำให้อัตโนมัติ)

```bash
make CONFIG=configs/ci/ihp130.mk librelane-input
```

### 7.4 Step 3: Run Full Chip Flow ⭐

```bash
make CONFIG=configs/ci/ihp130.mk librelane-chip
```

**ใช้เวลา:** 30 นาที (เครื่อง 8-core + SSD) – 2 ชั่วโมง (เครื่อง 4-core + HDD)

Expected log highlights (เรียงตามลำดับ):
```
[09:14:01] [INFO] Loading configuration retrosoc_asic ...
[09:14:02] [STEP 1/22] yosys.synthesis
            Synthesizing retrosoc_asic with Slang+Yosys
            ... (5-10 นาที)
            [SUCCESS] yosys.synthesis

[09:25:00] [STEP 2/22] openroad.floorplan
            Die area: 8000 x 8000
            Core area: 7270 x 7270
            ... (2-3 นาที)

[09:28:00] [STEP 3/22] openroad.tapendcap

[09:30:00] [STEP 4/22] openroad.pdn
            PDN grid: stdcell_grid (Metal1-Metal5)

[09:32:00] [STEP 5/22] openroad.ioplacement
            Placing 189 PADs on 4 sides

[09:35:00] [STEP 6/22] openroad.globalplacement
[09:40:00] [STEP 7/22] openroad.detailedplacement

[09:45:00] [STEP 8/22] openroad.cts
            TritonCTS: 0.18 ns slack
[09:50:00] [STEP 9/22] openroad.globalrouting
[10:00:00] [STEP 10/22] openroad.detailedrouting
[10:10:00] [STEP 11/22] openroad.filler

[10:12:00] [STEP 12/22] openroad.sta
            Worst slack: +0.42 ns (PASS)
[10:13:00] [STEP 13/22] openroad.power
            Internal: 12.3 mW, Switching: 5.1 mW, Leakage: 0.05 mW

[10:14:00] [STEP 14/22] klayout.drc
            0 DRC errors ✓

[10:18:00] [STEP 15/22] klayout.lvs
            Netlist matches schematic ✓

[10:20:00] [STEP 16/22] klayout.xtap

[10:21:00] [STEP 17/22] klayout.gds
            GDS-II written: 8000 x 8000 µm

[10:22:00] [STEP 18/22] magic.drc
[10:24:00] [STEP 19/22] netgen.lvs
[10:25:00] [STEP 20/22] cvc.rv

[10:26:00] [INFO] Flow complete
            Final GDS-II: build/.../final/gds/retrosoc_asic.gds
            Total time: 1h 12m
```

> 💡 **Tip:** ถ้าต้องการรัน background เพื่อ log เก็บไว้ดู:
> ```bash
> make CONFIG=configs/ci/ihp130.mk librelane-chip 2>&1 | tee chip-run.log
> ```

### 7.5 Resume Flow (ถ้า fail กลางทาง)

```bash
# ใช้ --resume --last-run (LibreLane 3.0.5)
cd build/ihp130-2026-09-01-14-30-abc12345/physical/librelane/chip
librelane config.json --manual-pdk --pdk ihp-sg13g2 \
  --pdk-root physical/pdk/IHP-Open-PDK \
  --scl sg13g2_stdcell --pad sg13g2_io \
  --run-tag current --resume --last-run
```

หรือใช้ Make:
```bash
# ลบ failed step แล้ว rerun (LibreLane cache aware)
make CONFIG=configs/ci/ihp130.mk librelane-chip
```

### 7.6 Run Step-by-Step (debug)

ถ้าอยากเห็นแต่ละ step แยก:

```bash
cd build/ihp130-.../physical/librelane/chip

# เฉพาะ synthesis
librelane config.json --manual-pdk --pdk ihp-sg13g2 \
  --pdk-root ... --scl sg13g2_stdcell --pad sg13g2_io \
  --from yosys.synthesis --to yosys.synthesis

# เฉพาะ floorplan
librelane config.json ... --from openroad.floorplan --to openroad.floorplan

# ตั้งแต่ detailed routing เป็นต้นไป
librelane config.json ... --from openroad.detailedrouting
```

### 7.7 Package Deliverable

```bash
make CONFIG=configs/ci/ihp130.mk librelane-package
```

Output: `build/.../physical/librelane/chip/retrosoc-ihp130-chip.tar.gz`

ภายใน archive:
```
retrosoc-ihp130-chip/
├── SHA256SUMS                        ← hash ของทุกไฟล์
├── final/
│   ├── gds/retrosoc_asic.gds         ← 🎯 FINAL GDS
│   ├── def/retrosoc_asic.def
│   ├── lef/retrosoc_asic.lef
│   ├── lib/                          ← post-route liberty (approx)
│   ├── net/retrosoc_asic.v           ← post-route netlist
│   ├── sdf/retrosoc_asic.sdf         ← SDF for post-layout sim
│   ├── spef/                         ← parasitic
│   ├── odb/                          ← OpenDB
│   └── ... (klayout/rcx/...)
└── evidence/
    ├── run/                          ← LibreLane run DB
    ├── config.json
    ├── result.json                   ← step results
    ├── manifest.json
    ├── librelane-doctor.json
    ├── librelane-chip.json
    └── dependencies.lock.json
```

---

## 8. Visualization

### 8.1 Open OpenROAD GUI (debug / inspection)

```bash
make CONFIG=configs/ci/ihp130.mk librelane-openroad
```

เปิด OpenROAD GUI พร้อม load design ปัจจุบัน → ดู:
- Floorplan, placement, routing
- Congestion map
- Timing paths
- Power distribution

### 8.2 Open KLayout (final GDS)

```bash
make CONFIG=configs/ci/ihp130.mk librelane-klayout
```

เปิด KLayout พร้อม load final GDS → ดู:
- Metal layers (M1-M5, TopMetal1, TopMetal2)
- All standard cells, macros, pads
- Bondpads, seal ring
- DRC errors (ถ้ามี — จะแสดง marker)

### 8.3 Take Screenshots for Report

ใน KLayout:
- **File → Load → `final/gds/retrosoc_asic.gds`**
- Zoom ให้เห็นทั้ง chip → **Edit → Copy Screenshot** → save as PNG
- Layer display: enable TopMetal1 + TopMetal2 (signal routing) และ Metal1 (PDN)

### 8.4 OpenROAD Scripted Inspection

```bash
# ใน OpenROAD GUI
openroad> read_def final/def/retrosoc_asic.def
openroad> read_verilog final/net/retrosoc_asic.v
openroad> read_spef  final/spef/retrosoc_asic.spef

# ดู congestion
openroad> global_routing_congestion -overflow

# ดู critical path
openroad> report_timing -delay_type max -max_paths 5
```

---

## 9. Output Structure & Artifacts

### 9.1 Build Variant Layout

```
build/<PROFILE>-<TIMESTAMP>-<CONFIG_HASH>/
├── sw/                                          ← firmware (rv32i ELF)
├── generated/
│   ├── mpw/<simulator>/                         ← generated MPW sources
│   ├── archinfo/                                ← generated archinfo
│   └── pin_map/                                 ← generated pad bindings
├── physical/
│   └── librelane/chip/                          ⭐ LibreLane run directory
│       ├── input/
│       │   ├── retrosoc_asic_sources.sv
│       │   ├── retrosoc_asic.sdc
│       │   ├── bondpad_70x70.gds
│       │   └── sram_blackboxes.vh
│       ├── config.json                          ← generated by us
│       ├── librelane.log                         ← full flow log
│       ├── result.json                           ← step-level results
│       ├── runs/current/                         ← LibreLane run DB
│       └── final/                                ⭐ FINAL ARTIFACTS
│           ├── gds/retrosoc_asic.gds
│           ├── def/retrosoc_asic.def
│           ├── lef/retrosoc_asic.lef
│           ├── net/retrosoc_asic.v
│           ├── sdf/retrosoc_asic.sdf
│           ├── spef/...
│           ├── odb/...
│           ├── klayout/...
│           ├── reports/                          ⭐ SIGN-OFF REPORTS
│           │   ├── synthesis/
│           │   │   ├── synthesis.stat
│           │   │   ├── synthesis.chk
│           │   │   └── synthesis.json
│           │   ├── placement/
│           │   ├── cts/
│           │   ├── routing/
│           │   ├── sta/
│           │   │   ├── cts.max.rpt
│           │   │   ├── cts.min.rpt
│           │   │   ├── route.max.rpt
│           │   │   ├── route.min.rpt
│           │   │   └── ...
│           │   ├── drc/
│           │   │   ├── klayout_drc.lyrdb
│           │   │   └── magic_drc.rep
│           │   ├── lvs/
│           │   │   ├── netgen_lvs.rpt
│           │   │   └── ...
│           │   └── power/
│           │       └── power.rpt
│           └── retrosoc-ihp130-chip.tar.gz       ← (after librelane-package)
└── meta/
    ├── manifest.json                             ← build manifest
    ├── librelane-doctor.json
    ├── librelane-chip.json
    └── warnings.json
```

### 9.2 Final GDS — Key Numbers

| Metric | Typical (retroSoC + IHP SG13G2) |
|--------|-------------------------------|
| Die size | 8000 × 8000 µm² (64 mm²) |
| Core area | 7270 × 7270 µm² (~52.8 mm²) |
| Cell count | ~250k – 400k |
| Standard cell area | ~5 – 8 mm² |
| Macro area (SRAM + bondpads) | ~3 – 5 mm² |
| Routing utilization | 50-70% |
| Wire length (estimated) | ~25 m |

### 9.3 Final Netlist + SDF

```bash
# Final gate-level netlist (post-route)
cat build/.../final/net/retrosoc_asic.v | head -30

# SDF (Standard Delay Format) for post-layout simulation
cat build/.../final/sdf/retrosoc_asic.sdf | head -10
```

---

## 10. Verification — DRC, LVS, STA, Antenna

### 10.1 DRC (Design Rule Check)

**KLayout DRC** (IHP SG13G2 native ruleset):

```bash
# Report path
ls build/.../final/reports/drc/
cat build/.../final/reports/drc/klayout_drc.json
```

Expected: `"DRC errors: 0"`

ถ้าเจอ errors:
- **Metal spacing:** เพิ่ม routing spacing, ลด congestion
- **Antenna:** เพิ่ม antenna diodes (IHP SG13G2 มี `sg13g2_antenna_*` cells)
- **Min area:** check `density` reports + ปรับ `FP_CORE_UTIL` ลง

### 10.2 LVS (Layout vs Schematic)

**Netgen** (schematic vs layout):

```bash
cat build/.../final/reports/lvs/netgen_lvs.rpt
```

Expected:
```
Netlists match uniquely.
Cells: 12345  NAND2_X1, ...
```

ถ้าไม่ match:
- Check power/ground nets (`VDD`, `VSS`, `IOVDD`, `IOVSS`)
- Check SRAM macro power pins (`VDDARRAY!`, `VSS!`)
- Check pad cell pin names

### 10.3 STA (Static Timing Analysis)

**OpenROAD STA** (per-corner):

```bash
ls build/.../final/reports/sta/
cat build/.../final/reports/sta/cts.max.rpt
cat build/.../final/reports/sta/route.max.rpt
```

Key metrics:
- **WNS (Worst Negative Slack):** ต้อง ≥ 0 ns
- **TNS (Total Negative Slack):** ต้อง = 0 ns
- **Max frequency:** `1 / (period - WNS)`

ตัวอย่าง:
```
Path: u_retrosoc.u_hazard3.alu_result → u_apb4_uart.u_tx_fifo
Startpoint: clk_system (rise)
Endpoint: u_tx_fifo.rdata_d (rise)
Path type: max
Slack:  +0.42 ns
Required: 13.888 ns
Arrival: 13.468 ns
```

ถ้า WNS < 0:
- ลด `CLOCK_PERIOD` → ลด clock frequency
- เพิ่ม `PL_TARGET_DENSITY_PCT` → ลด density
- เปิด `PL_ROUTABILITY_DRIVEN` → driven placement
- เพิ่ม `SYNTH_STRATEGY` area=2 หรือ 3 (ดู Section 11)

### 10.4 Antenna Check

```bash
cat build/.../final/reports/lvs/antenna.rpt
```

ถ้า antenna violations:
- LibreLane แก้ให้อัตโนมัติ (`RUN_ANTENNA_REPAIR=1`)
- เพิ่ม `ANTENNA_RATIO_THRESHOLD` ถ้า PDK อนุญาต

### 10.5 Power Analysis

```bash
cat build/.../final/reports/power/power.rpt
```

ตัวอย่าง output:
```
Total Power: 17.45 mW
  Internal:  12.30 mW (70.5%)
  Switching:  5.10 mW (29.2%)
  Leakage:    0.05 mW (0.3%)
```

---

## 11. Debug & Optimization Playbook

### 11.1 Common Errors

#### 11.1.1 `ERROR (OpenROAD): placement density overflow`

**อาการ:** Global placement fail — cells แน่นเกินไป
**แก้:**
```json
// ใน generate_chip_config.py — เพิ่ม PL_TARGET_DENSITY_PCT ลง
"PL_TARGET_DENSITY_PCT": 35   // (default 30)  → 35
// หรือเพิ่ม DIE_AREA
"DIE_AREA": [0, 0, 9000, 9000]  // 9×9 mm
```

#### 11.1.2 `ERROR: Clock period not met (WNS = -1.23 ns)`

**อาการ:** Timing ไม่ผ่าน
**แก้ 1 — ลด clock:**
```python
# ใน generate_chip_config.py
"CLOCK_PERIOD": 1_000_000_000 / 50_000_000  # 50 MHz แทน 72 MHz
```

**แก้ 2 — เพิ่ม utilization + rebalance:**
```json
"PL_TARGET_DENSITY_PCT": 50
"PL_ROUTABILITY_DRIVEN": true
"SYNTH_STRATEGY": 3   // area-driven synthesis
```

**แก้ 3 — multi-Vt swap (ถ้า PDK มี multi-Vt):**
```python
# ใน config — เพิ่ม
"SYNTH_TIEHI_PORT": "vdd",
"SYNTH_TIELO_PORT": "vss",
```

#### 11.1.3 `DRC: Metal1 spacing < 0.065 µm`

**อาการ:** DRC spacing error
**แก้:**
```python
# ใน config
"DETAILED_ROUTING": {
    "SPACING": 0.07   # เพิ่ม spacing
}
"GRT_REPAIR_ANTENNAS": true
"GRT_ANTENNA_RATIO_THRESHOLD": 400
```

#### 11.1.4 `LVS: Power nets not connected`

**อาการ:** LVS mismatch — VDD/VSS ไม่ match
**แก้:** ตรวจ `pdn_cfg.tcl`:
```tcl
# ต้องมี
add_global_connection -net $::env(VDD_NET) -inst_pattern .* -pin_pattern vdd -power
add_global_connection -net $::env(GND_NET) -inst_pattern .* -pin_pattern vss -ground
add_global_connection -net IOVDD -inst_pattern .* -pin_pattern iovdd -power
add_global_connection -net IOVSS -inst_pattern .* -pin_pattern iovss -ground
```

ถ้ายังไม่ผ่าน → ตรวจ `PDN_MACRO_CONNECTIONS` ใน config.json

### 11.2 Optimization Recipes

#### 11.2.1 Area-Driven (`SYNTH_RECIPE=area`)

ใช้เวลา compile นานขึ้น แต่ cell count ลด ~10-15%

```bash
make CONFIG=configs/ci/ihp130.mk SYNTH=YOSYS SYNTH_RECIPE=area synth
```

#### 11.2.2 Speed-Driven (`SYNTH_RECIPE=speed`)

WNS ดีขึ้น แต่ cell count เพิ่ม ~20-30%

```bash
make CONFIG=configs/ci/ihp130.mk SYNTH=YOSYS SYNTH_RECIPE=speed synth
```

#### 11.2.3 Multi-corner STA

เพิ่ม corners ใน SDC:
```tcl
# corners: typical_1p20V_25C, fast_1p32V_m40C, slow_1p08V_125C
set sta_corner [lindex $::env(STA_CORNERS) 0]
```

---

## 12. Sign-off Checklist

### 12.1 Pre-submission Checklist

- [ ] **`make setup`** ทำเสร็จเรียบร้อย, lock digest verified
- [ ] **`make doctor`** ผ่าน (ทุก tool + version)
- [ ] **`make librelane-doctor`** ผ่าน (LibreLane 3.0.5 + IHP PDK revision ตรงกัน)
- [ ] **`make librelane-chip`** เสร็จสมบูรณ์ 22/22 steps
- [ ] **DRC:** `klayout_drc.json` แสดง 0 errors
- [ ] **LVS:** `netgen_lvs.rpt` แสดง "match"
- [ ] **STA:** WNS ≥ 0, TNS = 0 ทั้ง 3 corners (typ/fast/slow)
- [ ] **Antenna:** 0 violations
- [ ] **Power:** reasonable (target < 30 mW สำหรับ retroSoC)
- [ ] **Density:** 30-70% (ไม่แน่น/ไม่โล่งเกิน)
- [ ] **GDS verified visually** ใน KLayout
- [ ] **`make librelane-package`** archive สร้าง + SHA256SUMS verified
- [ ] **`make check-warnings`** ไม่มี warning ใหม่ (vs committed baseline)

### 12.2 Multi-corner Validation

retroSoC IHP130 ใช้ 3 corners (จาก PDK liberty):

| Corner | VDD | Temp | Use case |
|--------|-----|------|----------|
| `typ_1p20V_25C` | 1.20 V | 25°C | nominal |
| `fast_1p32V_m40C` | 1.32 V | -40°C | hold analysis |
| `slow_1p08V_125C` | 1.08 V | 125°C | setup analysis |

วิธีรัน multi-corner:
```bash
# ตั้ง STA_CORNERS ใน config (ถ้าต้องการ explicit)
export STA_CORNERS="corner1 corner2 corner3"
make CONFIG=configs/ci/ihp130.mk librelane-chip
```

### 12.3 Tape-out Submission (IHP SG13G2)

> ⚠️ **Disclaimer (จาก retroSoC README):**
> A successful open-source run is evidence for **implementation development**, not a foundry production-signoff claim. Release review still requires qualified foundry decks, package and bond planning, electrical/ESD review, and approved waivers.

**ถ้าจะส่งจริง:**
1. ติดต่อ IHP สำหรับ qualified PDK + process design kit
2. ESD review (pad protection)
3. Package selection + bond plan
4. Power integrity analysis
5. DFM review กับ foundry

### 12.4 Submission to IHP Open MPW / Tiny Tapeout

ถ้าจะส่งผ่าน **Tiny Tapeout** (open shuttle):
1. Compress GDS + collateral เป็น tarball
2. Submit ผ่าน Tiny Tapeout web (https://tinytapeout.com/)
3. ⚠️ retroSoC ใหญ่เกินไปสำหรับ TT (TT รับ ~100×100 µm เท่านั้น)
   → ใช้ IHP Open MPW แทน (https://www.ihp-microelectronics.com/)

---

## 13. Troubleshooting Cookbook

### 13.1 Environment

| Error | Cause | Fix |
|-------|-------|-----|
| `librelane: command not found` | ไม่ได้ activate env | `source .cache/retrosoc/development/activate.sh` (manual) หรือ `nix run .#dev` |
| `klayout: command not found` | KLayout not installed | `apt install klayout` (Ubuntu) |
| `yosys: version mismatch` | wrong yosys version | `make setup` ใหม่ |
| `ModuleNotFoundError: librelane` | ไม่ได้ source activate.sh | `source .cache/retrosoc/development/activate.sh` |

### 13.2 PDK

| Error | Cause | Fix |
|-------|-------|-----|
| `IHP PDK checkout is missing` | setup ไม่ครบ | `make setup-pdk` |
| `IHP PDK revision mismatch` | lock เปลี่ยน | `rm -rf physical/pdk/IHP-Open-PDK && make setup-pdk` |
| `sg13g2_stdcell.v not found` | PDK files corrupted | `cd physical/pdk/IHP-Open-PDK && git status` |

### 13.3 Synthesis (Yosys)

| Error | Cause | Fix |
|-------|-------|-----|
| `ERROR: Module 'sg13g2_IOPadIn' not found` | PDK filelist ไม่ครบ | ตรวจ `rtl/filelist/pdk_ihp130.fl` ต้องมี `sg13g2_io.v` ก่อน `sg13g2_stdcell.v` |
| `ERROR: Can't find RM_IHPSG13_1P_4096x16_c3_bm_bist` | SRAM macro ไม่ถูก blackbox | ใช้ `sram_blackboxes.vh` (มีอยู่แล้ว) |
| `WARNING: undriven net` | RTL incomplete | ใช้ `make check-rtl-lint` ก่อน |
| `Synthesis area > die area` | utilization สูงเกิน | ลด `SYNTH_STRATEGY` หรือเพิ่ม `DIE_AREA` |

### 13.4 Place-and-Route (OpenROAD)

| Error | Cause | Fix |
|-------|-------|-----|
| `Placement density overflow` | cells แน่น | ลด `PL_TARGET_DENSITY_PCT` ลง 25 → 20 |
| `Routing congestion > 0.5` | routing ตัน | เพิ่ม `PL_ROUTABILITY_DRIVEN` |
| `WNS < 0` | timing ไม่ผ่าน | ลด clock period หรือเพิ่ม `SYNTH_STRATEGY=area` |
| `OpenROAD crash (segfault)` | OOM | เพิ่ม RAM หรือ ลด `ROUTING_LAYER` |

### 13.5 Verification

| Error | Cause | Fix |
|-------|-------|-----|
| `DRC: Metal spacing` | spacing เล็กเกิน | เพิ่ม `DETAILED_ROUTING.SPACING` |
| `DRC: Via1 enclosure` | via ไม่ครอบ metal | เพิ่ม `VIA_IN_PIN` rules |
| `LVS: property mismatch` | ไฟล์ไม่ match | ตรวจ `pdn_cfg.tcl` — `add_global_connection` |
| `STA: clock skew > 1 ns` | CTS ไม่สมดุล | เพิ่ม `CTS_TARGET_SKEW` |

### 13.6 Network / Download

| Error | Cause | Fix |
|-------|-------|-----|
| `Failed to download IHP-Open-PDK` | network issue | retry หรือใช้ VPN |
| `Checksum mismatch` | corrupted download | ลบ `.cache/retrosoc/downloads/...` แล้ว retry |
| `git clone: authentication failed` | HTTPS issue | `git config --global credential.helper store` |

---

## 14. Resources & Next Steps

### 14.1 Official Documentation

| Resource | URL |
|----------|-----|
| retroSoC GitHub | https://github.com/retroSoC/retroSoC |
| retroSoC Engineering | https://github.com/retroSoC/retroSoC/blob/main/docs/engineering.md |
| retroSoC Development | https://github.com/retroSoC/retroSoC/blob/main/docs/development-environment.md |
| LibreLane | https://github.com/librelane/librelane |
| LibreLane 3.0 docs | https://librelane.readthedocs.io/en/latest/ |
| IHP-Open-PDK | https://github.com/IHP-GmbH/IHP-Open-PDK |
| IHP SG13G2 LibreLane | https://github.com/IHP-GmbH/IHP-Open-PDK/tree/main/ihp-sg13g2/libs.tech/librelane |
| Hazard3 CPU | https://github.com/Wren6991/Hazard3 |

### 14.2 Workshop Materials (ที่อาจารย์เตรียมไว้)

| File | Description |
|------|-------------|
| `INDEX.md` | Index ของ workshop handbook (14 sections, ~310 KB) |
| `workshop-section-04-environment-setup.md` | Setup LibreLane + IHP SG13G2 |
| `workshop-section-06-soc-design.md` | ตัวอย่าง SoC design (RV32I) |
| `workshop-section-08-librelane-config.md` | config.yaml explained |
| `workshop-section-09-run-flow.md` | วิธีรัน LibreLane flow |
| `workshop-section-12-signoff-tapeout.md` | Sign-off & tape-out |

### 14.3 Communities

| Community | Link |
|-----------|------|
| FOSSi Foundation | https://fossi-foundation.org/ |
| FOSSi Dialogue | https://libera.chat/ (channel: #fossi) |
| Tiny Tapeout Discord | https://tinytapeout.com/discord |
| IHP Open PDK Discussions | https://github.com/IHP-GmbH/IHP-Open-PDK/discussions |
| retroSoC Issues | https://github.com/retroSoC/retroSoC/issues |

### 14.4 Learning Path (หลังจากจบคู่มือนี้)

1. **เปลี่ยน pin map** — เพิ่ม GPIO, เปลี่ยน SDIO → USB mapping
2. **ทดลอง multi-corner STA** — เพิ่ม `corner1`, `corner2`, `corner3` ใน config
3. **เพิ่ม CoreMark benchmark** — `make CONFIG=configs/benchmark/ihp130-hazard3-coremark.mk coremark-report`
4. **เปลี่ยน SRAM instances** — แก้ `SRAM_INSTANCES` ใน `generate_chip_config.py`
5. **เปิด PLL** — `HAVE_PLL=YES` + `xtal` pads + `pll_rcu` (ต้องมี qualified timing profile)
6. **Post-layout simulation** — ใช้ `final/sdf/retrosoc_asic.sdf` + testbench
7. **Formal verification** — `make formal` (SymbiYosys + Bitwuzla)
8. **ส่ง MPW** — ติดต่อ IHP Open MPW หรือ efabless

### 14.5 Cheat Sheet — Commands ที่ใช้บ่อย

```bash
# === Setup ===
make setup                                       # ดาวน์โหลด PDK + IP (first time)
make CONFIG=configs/ci/ihp130.mk doctor          # ตรวจ env
make CONFIG=configs/ci/ihp130.mk config          # ดู config ที่เลือก

# === LibreLane Chip Flow ⭐ ===
make CONFIG=configs/ci/ihp130.mk librelane-doctor      # validate
make CONFIG=configs/ci/ihp130.mk librelane-input       # generate inputs
make CONFIG=configs/ci/ihp130.mk librelane-chip        # 🎯 RTL→GDS
make CONFIG=configs/ci/ihp130.mk librelane-openroad    # เปิด OpenROAD
make CONFIG=configs/ci/ihp130.mk librelane-klayout     # เปิด KLayout
make CONFIG=configs/ci/ihp130.mk librelane-package     # สร้าง tarball

# === Auxiliary flows ===
make CONFIG=configs/ci/ihp130.mk SIMU=IVERILOG sim    # ทดสอบ RTL
make CONFIG=configs/ci/ihp130.mk SYNTH=YOSYS synth     # Yosys synthesis
make CONFIG=configs/ci/ihp130.mk STA=OPENSTA sta       # OpenSTA core STA
make CONFIG=configs/ci/ihp130.mk rtl-lint              # Verilator lint
make CONFIG=configs/ci/ihp130.mk formal                # SymbiYosys proofs

# === Clean ===
make CONFIG=configs/ci/ihp130.mk clean                 # ลบ build variant ปัจจุบัน
make clean-all                                         # ลบ build ทั้งหมด
make purge-cache                                       # ลบ download cache

# === Output paths ===
ls build/<variant>/physical/librelane/chip/final/gds/   # 🎯 FINAL GDS
ls build/<variant>/physical/librelane/chip/final/reports/   # reports ทั้งหมด
ls build/<variant>/meta/                               # manifest + doctor + chip results
```

### 14.6 What's Next?

เมื่อทำเสร็จเรียบร้อย คุณจะมี:

✅ **ชุด LibreLane IHP130 single-level pad-ring flow ที่ทำงานได้จริง**
✅ **Final GDS** (`retrosoc_asic.gds` — 8 × 8 mm)
✅ **Full sign-off reports** (DRC, LVS, STA, Antenna, Power)
✅ **Checksummed deliverable archive** (`retrosoc-ihp130-chip.tar.gz`)
✅ **ความเข้าใจ retroSoC architecture** (Hazard3 + peripherals + bus fabric)
✅ **ความเข้าใจ LibreLane flow** (22 steps จาก RTL ถึง GDS)

**ขั้นตอนถัดไปที่แนะนำ:**
1. ลองเปลี่ยน `pin_map.json` → เพิ่ม/ลด pads
2. ลอง optimize ด้วย `SYNTH_RECIPE=area` vs `speed`
3. ทดลอง multi-corner STA
4. เปิด `formal` flow (SymbiYosys proofs)
5. ศึกษา `tests/test_librelane_flow.py` — เป็น regression test ที่ comprehensive

---

## Appendix A: Quick Reference — ไฟล์สำคัญ

| ไฟล์ | บทบาท |
|------|-------|
| `Makefile` | Top-level Makefile (root) |
| `configs/ci/ihp130.mk` | Configuration profile |
| `rtl/mini/top/retrosoc_asic.sv` | Top-level SoC (synth target) |
| `rtl/filelist/pdk_ihp130.fl` | IHP130 filelist (sg13g2_io, stdcell, SRAM) |
| `rtl/mini/pin_map/pin_map.json` | Pin map (canonical, 109 signal pads) |
| `rtl/mini/integration/clock_reset_domains.json` | Clock/reset domains |
| `physical/librelane/Makefile` | LibreLane Makefile (includable) |
| `physical/librelane/scripts/generate_chip_config.py` | Generate config.json |
| `physical/librelane/scripts/generate_sdc.py` | Generate SDC |
| `physical/librelane/scripts/doctor.py` | Validate env (LibreLane + PDK) |
| `physical/librelane/scripts/package.py` | Create deliverable archive |
| `physical/librelane/pdn_cfg.tcl` | PDN configuration |
| `physical/librelane/sram_blackboxes.vh` | SRAM blackbox declarations |
| `physical/librelane/bondpad/bondpad_70x70.lef` | Bondpad LEF |
| `dependencies/dependencies.lock.json` | Lock-pinned versions |
| `docs/engineering.md` | Engineering workflow |
| `docs/development-environment.md` | Env setup guide |
| `tests/test_librelane_flow.py` | Regression tests |

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **BIST** | Built-In Self-Test (SRAM macro feature) |
| **Bondpad** | 70×70 µm metal pad for wire-bonding to package |
| **CDC** | Clock Domain Crossing |
| **DRC** | Design Rule Check |
| **FOSSi** | Free and Open Source Silicon Foundation |
| **GDSII** | Graphic Data Stream (chip layout format) |
| **Hazard3** | RISC-V RV32IM CPU by Luke Wren |
| **LVS** | Layout vs Schematic |
| **MPW** | Multi-Project Wafer |
| **PDK** | Process Design Kit |
| **PDN** | Power Distribution Network |
| **Rib/Lib** | retroSoC internal bus (Rib) + Wishbone-style (Lib) |
| **SDC** | Synopsys Design Constraints (timing) |
| **STA** | Static Timing Analysis |
| **ULPI** | UTMI+ Low Pin Interface (USB 2.0 PHY) |
| **WNS/TNS** | Worst/Total Negative Slack |

---

**Happy chip-taping! 🎉**

> ถ้ามีข้อสงสัยเพิ่มเติม:
> - 📖 เปิด `docs/engineering.md` ใน retroSoC repo
> - 💬 ถามใน FOSSi Discord / Tiny Tapeout Discord
> - 🐛 เปิด issue ที่ https://github.com/retroSoC/retroSoC/issues
> - 📚 กลับไปอ่าน `INDEX.md` ของ workshop handbook สำหรับ deep-dive sections
