---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [perception, opencv, computer-vision]

# OpenCV Pipelines

> [!question] Explain it cold
> 
> - What's the difference between classical CV and learned CV, and when do you pick classical?
> - Walk a basic color/feature detection pipeline.
> - What's the color-space trick that makes color segmentation robust?

---

## Core idea

Classical computer vision = hand-engineered image operations (filters, thresholds, transforms, feature detectors) with no learning. Still the right tool when the problem is well-defined, compute is tight, or you need determinism and explainability — exactly the edge/embedded constraints in robotics. Learned CV ([[YOLOv8 Detection]]) wins on general semantic recognition; classical wins on speed, predictability, and simple geometric targets.

## Key facts & formulas

- **Pipeline shape**: capture → preprocess (resize, blur/denoise) → color/feature extraction → threshold/segment → morphology (erode/dilate to clean noise) → contours/blobs → geometry (centroid, bounding box) → decision.
- **Color segmentation in HSV, not RGB**: HSV separates _hue_ (color) from _brightness_, so a threshold on hue is robust to lighting changes — RGB thresholds break the moment lighting shifts. The standard trick for "find the red object."
- **Feature detectors**: edges (Canny), corners (Harris), keypoints+descriptors (ORB/SIFT/SURF) for matching/tracking across frames — the front-end of classical visual odometry and the descriptor side of [[SLAM & RTAB-Map]].
- **Morphological ops**: erosion/dilation/open/close to remove speckle and fill holes in a binary mask before contour extraction.
- **Camera model**: pinhole + intrinsics (focal length, principal point) and distortion coefficients; calibration (checkerboard) maps pixels to rays — prerequisite for any metric vision.

## Where I've used it

- **WALL-E V3**: OpenCV pipelines on the Jetson for the camera processing feeding the semantic-nav side alongside [[YOLOv8 Detection]]. Classical CV handles the cheap, deterministic preprocessing; YOLO handles semantic detection. Ran in the Jetson/CUDA context ([[CUDA & CuPy]], [[Jetson Orin Setup]]).
- **Kapila final**: OpenCV pipelines on Jetson came up explicitly (alongside CuPy RawKernel, CNN inference).

## Interview follow-ups

- **Q:** When classical CV over a neural net?
    - **A:** Well-defined geometric/color targets, tight compute or latency budgets, need for determinism/explainability, or no training data. A color blob or fiducial marker doesn't need a CNN — classical is faster and predictable on the edge.
- **Q:** Why HSV for color detection?
    - **A:** HSV separates hue from brightness, so a hue threshold survives lighting changes; RGB mixes color and intensity, so RGB thresholds break under changing light.
- **Q:** Clean up a noisy binary mask?
    - **A:** Morphological opening (erode then dilate) to remove speckle, closing to fill holes, then contour extraction for blobs/centroids.

## Gotchas / what trips me up

- Thresholding in RGB and wondering why it fails under lighting change.
- Skipping calibration then expecting metric/geometric results.

## Links

- Related: [[YOLOv8 Detection]], [[SLAM & RTAB-Map]], [[CUDA & CuPy]], [[Jetson Orin Setup]], [[Sensor Fusion]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

When do you pick classical CV over a neural net? ? Well-defined geometric/color targets, tight compute/latency, need for determinism/explainability, or no training data. A color blob or fiducial doesn't need a CNN.

Why segment color in HSV instead of RGB? ? HSV separates hue from brightness, so a hue threshold is robust to lighting changes; RGB mixes color and intensity so its thresholds break under changing light.

Typical classical CV pipeline stages? ? Capture → preprocess (resize/blur) → color/feature extraction → threshold/segment → morphology (erode/dilate) → contours/blobs → geometry (centroid/bbox) → decision.