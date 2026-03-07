# EdgeTrack Architecture: Dual-Resolution Stereo with ROI Point Cloud

The goal of this pipeline is **deterministic, geometry-first 3D acquisition with very low CPU load**.

Instead of computing dense depth everywhere, EdgeTrack generates **a small number of high-quality 3D points per frame (≈ 200–2,000)** only in relevant regions of interest (ROI), such as **hands, tools, or interaction areas**.

---

# 1) Sensor & Capture Configuration (High-Resolution Master Stream)

**Input (per stereo camera):**

* Resolution: **1280 × 800**
* Frame rate: **120 FPS**
* Shutter: **Global Shutter**
* Format: **RAW10** (synchronized)
* Illumination: **NIR + bandpass filter**
* Exposure: **controlled / deterministic** (timing and exposure are predictable)

**Output**

```
L_raw10[t]
R_raw10[t]
```

Timestamped and synchronized stereo frames.

---

# 2) Low-Resolution Downscale Stream (for Coarse Dense / Motion)

From the high-resolution RAW10 stream, a low-resolution image is derived for each frame.

* Downscale: **1280 × 800 → 320 × 200** (exact **×4 scaling** in both axes)
* Format: **RAW8 / 8-bit** (optimized for fast matching operations)
* Optional: **4×4 box filter** instead of nearest neighbor
  (more stable texture, reduced aliasing)

**Output**

```
L_320[t]
R_320[t]
```

**Note**

The downscale operation is extremely lightweight and remains perfectly synchronized because it is derived from the same frame.

---

# 3) Preprocessing & Rectification

Stereo matching requires **rectified images**, where epipolar lines are horizontal.

Steps:

* Calibration (intrinsics / extrinsics) performed once and periodically verified
* Rectification maps:

High-resolution:

```
rectify_1280
```

Low-resolution:

```
rectify_320
```

**Output**

```
L_320_rect[t]
R_320_rect[t]

L_1280_rect[t]
R_1280_rect[t]
```

Optionally only ROI crops are rectified in high resolution.

---

# 4) Stage A: Coarse Dense Disparity (320 × 200 @120 FPS)

This stage intentionally computes **dense disparity at low resolution** to obtain a fast and robust global overview.

Possible algorithms:

* **SAD**
* **Census transform**
* lightweight **SGBM (reduced configuration)**

Typical configuration:

* Limited disparity range
* Example working distance ~0.5 m → ~30 px disparity at 320 width

Additional outputs:

* validity mask
* cost / confidence estimate

**Output**

```
D_320[t]   → disparity map
C_320[t]   → confidence / cost
V_320[t]   → valid match mask
```

---

# 5) Stage B: Motion / Target Detection (ROI Candidates)

Goal: **detect where motion occurs**, such as hands, tools, or objects.

A minimal and robust approach:

Motion energy:

```
M_320[t] = abs(L_320_rect[t] − L_320_rect[t−1])
```

Then:

* thresholding
* morphological filtering (remove noise / fill small holes)
* combine with `V_320[t]` (regions with valid disparity)

Connected component analysis produces bounding boxes.

**Output**

```
ROI_320[t] = {roi_i}
```

Each ROI contains:

* bounding box in 320 coordinates
* motion score
* coarse disparity estimate

```
d_pred_320
```

(e.g., median disparity in ROI)

* confidence score (e.g., median of `C_320`)

---

# 6) ROI Tracking & Stabilization

ROIs should remain **temporally stable** and not jump between frames.

Methods:

* hysteresis (ROI persists even if score slightly drops)
* smoothing of ROI size and position
* simple bounding box tracking
* optional feature tracking inside the ROI

Optional:

* **Kalman filter** for center position and disparity prediction

**Output**

```
ROI_320_stable[t]
d_pred_320_stable
```

---

# 7) Mapping to High Resolution (1280 × 800)

Because the scale factor is exactly **4**, coordinates map directly:

```
x_hi = 4 × x_lo
y_hi = 4 × y_lo
w_hi = 4 × w_lo
h_hi = 4 × h_lo
```

Disparity also scales in pixel space:

```
d_pred_hi = 4 × d_pred_320
```

**Output**

```
ROI_1280[t]
d_pred_hi[t]
```

---

# 8) Stage C: High-Resolution ROI Disparity Refinement

(1280 × 800 @120 FPS, ROI Only)

Precision refinement occurs only where necessary.

Process:

1. Extract high-resolution ROI crops from

```
L_1280_rect[t]
R_1280_rect[t]
```

2. Perform disparity search only around the predicted value:

```
d ∈ [d_pred_hi − Δ , d_pred_hi + Δ]
```

Typical values:

* Δ = **8–16 px** when confidence is high
* Δ expanded when confidence is low

Important:

The **tracking ROI can remain small** (e.g., 30×30 pixels), but the **matching strip must be wider horizontally** to allow disparity search.

Example:

```
match_strip ≈ 120 × 16
or
160 × 16
```

depending on search range and patch size.

Possible outputs:

* small local disparity map
* sparse disparity samples

**Output**

```
D_roi_hi[t]     → local disparity map
or
P_roi_hi[t]     → sparse disparity samples
```

---

# 9) 3D Reconstruction: ROI Point Cloud

Disparity is converted to metric 3D coordinates.

Depth:

```
Z = f × B / d
```

Coordinates:

```
X = (x − cx) × Z / f
Y = (y − cy) × Z / f
```

Where:

* `f, cx, cy` come from calibration
* `B = 80 mm` baseline

Point selection strategy (to keep point count small):

Per ROI:

* grid sampling (e.g., 10 × 10)
* feature points (corners)
* best-K pixels based on confidence

Target density:

```
≈ 200 – 2,000 points per frame
```

**Output**

```
Cloud_roi[t]
```

Sparse point set with confidence values.

---

# 10) Post-Filtering (Lightweight & Deterministic)

To reduce jitter and remove outliers:

Possible filters:

* left-right consistency check
* confidence thresholding
* outlier rejection
* median filtering (on sparse samples)
* temporal smoothing

Examples:

* exponential moving average
* Kalman filter on ROI centers or keypoints

**Output**

```
Cloud_roi_filtered[t]
```

---

# 11) Output to CoreFusion / MotionCoder

**EdgeTrack (edge devices) outputs:**

```
Cloud_roi_filtered[t]
```

plus:

* ROI metadata
* bounding boxes
* timestamps
* confidence values

Optional:

* pre-estimated keypoints

---

### CoreFusion (host side)

CoreFusion can then perform:

* multi-rig fusion
* temporal smoothing
* outlier rejection
* 3D keypoint fitting
* confidence estimation
* export to gesture-ready structures

for **MotionCoder**.

---

# Summary

This architecture combines:

* **High-resolution RAW10 capture** (quality & deterministic timing)
* **Low-resolution dense disparity @120 FPS** as a global predictor
* **High-resolution ROI refinement @120 FPS** for precision
* **Sparse ROI point clouds** instead of expensive full dense depth

Final output:

**Hundreds to a few thousand high-confidence 3D points per frame**, ideal for stable keypoint extraction and gesture interaction systems.

---