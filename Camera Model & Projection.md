---
type: concept
domain: Perception & ML
status: drafted
last-reviewed: 2026-06-02
tags: [camera, pinhole, intrinsics, extrinsics, projection, calibration, stereo]
---

# Camera Model & Projection

> [!question] Explain it cold
> - What is the pinhole camera model and what are intrinsic parameters?
> - How do you project a 3D point into pixel coordinates?
> - What does camera calibration actually compute?

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

**Radial distortion** (barrel/pincushion):
```
x_d = x(1 + k1*r² + k2*r⁴ + k3*r⁶)
y_d = y(1 + k1*r² + k2*r⁴ + k3*r⁶)
```

**Tangential distortion** (lens not parallel to image plane):
```
x_d = x + 2*p1*x*y + p2*(r² + 2x²)
y_d = y + p1*(r² + 2y²) + 2*p2*x*y
```

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

#flashcards

Pinhole projection formula — 3D point (X,Y,Z) to pixel (u,v)? ? u = fx*(X/Z) + cx, v = fy*(Y/Z) + cy. Where fx,fy are focal lengths in pixels and cx,cy is the principal point (image center).

What do camera intrinsics represent physically? ? fx,fy: focal length in pixels — how much a unit displacement in 3D maps to pixels. cx,cy: principal point — where the optical axis hits the image plane (usually center). Together they form the K matrix.

How do you get a 3D point from a depth image pixel? ? Backproject: X=(u-cx)*depth/fx, Y=(v-cy)*depth/fy, Z=depth. This reverses the projection equation using the known depth value.

What does camera calibration compute and what's a good reprojection error? ? Calibration finds the intrinsic matrix K and distortion coefficients [k1,k2,p1,p2,k3] from checkerboard images using cv2.calibrateCamera(). RMS reprojection error < 0.5 pixels = good calibration.
