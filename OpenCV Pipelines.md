---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [perception, opencv, computer-vision]

# OpenCV Pipelines


---

## Core idea

Classical computer vision = hand-engineered image operations (filters, thresholds, transforms, feature detectors) with no learning. Still the right tool when the problem is well-defined, compute is tight, or you need determinism and explainability — exactly the edge/embedded constraints in robotics. Learned CV ([[YOLOv8 Detection]]) wins on general semantic recognition; classical wins on speed, predictability, and simple geometric targets.

## Key facts & formulas

- **Pipeline shape**: capture → preprocess (resize, blur/denoise) → color/feature extraction → threshold/segment → morphology (erode/dilate to clean noise) → contours/blobs → geometry (centroid, bounding box) → decision.
- **Color segmentation in HSV, not RGB**: HSV separates _hue_ (color) from _brightness_, so a threshold on hue is robust to lighting changes — RGB thresholds break the moment lighting shifts. The standard trick for "find the red object."
- **Feature detectors**: edges (Canny), corners (Harris), keypoints+descriptors (ORB/SIFT/SURF) for matching/tracking across frames — the front-end of classical visual odometry and the descriptor side of [[SLAM & RTAB-Map]].
- **Morphological ops**: erosion/dilation/open/close to remove speckle and fill holes in a binary mask before contour extraction.
- **Camera model**: pinhole + intrinsics (focal length, principal point) and distortion coefficients; calibration (checkerboard) maps pixels to rays — prerequisite for any metric vision.


## Links

- Related: [[YOLOv8 Detection]], [[SLAM & RTAB-Map]], [[CUDA & CuPy]], [[Jetson Orin Setup]], [[Sensor Fusion]]
- Parent: [[00 Knowledge Map]]

---

