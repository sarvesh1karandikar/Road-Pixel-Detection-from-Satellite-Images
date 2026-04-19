# Road Pixel Detection from Satellite Images

![MATLAB](https://img.shields.io/badge/MATLAB-R2013%2B-orange?logo=mathworks)
![Image Processing Toolbox](https://img.shields.io/badge/Toolbox-Image%20Processing-blue?logo=mathworks)
![Computer Vision](https://img.shields.io/badge/Domain-Computer%20Vision-green)
![OpenCV Port](https://img.shields.io/badge/Port%20Potential-Python%20%2F%20OpenCV-yellow?logo=python)

**Classical computer vision pipeline that detects road pixels in satellite imagery using morphological operations, K-means colour segmentation, and Canny-style edge thinning — no neural network required.**

---

## What This Does

Given a satellite image (`.jpg`), the system identifies which pixels belong to roads and renders them as a coloured overlay on the original image. Two independent MATLAB GUIDE applications implement slightly different pipelines so you can compare approaches side by side.

The project demonstrates that strong computer vision results are achievable with deterministic signal-processing techniques — an important contrast to the deep-learning-heavy landscape of modern CV.

---

## Algorithm

### Pipeline 1 — Connected Component Analysis (`Connected Component Analysis/`)

| Step | MATLAB call | Purpose |
|------|------------|---------|
| 1. Load & resize | `imread`, `imresize([256,256])` | Normalise input |
| 2. Greyscale | `rgb2gray` | Reduce to luminance channel |
| 3. Global threshold | `gray > 150` | Binary segmentation of bright road pixels |
| 4. Noise removal | `bwmorph(bw, 'clean', 5)` | Remove isolated foreground pixels |
| 5. Majority filter | `bwmorph(bw1, 'majority', 10)` | Fill holes, smooth regions |
| 6. Connected components | `bwconncomp` + `regionprops('Area')` | Label each blob |
| 7. Area filter | `ismember(L, find([s.Area] >= 500))` | Discard small blobs (< 500 px) |
| 8. Second majority pass | `bwmorph(bw3, 'majority', 10)` | Re-smooth after area filter |
| 9. Morphological closing | `imclose(bw4, strel('disk', 3))` | Bridge small gaps in roads |
| 10. Dilation | `imdilate(bw5, se)` | Widen road skeleton |
| 11. Thinning | `bwmorph(bw6, 'thin', 2)` | Reduce road to near-centreline |
| 12. Overlay | `imfuse(gray, bw7)` | Fuse mask with original |

### Pipeline 2 — Morphological + K-Means (`Morphological Operation/`)

| Step | MATLAB call | Purpose |
|------|------------|---------|
| 1. Gaussian smoothing | `imfilter(img, fspecial('gaussian',[5 5],2))` | Suppress texture noise |
| 2. Histogram equalisation | `histeq` | Normalise contrast |
| 3. K-means segmentation | `kmeans` on L*a*b* colour space (k=3) | Cluster land cover types |
| 4. Region growing | `regiongrowing(I, x, y, 0.2)` | Expand seed point into homogeneous road region |
| 5. Morphological opening | `imopen(J, strel('disk',2))` | Remove protrusions |
| 6. Intersection mask | Pixel-wise AND of opening + region grow | Retain only high-confidence road pixels |
| 7. Edge thinning (Canny) | `thinning(F)` — custom Canny implementation | Produce single-pixel-wide road edges |
| 8. Colour overlay | Per-pixel RGB assignment (road pixels -> yellow/green highlight) | Final visualisation |

Adaptive thresholding (`adaptivethreshold.m`) is also available as an alternative to the global threshold in Step 3 of Pipeline 1. It uses a local window mean/median to handle uneven illumination across scenes.

---

## Results

The output is a fused image where detected road pixels are rendered as a coloured overlay (yellow-green tint) on the original greyscale satellite tile. Intermediate pipeline stages are shown in separate axes panels within the MATLAB GUI, letting you inspect each step individually:

- **Panel 1:** Original satellite image (256 x 256)
- **Panels 2-7:** Progressive binary masks through the pipeline
- **Panel 8:** Final colour overlay — roads highlighted over the original image

Example output images are stored in `Connected Component Analysis/` (output10.jpg through output17.jpg) and intermediate morphological steps in `Morphological Operation/` (J1.jpg through J17.jpg).

---

## Tech Stack

- **MATLAB** (tested with R2013b, GUIDE-based GUI)
- **Image Processing Toolbox** — `bwmorph`, `imclose`, `imdilate`, `bwconncomp`, `regionprops`, `imfuse`
- **Statistics and Machine Learning Toolbox** — `kmeans` (Pipeline 2 only)
- **Python / OpenCV** — not yet implemented, but a direct port is feasible (see Portfolio Note below)

---

## Setup & Running

### Prerequisites

- MATLAB R2013b or later
- Image Processing Toolbox
- Statistics and Machine Learning Toolbox (Pipeline 2 only)

### Running Pipeline 1 (Connected Component Analysis)

```matlab
cd 'Connected Component Analysis'
ProjectGUI
```

1. Click **Load Image** and select any `.jpg` satellite tile.
2. Step through buttons 1-7 to execute the pipeline stage by stage.
3. Panel 8 shows the final road overlay.

### Running Pipeline 2 (Morphological + K-Means)

```matlab
cd 'Morphological Operation'
main1
```

Follow the same step-through buttons: Gaussian smooth -> histogram equalise -> K-means segment -> region grow -> morph open -> intersection mask -> thinning -> overlay.

---

## Portfolio Note — Live Demo Vision

A compelling live demo would look like this:

> **Upload any satellite image -> detected roads shown as a red/yellow overlay in the browser**

To make that happen, the MATLAB pipeline needs to be ported to **Python + OpenCV** first. The mapping is direct:

| MATLAB | Python / OpenCV equivalent |
|--------|---------------------------|
| `rgb2gray` | `cv2.cvtColor(..., cv2.COLOR_RGB2GRAY)` |
| `bwmorph('majority')` | `cv2.morphologyEx(MORPH_CLOSE)` |
| `bwconncomp` + area filter | `cv2.connectedComponentsWithStats` |
| `imclose`, `imdilate` | `cv2.morphologyEx`, `cv2.dilate` |
| `kmeans` on L*a*b* | `sklearn.cluster.KMeans` |
| Canny thinning | `cv2.Canny` |

Once the Python port exists, a **Streamlit** front-end (file uploader -> processed image side-by-side) can be deployed to **HuggingFace Spaces** for free — zero infrastructure, instantly shareable.

---

## What I Learned / Key Insights

1. **Morphological sequencing matters.** Applying `majority` before and after the area filter produces cleaner masks than a single pass — the first pass smooths noise, the area filter removes false positives, and the second pass re-fills gaps created by removal.

2. **K-means on L*a*b* outperforms RGB thresholding** for scenes with variable lighting because the L* channel separates luminance from chrominance, making road-vs-vegetation clustering more stable.

3. **Connected component area filtering is a surprisingly powerful false-positive suppressor.** Roads form large connected blobs; vegetation and building artefacts tend to be small. A single 500-pixel area threshold removes most noise without a complex classifier.

4. **Classical CV is still competitive for constrained domains.** On well-lit, nadir-view satellite imagery with clear road markings, this deterministic pipeline matches the visual quality of lightweight deep-learning approaches while being fully interpretable and requiring no training data.

5. **GUI-first development aids debugging.** Building the pipeline as a step-through MATLAB GUIDE application lets intermediate results be inspected visually at each stage, drastically reducing iteration time during parameter tuning.
