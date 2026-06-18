# Quantizable deep learning models for industrial robots, UGVs, and UAVs

- **Date:** 2026-06-18
- **Topic:** Robotics / Edge AI / Model Selection

## Question

What deep learning models might be useful for industrial robots, UGVs, and UAVs that can be quantized?

## Answer

The constraint that matters: it has to be **quantization-friendly** (clean INT8/FP8, ideally exportable to ONNX → TensorRT or compilable to a Vitis AI `.xmodel`) and run in real time inside a drone/robot power+thermal envelope. That rules out most giant transformers at full size, but a lot of strong models fit. Grouped by capability.

### 1. Object detection (the workhorse)

The most common edge perception task — find/classify things in a frame.

- **YOLO family (v8/v10/v11, YOLO-NAS)** — the default. Great speed/accuracy, first-class INT8 export, well-supported on both TensorRT and Vitis AI. YOLO-NAS was literally NAS-designed to be quantization-friendly (minimal INT8 accuracy drop).
- **RT-DETR / RT-DETRv2** — real-time transformer detector; NMS-free, quantizes reasonably with QAT.
- **SSD-MobileNet / EfficientDet-Lite** — older but extremely cheap; good for MCU-class or always-on.
- **Use cases:** UAV target/obstacle spotting, UGV pedestrian/vehicle detection, industrial part/defect detection, pick-and-place item localization.

### 2. Semantic / instance / panoptic segmentation

Per-pixel understanding — drivable terrain, defects, free space.

- **YOLOv8-seg / YOLO11-seg** — instance masks at detection speed; easy to quantize.
- **DeepLabV3+ (MobileNet backbone)** — classic semantic seg, INT8-friendly, common on Vitis AI model zoo.
- **BiSeNet / DDRNet / PP-LiteSeg / Fast-SCNN** — real-time semantic seg backbones designed for embedded.
- **SAM / MobileSAM / EfficientViT-SAM** — promptable segmentation; the lite variants are edge-viable.
- **Use cases:** UGV terrain/drivable-area segmentation, UAV crop/field analysis, industrial surface-defect maps.

### 3. Monocular depth & 3D

- **MiDaS / Depth Anything (small/ViT-S) / Metric3D** — monocular depth; small variants quantize and run on Jetson.
- **FastDepth / MobileStereoNet** — lightweight depth for obstacle avoidance.
- **Use cases:** UAV collision avoidance, UGV navigation without LiDAR, bin-picking depth estimation.

### 4. Visual SLAM / VIO front ends (learned components)

- **SuperPoint + SuperGlue / LightGlue** — learned feature detection + matching; quantizable and far more robust than classical ORB in low texture. Drops into a SLAM front end.
- **Learned VO (e.g. TartanVO-style)** — end-to-end visual odometry.
- **Use cases:** GPS-denied UAV/UGV localization (ties back to `0001-how-slam-works.md`).

### 5. Multi-object tracking

- **ByteTrack / OC-SORT / BoT-SORT** — mostly classical association on top of a quantized detector, so the heavy part (the detector) is already INT8. Cheap to add.
- **Use cases:** UAV following a target across frames, UGV tracking workers/vehicles, conveyor tracking.

### 6. Pose / keypoint estimation

- **YOLOv8-pose / RTMPose / MoveNet / BlazePose** — human or object keypoints; light and quantizable.
- **Use cases:** human-robot interaction safety, gesture cues, industrial assembly verification.

### 7. Classification / anomaly / quality backbones

- **MobileNetV3 / EfficientNet-Lite / GhostNet / RepVGG / ShuffleNet** — efficient backbones; RepVGG and MobileNet quantize especially cleanly (RepVGG's reparam'd plain-conv structure is INT8-friendly).
- **PaDiM / PatchCore (anomaly detection)** — for "is this part defective?" with few examples.
- **Use cases:** industrial visual QC, defect/anomaly detection, scene classification.

### 8. Point-cloud / LiDAR (for UGVs and heavier UAVs)

- **PointPillars / CenterPoint / SECOND** — 3D object detection from LiDAR; PointPillars in particular is a staple in the Vitis AI / TensorRT automotive world and quantizes well.
- **Use cases:** UGV 3D obstacle detection, autonomous ground navigation.

### 9. The "newer / heavier" tier (quantizable but watch the budget)

- **Small VLMs / VLA models** — e.g. distilled/quantized OpenVLA, π0, or MobileVLM-class. INT4/INT8-quantized small LLMs (Llama-3.2-1B/3B, Phi-3-mini, Qwen2.5) can run on **Jetson Thor** for natural-language tasking or high-level reasoning. Heavy but feasible on the top tier.
- **Transformer detectors / ViT backbones** — quantize with QAT; doable but more sensitive to INT8 than CNNs.
- **Use cases:** UAV/UGV natural-language mission tasking, semantic scene reasoning, "go inspect the red valve."

---

## Quantization friendliness — quick guide

| Quantizes very cleanly (INT8 PTQ) | Needs care (prefer QAT) |
|---|---|
| YOLO family, MobileNet, EfficientNet-Lite, RepVGG, GhostNet, DeepLabV3+, PointPillars, SuperPoint | Transformers (RT-DETR, ViT, SAM, VLMs), some depth nets, anything with LayerNorm/attention-heavy blocks |

**Rules of thumb:**
- **CNNs quantize easier than transformers.** Convs + BN fold to INT8 with little loss; attention/LayerNorm are more sensitive.
- **Prefer PTQ (post-training) first**; fall back to **QAT (quantization-aware training)** only if accuracy drops too much.
- **Match precision to the chip:** INT8 everywhere; FP8/FP4 only on Thor's Blackwell GPU; Kria DPU is INT8-only.
- **Watch unsupported ops** — exotic activations/custom layers may not have an INT8 kernel on TensorRT or a DPU mapping in Vitis AI. Check the op coverage before committing.

## Suggested starting stack per platform

- **UAV (small, power-tight):** YOLOv11-n/s detector + Depth Anything-S + ByteTrack, on Jetson Orin Nano (INT8).
- **UGV (more compute, navigation):** YOLO-seg + PointPillars (if LiDAR) + SuperPoint/LightGlue for VIO, on Jetson Orin or Kria.
- **Industrial arm/fixed:** YOLO detection + PatchCore anomaly + pose/keypoint for verification, on whatever NPU/SoC fits the cell.
