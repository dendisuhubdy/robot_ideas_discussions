# What's the difference between TensorRT and Vitis AI?

- **Date:** 2026-06-18
- **Topic:** Robotics / Edge AI / Inference Toolchains

## Question

What's the difference between TensorRT and Vitis AI?

## Answer

Both turn a trained neural network into an optimized binary for inference — but they target fundamentally different silicon, which drives every other difference.

### One-line summary

- **TensorRT** = NVIDIA's compiler/runtime for **GPUs** (datacenter + Jetson). Optimizes for a fixed, programmable GPU architecture.
- **Vitis AI** = AMD/Xilinx's quantizer/compiler for the **DPU**, a soft IP core that runs on **FPGAs / adaptive SoCs** (Kria, Zynq, Versal). Optimizes for reconfigurable dataflow hardware.

### Side-by-side

| | **TensorRT** | **Vitis AI** |
|---|---|---|
| Vendor | NVIDIA | AMD (formerly Xilinx) |
| Target hardware | NVIDIA GPUs — Jetson (Orin/Thor), datacenter | DPU core on Kria / Zynq UltraScale+ / Versal FPGAs |
| Compute model | Fixed GPU SIMT architecture | Reconfigurable DPU; FPGA fabric can be re-architected |
| Output artifact | Serialized **TensorRT engine** (`.engine`/`.plan`) | Compiled **`.xmodel`** graph for a specific DPU config |
| Build location | Often **on-device / JIT** (engine tied to GPU + TRT version) | **AOT** — compiled on host, device just loads it |
| Quantization | INT8 (PTQ/QAT), FP16, and FP8/FP4 on Blackwell (Thor) | Primarily **INT8** (PTQ + QAT via the Vitis AI quantizer) |
| Key optimizations | Layer/tensor fusion, kernel auto-tuning, precision calibration, dynamic tensor memory | Quantize → compile to DPU instructions; fixed-function dataflow |
| Frameworks in | ONNX, PyTorch, TensorFlow → ONNX | TensorFlow, PyTorch, ONNX (via Vitis AI quantizer) |
| Runtime | TensorRT runtime / Triton; CUDA stack | Vitis AI Runtime (VART), XRT |
| Flexibility | Tuned for one GPU SKU; recompile per arch | Can also **re-architect the FPGA fabric** itself for custom ops/dataflow |

### The deeper distinction

- **TensorRT optimizes software for fixed hardware.** The GPU is what it is; TensorRT picks the best kernels, fuses layers, and calibrates precision to squeeze that specific GPU. Engines are **not portable** across GPU architectures or even TRT versions — hence the on-device build pattern.
- **Vitis AI optimizes a model for hardware you can change.** The DPU is a configurable IP block; you can pick a DPU variant (or in principle customize the fabric) to match your model's dataflow, then compile the network to that. This is the FPGA advantage: **deterministic latency** and **custom dataflow**, at the cost of a more involved toolchain and lower peak TOPS than a comparable GPU.

### When you'd reach for each

- **TensorRT** — you're on Jetson/NVIDIA, want the richest operator coverage, easy FP16/INT8/FP8, fast iteration, and a huge ecosystem. Best default for most edge vision/gen-AI on NVIDIA.
- **Vitis AI** — you're on Kria/Zynq/Versal and need **hard real-time, deterministic latency**, ultra-low/bounded power, custom I/O integration, or sensor-to-inference pipelines fused into the FPGA fabric. Common in automotive, aerospace, and industrial where latency determinism matters more than peak throughput.

### Tie-in to the device notes

This is exactly why the delivery pipeline keys artifacts by hardware **target**: a Jetson bundle ships a **TensorRT engine** (often built on-device), while a Kria bundle ships an AOT-compiled **`.xmodel`** from Vitis AI. Same source model, two non-interchangeable toolchains. (See `0006-quantized-model-delivery-and-runtime.md`.)
