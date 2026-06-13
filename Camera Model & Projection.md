---
type: concept
domain: Perception & ML
status: drafted
last-reviewed: 2026-06-02
tags: [camera, pinhole, intrinsics, extrinsics, projection, calibration, stereo]
---

# Camera Model & Projection


---

## Pinhole camera model

A pinhole camera maps a 3D world point to a 2D image point by projecting through a single point (the optical center / principal point).

```
3D point P = (X, Y, Z) in camera frame
Image point p = (u, v) in pixels

u = fx * (X/Z) + cx
v = fy * (Y/Z) + cy
```

Where:
- `fx, fy` — focal lengths in pixels (fx ≈ fy for square pixels)
- `cx, cy` — principal point (image center, usually width/2, height/2)
- `Z` — depth (distance along optical axis)

---

## Intrinsic matrix K

```
    [fx  0  cx]
K = [ 0 fy  cy]
    [ 0  0   1]

Projection: p = K * [X/Z, Y/Z, 1]ᵀ
```

For RealSense D435 (640x480):
- fx ≈ fy ≈ 614 pixels (from camera_info topic)
- cx ≈ 320, cy ≈ 240

---

## Extrinsic matrix [R|t]

Transforms a point from world frame to camera frame:
```
P_cam = R * P_world + t
```

- `R` — 3x3 rotation matrix (camera orientation relative to world)
- `t` — 3x1 translation (camera position in world)

Full projection: `p = K * [R|t] * P_world`

---

## Distortion

Real lenses aren't perfect pinholes. Distortion bends straight lines.

**Radial distortion** — radially symmetric around principal point (cx, cy). Two forms:
- **Barrel** — lines bow *outward* toward edges. Common in wide-angle/fisheye lenses. Typical in robotics cameras.
- **Pincushion** — lines pinch *inward* toward center. Common in telephoto/zoom lenses.

```
x_d = x(1 + k1*r² + k2*r⁴ + k3*r⁶)
y_d = y(1 + k1*r² + k2*r⁴ + k3*r⁶)
```

Coefficients: `k1, k2, k3`

**Tangential distortion** — caused by lens elements not perfectly parallel to the image sensor (mechanical misalignment). Makes the image look *skewed or shifted asymmetrically* toward one side or corner — unlike radial which is symmetric, tangential shears the image in a direction.
```
x_d = x + 2*p1*x*y + p2*(r² + 2x²)
y_d = y + p1*(r² + 2y²) + 2*p2*x*y
```

Coefficients: `p1, p2`

Distortion coefficients: `[k1, k2, p1, p2, k3]` — output of `cv2.calibrateCamera()`

---

## Camera calibration

Calibration finds K and distortion coefficients from a set of images of a known pattern (checkerboard).

```python
import cv2
import numpy as np

# Find corners in checkerboard images
objpoints = []  # 3D points in real world (z=0 for flat board)
imgpoints = []  # 2D points in image plane

for img in calibration_images:
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    ret, corners = cv2.findChessboardCorners(gray, (9, 6), None)
    if ret:
        objpoints.append(object_points)  # known 3D pattern
        imgpoints.append(corners)

# Calibrate
ret, K, dist, rvecs, tvecs = cv2.calibrateCamera(
    objpoints, imgpoints, gray.shape[::-1], None, None)

# K is the intrinsic matrix, dist is distortion coefficients
```

**RMS reprojection error < 0.5 pixels** = good calibration.

---

## Depth to 3D (backprojection)

Given a depth image pixel `(u, v)` with depth `d`:
```python
X = (u - cx) * d / fx
Y = (v - cy) * d / fy
Z = d
```

This is exactly what RTAB-Map and the YOLO nav node do: pixel + depth → 3D camera frame → TF → map frame.

---

## Stereo vision

Two cameras, known baseline `b`. Disparity `d = u_left - u_right`.

```
Z = fx * b / d
```

Closer objects → larger disparity → Z is smaller.
RealSense D435 uses IR stereo (structured light) internally to compute depth — you get the depth image directly.

---

## Common interview questions

**Q: How does a depth camera give you 3D points?**
A: RGB pixel (u,v) + depth Z → backproject: X=(u-cx)*Z/fx, Y=(v-cy)*Z/fy. Transform to world frame via TF.

**Q: What is focal length in pixels?**
A: Physical focal length (mm) divided by pixel size (mm/pixel). Relates angular field of view to image size. Higher fx = narrower FOV, larger objects in image.

**Q: What happens if you use wrong intrinsics?**
A: 3D points project to wrong pixel positions. SLAM features mismatch, depth backprojection is wrong, robot navigates to wrong goals.

---

## Links
- Related: [[OpenCV Pipelines]], [[YOLOv8 Detection]], [[SLAM & RTAB-Map]], [[Sensor Fusion]], [[Odometry vs Localization vs SLAM]]
- Parent: [[00 Knowledge Map]]

---

