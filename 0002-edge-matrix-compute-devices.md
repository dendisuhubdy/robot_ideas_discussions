# Edge devices for efficient matrix computation (beyond Jetson)

- **Date:** 2026-06-18
- **Topic:** Robotics / Edge AI / Hardware Accelerators

## Question

Other than Jetson Nano and Jetson Thor, what other types of devices can do efficient matrix computation on the edge?

## Answer

### Dedicated NPUs / inference ASICs (best TOPS/watt)

These are purpose-built matrix-multiply engines — usually the efficiency sweet spot for edge vision/inference.

- **Hailo** — Hailo-8 (~26 TOPS, ~2.5W), Hailo-10 (gen-AI focused), Hailo-15 (vision SoC). M.2/PCIe form factors, excellent TOPS/W. Probably the most direct Jetson alternative for pure inference.
- **Google Coral / Edge TPU** — ~4 TOPS INT8, ~2W. USB Accelerator, M.2, Dev Board. Mature TFLite tooling, but INT8-only and aging.
- **Axelera Metis** — digital in-memory compute AIPU, ~214 TOPS on M.2/PCIe. Strong for multi-stream vision.
- **SiMa.ai MLSoC** — purpose-built embedded edge ML, good software stack for computer vision pipelines.
- **MemryX MX3** — accelerator chiplets, easy multi-chip scaling.
- **Kneron KL530/730** — low-power NPUs aimed at smart cameras/IoT.
- **Ambarella CV5/CV72** — vision SoCs (CVflow), big in automotive/security cameras.

### SoCs with integrated NPU (good for "one chip does everything")

- **Qualcomm** — Snapdragon (Hexagon NPU), QCS6490/RB5 robotics, Dragonwing, Cloud AI 100 edge cards. Very strong perf/W, especially for on-device gen-AI.
- **Rockchip RK3588 / RK3576** — ~6 TOPS NPU, cheap, ubiquitous in SBCs (Radxa, Orange Pi). Great price/perf for hobbyist-to-product.
- **NXP i.MX 8M Plus / i.MX 9** — integrated NPU (eIQ Neutron), industrial temp ranges and long lifecycle support.
- **TI TDA4VM / Jacinto** — C7x DSP + MMA (matrix multiply accelerator), automotive ADAS focus.
- **Apple Neural Engine (M-/A-series)** — extremely efficient, but closed ecosystem.
- **Intel Core Ultra** — built-in NPU (OpenVINO), plus the older Movidius Myriad X VPU / Neural Compute Stick.
- **AMD Ryzen AI** — XDNA NPU in laptop/embedded SoCs.

### FPGA / reconfigurable

Worth it when you need custom dataflow or deterministic latency.

- **AMD/Xilinx** — Versal AI Edge (AI Engines), Kria K26 SOM, Zynq UltraScale+ with DPU.
- **Lattice** — sensAI for ultra-low-power always-on.

### Ultra-low-power / TinyML / MCU-class (mW range)

For battery and always-on sensing rather than heavy matmul.

- **ARM Ethos-U55/U65/U85** — NPU IP paired with Cortex-M; shows up across many vendor MCUs.
- **STMicro STM32N6** — Neural-ART accelerator on an MCU.
- **Analog Devices MAX78000** — CNN accelerator MCU, microjoule-per-inference.
- **GreenWaves GAP9** — RISC-V parallel cluster for audio/sensor ML.
- **Syntiant NDP** — always-on audio/keyword.

### Novel architectures (compute-in-memory / neuromorphic)

- **Mythic M1076** — analog in-memory compute, very high efficiency for fixed weights.
- **BrainChip Akida** — neuromorphic, event-driven, sparse workloads.

---

## Cheat sheet: TOPS/W and form factor

Numbers are vendor-stated, INT8 unless noted, and approximate — treat as ballpark for fast triage, not benchmarks. "Perf/W" is a rough efficiency tier, not a precise ratio.

| Device | Class | Peak TOPS | Power | Perf/W tier | Form factor | Best for |
|---|---|---|---|---|---|---|
| **Hailo-8** | NPU ASIC | ~26 | ~2.5 W | ★★★★★ | M.2 / PCIe | Pure inference, best TOPS/W |
| **Hailo-10** | NPU ASIC | ~40 | ~3.5 W | ★★★★★ | M.2 | On-device gen-AI / LLM |
| **Hailo-15** | Vision SoC | ~20 | low-single-W | ★★★★☆ | SoC | Smart cameras |
| **Google Coral / Edge TPU** | NPU ASIC | ~4 | ~2 W | ★★★★☆ | USB / M.2 / board | Mature TFLite, INT8-only |
| **Axelera Metis** | In-mem AIPU | ~214 | ~10–15 W | ★★★★☆ | M.2 / PCIe | Multi-stream vision |
| **SiMa.ai MLSoC** | Edge ML SoC | ~50 | ~5–10 W | ★★★★☆ | SoM / board | CV pipelines, good SW |
| **MemryX MX3** | Accel chiplet | ~5 /chip | ~1–2 W | ★★★★☆ | M.2 (scalable) | Multi-chip scaling |
| **Kneron KL730** | NPU | ~8 | ~1–2 W | ★★★★☆ | chip / module | Smart cameras / IoT |
| **Ambarella CV72** | Vision SoC | ~20–30 | ~5 W | ★★★★☆ | SoC | Automotive / security cam |
| **Qualcomm QCS6490 / RB5** | SoC + Hexagon | ~12–15 | ~5–10 W | ★★★★☆ | SoM / board | Robotics, on-device gen-AI |
| **Qualcomm Cloud AI 100** | Edge card | ~70–400 | ~15–75 W | ★★★★☆ | M.2 / PCIe | High-throughput edge |
| **Rockchip RK3588** | SoC + NPU | ~6 | ~5–8 W | ★★★☆☆ | SBC (Radxa/Opi) | Cheap price/perf, hobby→prod |
| **NXP i.MX 8M Plus** | SoC + NPU | ~2.3 | ~3–5 W | ★★★☆☆ | SoM / SBC | Industrial, long lifecycle |
| **TI TDA4VM** | SoC + MMA | ~8 | ~5–20 W | ★★★☆☆ | SoM | Automotive ADAS |
| **Apple Neural Engine** | SoC NPU | ~18–38 | ~5–15 W | ★★★★★ | SoC (closed) | Apple-only, very efficient |
| **Intel Core Ultra** | CPU + NPU | ~11–48 | ~15–28 W | ★★★☆☆ | laptop / NUC | OpenVINO, x86 ecosystem |
| **AMD Ryzen AI** | CPU + XDNA NPU | ~16–50 | ~15–28 W | ★★★☆☆ | laptop / embedded | x86 + NPU |
| **AMD/Xilinx Kria K26** | FPGA SOM | ~1.4 (DPU) | ~5–15 W | ★★★☆☆ | SOM | Custom dataflow, det. latency |
| **Versal AI Edge** | FPGA + AIE | ~10–75 | ~10–30 W | ★★★★☆ | card / module | Custom pipelines, low latency |
| **ARM Ethos-U85** | MCU NPU IP | ~0.1–4 | mW–1 W | ★★★★★ | IP in MCU | TinyML, always-on |
| **STM32N6** | MCU + NPU | ~0.6 | <1 W | ★★★★★ | MCU | Battery sensing |
| **MAX78000** | MCU + CNN | tiny | µJ/inference | ★★★★★ | MCU | Always-on, microjoule |
| **GreenWaves GAP9** | RISC-V cluster | tiny | mW | ★★★★★ | MCU | Audio / sensor ML |
| **Syntiant NDP** | Always-on NPU | tiny | µW–mW | ★★★★★ | chip | Keyword / audio wake |
| **Mythic M1076** | Analog in-mem | ~25 | ~3–4 W | ★★★★★ | M.2 / chip | Fixed-weight CNNs |
| **BrainChip Akida** | Neuromorphic | event-driven | mW | ★★★★★ | IP / chip | Sparse, event-based |

### How to read it quickly

- **Max efficiency, pure inference:** Hailo-8/10, Coral, Mythic.
- **One chip does everything (CPU+NPU+I/O) for robotics:** Qualcomm QCS6490/RB5, Rockchip RK3588, NXP i.MX.
- **Raw throughput at the edge:** Axelera Metis, Qualcomm Cloud AI 100, Versal AI Edge.
- **Battery / always-on (mW):** Ethos-U, STM32N6, MAX78000, Syntiant, GAP9.
- **Custom dataflow / hard real-time latency:** Xilinx Kria / Versal FPGAs.
