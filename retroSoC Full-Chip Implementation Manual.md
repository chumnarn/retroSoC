คู่มือ **retroSoC Full-Chip Implementation Manual** 

<deliver-assets>
<media type="file" src="C:\Users\000148\.minimax\workspace\retrosoc-librelane-ihp130-manual.md" caption="retroSoC Full-Chip Implementation Manual — LibreLane 3.0.5 + IHP SG13G2 (52 KB, 1,388 lines, 14 sections)" />
</deliver-assets>


### โครงสร้างคู่มือ (14 sections, step-by-step)

| # | Section | เนื้อหาสำคัญ |
|---|---------|--------------|
| 1 | **Overview & Goal** | retroSoC, IHP SG13G2, LibreLane 3.0.5 — what/why/runtime |
| 2 | **Prerequisites** | HW 16GB+/30GB disk, Linux x86_64, knowledge self-check |
| 3 | **Architecture Deep Dive** | Hazard3 + peripherals + pad-ring 189 instances + 5 clock domains |
| 4 | **Environment Setup** | **3 วิธี:** Docker (recommended) / Nix / Manual + เปรียบเทียบ |
| 5 | **Project Initialization** | `make setup`, `make doctor`, `make config` |
| 6 | **Configuration Generation** | `generate_chip_config.py` + `generate_sdc.py` — auto-generated |
| 7 | **Run LibreLane Flow** ⭐ | `librelane-doctor` → `librelane-chip` (30 นาที - 2 ชม.) → `package` |
| 8 | **Visualization** | OpenROAD + KLayout + screenshot |
| 9 | **Output Structure & Artifacts** | `build/<variant>/.../final/` — GDS, DEF, LEF, SDF, SPEF |
| 10 | **Verification — DRC, LVS, STA, Antenna** | อ่าน reports + ตีความ WNS/TNS |
| 11 | **Debug & Optimization Playbook** | recipes: area / speed / multi-corner |
| 12 | **Sign-off Checklist** | 13 items + multi-corner + MPW submission |
| 13 | **Troubleshooting Cookbook** | per-error fix table (env/PDK/synth/PnR/verify) |
| 14 | **Resources & Next Steps** | official docs, communities, learning path, cheat sheet |

### จุดเด่น

✅ **Verified content** — ดึงมาจาก source code จริงของ retroSoC (configs/ci/ihp130.mk, physical/librelane/Makefile, generate_chip_config.py, generate_sdc.py, pdn_cfg.tcl, sram_blackboxes.vh, doctor.py, package.py, dependencies.lock.json, docs/engineering.md)

✅ **Step-by-step copy-paste commands** — ทุกคำสั่งพร้อมรัน, มี expected output

✅ **3 environment options** พร้อมเปรียบเทียบ (Docker/Nix/Manual) — เลือกตาม use case

✅ **อธิบาย LibreLane config ทุก parameter** — DIE_AREA, CORE_AREA, CLOCK_PERIOD, MACROS, PDN, VDD_NETS, GND_NETS

✅ **Pad-ring 4-side layout** (189 instances) — เข้าใจว่าทำไมเป็น single-level chip flow

✅ **5 clock domains + async groups** ใน SDC — ตามที่ retroSoC ใช้จริง

✅ **Troubleshooting table** แยกหมวด env/PDK/synth/PnR/verify/network — แก้ปัญหาเร็ว

✅ **Sign-off checklist** 13 items + multi-corner + MPW submission — พร้อม tape-out

### เริ่มต้นใช้งาน (5 คำสั่ง)

```bash
git clone https://github.com/chumnarn/retroSoC.git
cd retroSoC
make setup
make CONFIG=configs/ci/ihp130.mk librelane-doctor
make CONFIG=configs/ci/ihp130.mk librelane-chip
```

🎯 **Final output:** `build/<variant>/physical/librelane/chip/final/gds/retrosoc_asic.gds` (8 × 8 mm)
