# How quantized models get delivered and run continuously on edge devices

- **Date:** 2026-06-18
- **Topic:** Robotics / Edge AI / Model Ops / UAV

## Question

How do quantized models get delivered and run continuously on these devices (Jetson-based Talon N1 / Apex T1, Kria-based Raptor K1), especially given UAV intermittent connectivity?

## Answer

Think of it as two planes: a **delivery pipeline** (cloud → device) and a **runtime** (load → serve → hot-swap). The wrinkle for UAVs is intermittent connectivity, so the design has to assume the model is fetched on the ground and runs fully offline in the air.

### 1. Per-device model format (quantization is device-specific)

You can't ship one artifact to all three — the quantized binary is tied to the accelerator:

- **Talon N1 / Apex T1 (Jetson) → TensorRT engines.** Quantize to INT8 (Orin Nano) or FP8/FP4 (Thor's Blackwell GPU). Two options: ship a portable ONNX and build the TensorRT engine on-device on first install (handles driver/TRT version drift), or pre-build engines AOT for a pinned JetPack and ship those (faster cold-start, but version-locked). For drones I'd do on-device build once, then cache the serialized engine.
- **Raptor K1 (Kria K26) → quantize with Vitis AI (INT8) and compile to a `.xmodel` for the DPU.** This is AOT only — the device just loads the compiled graph.

So your registry stores artifacts keyed by `(model, version, target)` where `target` = `orin-nano-jp6`, `thor-jp7`, `kria-k26-dpu`, etc.

### 2. Delivery pipeline (cloud → device)

A standard OTA-for-models flow:

1. **Model registry + object store / CDN** — signed, versioned artifacts (think a private S3/R2 bucket fronted by your Rust backend, which you already have).
2. **Device pull, not push** — each drone's agent polls `GET /models/manifest?device=<id>` and gets the manifest of models assigned to its hardware target. Pull is better than push for fleets that are intermittently online.
3. **Signed + hashed artifacts** — sign each blob (e.g. Ed25519) and verify the signature + SHA-256 before it's ever loaded. This is non-negotiable for anything defense-adjacent — it's your supply-chain integrity gate.
4. **Resumable / delta downloads** — over LTE on the ground, use HTTP range requests so a dropped link resumes; ship weight deltas when only a fine-tune changed.
5. **A/B model slots** — download into an inactive slot, verify, then atomically flip. Keep the previous slot as last-known-good for instant rollback.

### 3. Running it continuously (the runtime)

A long-lived inference daemon (systemd service on Jetson, app on Kria) that:

- **Loads the active engine into a persistent context once and keeps it warm** — you pay the load/JIT cost a single time, then stream frames through it. Re-loading per frame is the classic mistake that kills throughput.
- **Hot-reloads without dropping the flight pipeline:** load the new engine in a second context, warm it, swap the pointer, free the old one (double-buffering). No process restart, no gap in perception.
- **Runs a watchdog:** if the new model crashes or its health-check (latency/accuracy canary) regresses, auto-revert to the LKG slot.
- **Bounds resources** — pin to specific GPU/DPU, cap power mode (the Jetson `nvpmodel` / Thor 40–130 W envelope), since on a drone thermal + battery are hard limits.

### 4. The UAV reality layer

- **Fetch on the ground, run in the air.** Sync models over LTE/Wi-Fi at base; in flight the device is autonomous with whatever's cached. Manifest reconciliation happens opportunistically when a link returns.
- **Encrypt at rest + secure/measured boot** so a recovered airframe doesn't leak your models or weights.
- **Telemetry back-channel:** report active model version, inference latency, and accuracy canaries per device so you can manage the fleet.

### Where this becomes a product

This delivery+runtime layer is exactly the kind of thing you could ship with the bundles as a managed "fleet model-ops" service — the hardware is the wedge, recurring model-OTA is the attach. Worth considering as a line item.
