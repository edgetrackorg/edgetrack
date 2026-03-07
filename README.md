# EdgeTrack

EdgeTrack is designed as a deterministic capture backend for professional 3D authoring and interaction systems, while also being highly suitable for robotics, teleoperation, VR environments, and other precision spatial workflows.

**EdgeTrack** is an on-edge capture and preprocessing stack for synchronized **RAW10 mono multi-camera** pipelines, targeting **Raspberry Pi 5 (primary)** or **Radxa Dragon Q6A (secondary)**.
It provides **deterministic camera I/O**, **calibration-aware undistortion/normalization**, and **on-edge stereo reconstruction** to output **metric-scale 3D keypoints derived from calibrated stereo geometry**, with time-consistent sampling (error bounds depend on calibration and mechanics). Optionally, EdgeTrack can publish **ROI-reduced sparse 3D point clouds** to a host over **Gigabit LAN**.

EdgeTrack is designed to pair with **[TDMStrobe](https://github.com/edgetrackorg/tdmstrobe)** and **[CoreFusion](https://github.com/edgetrackorg/corefusion)** to deliver stable synchronization, low-latency capture, and **deterministic NIR illumination**. TDMStrobe provides hardware-timed strobe control aligned to camera triggers and can coordinate illumination using an **A/B/C/D phase sequence** across multiple rigs to reduce cross-talk and keep exposure consistent. CoreFusion ingests the time-aligned outputs from multiple EdgeTrack nodes over **Gigabit Ethernet** and fuses them into a coherent representation (e.g., multi-view keypoints, tracks, or sparse geometry) with predictable timing and well-defined latency.

> **Status:** early prototype. APIs, wiring, and schemas may change.

---

## Features

* **Metric 3D on the edge:** On-device stereo reconstruction outputs **metric 3D keypoints** (and optional sparse ROI point clouds) instead of raw video streams or purely 2D detections.
* **RAW-first capture:** **RAW10** preserves linear sensor data at the edge, avoiding ISP-induced artifacts that can reduce calibration accuracy and stereo stability.
* **Lean capture stack:** Minimizes heavy camera abstractions where deterministic control matters, using a **direct V4L2-style RAW** pipeline instead of relying solely on higher-level layers.
* **Deterministic timing:** Synchronized **global-shutter sensors** with **TDM strobe phases** enable repeatable capture timing and consistent geometry.
* **850/940 nm IR-ready:** Supports **850 nm or 940 nm** illumination (sensor- and filter-dependent) for stable, ambient-robust imaging and improved user comfort.
* **Multi-mode illumination:** **NIR flood** is optimized for close range (≈ **up to 1.2 m**). **VCSEL pattern projection** is optimized for medium range (≈ **1.2–10 m**) and can also help at closer distances in low-texture scenes.
* **Linux on the edge:** A Linux-based stack enables robust deployment and long-term maintainability (drivers, networking, tooling).
* **Ethernet-first networking:** Designed for **wired Ethernet** (PoE where applicable) to deliver predictable bandwidth, low jitter, and reliable multi-rig synchronization.
* **Bandwidth-efficient output:** Streams **3D keypoints, sparse ROI point clouds, and references** by default, without requiring raw video. An optional **H.265 preview** is available for monitoring and setup, and **RAW output** can be enabled when needed.
* **Scales across rigs:** Each device runs a **2-camera stereo pair at the edge**; combine **2–8 stereo rigs** over Ethernet to cover larger capture volumes and **significantly reduce occlusions** via multi-view geometry.
* **Low-latency pipeline:** Edge-side preprocessing and triangulation reduce host load and end-to-end latency.
* **Marker-optional:** Gloveless by default; optional **wristbands** can improve arm stability; **fingertip markers** enable the highest precision.
* **Optional tracked peripherals:** Supports tracked tools such as **VR headset markers, 3D pens/styluses, and props**, enabling high-precision workflows beyond hand tracking.
* **Optional GPU/NPU AI assist:** Used for robustness enhancements such as disambiguation, confidence estimation, and failure detection. In multi-view setups (**>2 cameras**), geometric redundancy typically reduces reliance on AI-based methods.
* **Optional RGB helper camera:** A center RGB camera can assist setup (visual inspection, text/marker reading, calibration aids) **without entering the reconstruction path**.

---

## System Overview

```
[ TDMStrobe ] ─► [ EdgeTrack ] ─► [ CoreFusion ] ─► your Application
```

---

## Bill of Materials (BOM)

**Core**

* **Raspberry Pi 5 (8 GB RAM)** — recommended; 4 GB is possible, 16 GB is optional
* **Cooling**: Pi 5 **active cooler** (heatsink + fan)
* **Storage**: microSD (≥ 64 GB) *or* NVMe (preferred for logs)

**Cameras (Stereo per Pi)**

* 2× **global‑shutter mono** modules (e.g., OV9281 1280×800 @ up to 120 fps), Source: 👉 [Sensor Guide](https://github.com/edgetrackorg/overview/blob/main/docs/sensor-guide.md)
* 2× **lenses** matched to FOV (target 60–90° for precision; 120° only for bring‑up), Source: 👉 [Vision Geometry Rules](https://github.com/edgetrackorg/overview/blob/main/docs/geo_rules.md)
* **850 nm band‑pass filters** (camera‑safe IR)
* **Rigid stereo mount** with baseline ~80–300 mm (context‑dependent)

**Lighting / Sync**

* **TDMStrobe** controller based on RP2040/Pico, providing A/B phases (C/D optional), Source: 👉 [TDMStrobe](https://github.com/edgetrackorg/tdmstrobe)

---

## Roadmap

Coming soon. 

---

## License

**Apache‑2.0** (code, firmware, docs). Hardware files (rig plates, brackets) may use **CERN‑OHL‑S**.

---

## Safety

### NIR Illumination (850 nm vs 940 nm) & Eye Safety

* **Never look into emitters.** Use black matte **baffles/shields**, aim emitters away from faces, and add **hardware interlocks** (LEDs off on loss of sync, open covers, or presence detection).
* Keep **exposure short** (strobe pulses strictly within camera exposure) and **average irradiance low**.
* Prefer **850 nm band-pass filters** on cameras to reduce the required LED output power.
* **850 nm and 940 nm are both IR-A** and are **not inherently eye-safe**; safety depends on irradiance, geometry, duty cycle, distance, and exposure time (IEC 62471).

### Solution Strategies

**Option A — Prefer more viewpoints over more power (recommended for 940 nm)**
- If **940 nm illumination** is preferred (reduced visible glow), the recommended approach is to **increase the number of stereo rigs (viewpoints)** to maintain SNR while keeping **irradiance low**, rather than compensating with higher-power NIR emitters.

**Option B — Side / rear placement (recommended)**
- Mount stereo pairs **left/right and slightly behind** the workspace, aimed toward the work area. Add **one or two top stereo pairs** for occlusion-free coverage. This directs NIR **away from the eyes** while maintaining uniform scene illumination.
Future refinement: recess-mount one stereo pair near the table center and another near the back edge for a slimmer, more robust setup.

**Option C — Front placement with HMD only**
- If stereo pairs must face forward, operate with a **closed VR headset** (no see-through optics) so the user’s eyes are **occluded**. Baffles and interlocks are still required to protect bystanders.

**Option D — IR-filtering safety eyewear**
- Use **visible-light-transmitting eyewear** that strongly attenuates **near-IR (≈ 780–950 nm)** (specified optical density at **850 nm / 940 nm**) so users retain normal vision while IR exposure is reduced.

**Option E — Side-shield eyewear (“horse-blinkers” concept)**
- Provide **IR-blocking safety glasses with side shields** for operators and visitors when emitters face forward. Ensure proper **near-IR attenuation ratings** and a snug fit to block off-axis radiation. and ensure a snug fit to block off-axis light.

---

## Disclaimer

Prototype hardware. Use at your own risk. Ensure eye‑safety and proper thermal design in all setups.