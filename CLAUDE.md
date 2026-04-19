# CLAUDE.md — Road Pixel Detection from Satellite Images

## Project Summary

Classical computer vision project that detects road pixels in satellite images without any deep learning. Two MATLAB GUIDE applications implement parallel pipelines:

- **Pipeline 1** (`Connected Component Analysis/ProjectGUI.m`): global threshold -> morphological cleaning -> connected component area filtering -> dilation/thinning -> overlay
- **Pipeline 2** (`Morphological Operation/main1.m`): Gaussian smooth -> histogram equalise -> K-means (L*a*b*, k=3) -> region growing -> morphological opening -> intersection mask -> Canny thinning -> overlay

Both pipelines produce a final panel where detected road pixels are rendered as a coloured (yellow/green) overlay on the original greyscale satellite tile.

---

## How to Run

### Requirements

- MATLAB R2013b or later
- Image Processing Toolbox (both pipelines)
- Statistics and Machine Learning Toolbox (Pipeline 2 — `kmeans`)

### Pipeline 1

```matlab
cd 'Connected Component Analysis'
ProjectGUI          % launches GUIDE GUI
```

Click "Load Image", select a `.jpg` satellite file, then step through buttons 1-8.

### Pipeline 2

```matlab
cd 'Morphological Operation'
main1               % launches GUIDE GUI
```

Step through buttons in order: load image -> Gaussian smooth -> histogram equalise -> K-means -> region grow -> morph open -> intersection -> thinning -> overlay.

---

## Algorithm Details (read from .m files)

### Pipeline 1 — Connected Component Analysis

```
1.  rgb2gray(image)                          -- greyscale
2.  gray > 150                               -- global threshold
3.  bwmorph(bw,  'clean',    5)              -- remove isolated pixels
4.  bwmorph(bw1, 'majority', 10)             -- fill holes / smooth
5.  bwconncomp(bw2)                          -- label connected blobs
6.  regionprops(cc, 'Area')                  -- measure blob areas
7.  ismember(L, find([s.Area] >= 500))       -- keep blobs >= 500px
8.  bwmorph(bw3, 'majority', 10)             -- re-smooth
9.  imclose(bw4, strel('disk', 3))           -- close small gaps
10. imdilate(bw5, strel('disk', 3))          -- widen roads
11. bwmorph(bw6, 'thin', 2)                  -- reduce to centreline
12. imfuse(gray, bw7)                        -- colour overlay
```

`adaptivethreshold.m` implements local window mean/median thresholding as an alternative to the global `gray > 150` step. Parameters: window size `ws`, constant offset `C`, mean/median toggle `tm`.

### Pipeline 2 — Morphological + K-Means

```
1. imfilter(img, fspecial('gaussian',[5 5],2))  -- Gaussian smooth
2. rgb2gray; histeq(img)                         -- greyscale + hist eq
3. makecform('srgb2lab'); kmeans(ab, 3)          -- K-means on a*b* channels
4. regiongrowing(I, seedX, seedY, 0.2)           -- region growing
5. imopen(J, strel('disk', 2))                   -- morph open (disk, r=2)
6. imopen(J, strel('line', 1, 45))               -- morph open (line, 45 deg)
7. pixel-wise AND of steps 4,5,6                 -- intersection mask
8. thinning(F)  -- custom Canny (sobel/prewitt/roberts/LoG/canny modes)
9. per-pixel RGB: road pixels -> im2(i,j,2/3)=255, im2(i,j,1)=0  -- yellow overlay
```

`thinning.m` is a full custom edge-detection implementation supporting Sobel, Prewitt, Roberts, LoG, and Canny methods with non-maximum suppression and hysteresis thresholding.

---

## Current Limitations

1. **Fixed 256x256 resize.** All inputs are hard-resized to 256x256. Larger tiles require re-parameterising the area threshold and structuring element sizes.
2. **Hard-coded threshold (150).** The global threshold works on the included test images but will fail on darker or overcast satellite imagery without manual tuning.
3. **Hard-coded region-growing seed (x=10, y=20).** Pipeline 2 uses a fixed seed point. Works for the included images but breaks on arbitrary inputs.
4. **No generalisation validation.** The pipeline was developed and tested on a small, specific dataset. Performance on other regions/resolutions is unknown.
5. **MATLAB GUIDE deprecated.** GUIDE was removed in MATLAB R2022a. Running on newer MATLAB requires converting to App Designer.
6. **No quantitative evaluation.** Accuracy vs. ground truth is shown only as a bar chart (`bar(x)`) for a single comparison; no precision/recall/IoU metrics.
7. **Not deployable.** MATLAB license required — no browser or server demo is possible without porting to Python.

---

## Enhancement TODO

### Python OpenCV Port (KEY — unlocks demo-ability)

Direct function mapping:

| MATLAB | Python equivalent |
|--------|------------------|
| `rgb2gray` | `cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)` |
| `bwmorph('clean')` | `cv2.morphologyEx(cv2.MORPH_OPEN, kernel_1)` |
| `bwmorph('majority')` | `cv2.morphologyEx(cv2.MORPH_CLOSE, kernel_large)` |
| `bwconncomp` + area filter | `cv2.connectedComponentsWithStats`, filter by `CC_STAT_AREA` |
| `imclose`, `imdilate` | `cv2.morphologyEx(MORPH_CLOSE)`, `cv2.dilate` |
| `bwmorph('thin')` | `skimage.morphology.thin` or `cv2.ximgproc.thinning` |
| `kmeans` on L*a*b* | `cv2.cvtColor(COLOR_BGR2Lab)` + `sklearn.cluster.KMeans` |
| `regiongrowing` | `skimage.segmentation.flood` |
| Canny | `cv2.Canny` |
| `imfuse` / overlay | NumPy array channel assignment |

Target file: `road_detection.py` — single function `detect_roads(image_path) -> np.ndarray` returning the overlay image.

### Streamlit UI

```python
# app.py skeleton
import streamlit as st
from road_detection import detect_roads

uploaded = st.file_uploader("Upload satellite image", type=["jpg","png"])
if uploaded:
    result = detect_roads(uploaded)
    col1, col2 = st.columns(2)
    col1.image(original, caption="Input")
    col2.image(result,   caption="Detected Roads")
```

### HuggingFace Spaces Deployment

After Streamlit UI works:
1. Create `requirements.txt` (opencv-python, scikit-image, scikit-learn, streamlit, Pillow)
2. Push to HuggingFace Spaces as a Streamlit Space
3. Set `app_file: app.py` in README YAML front-matter

---

## Recommended Demo Tier

**Medium Lift** — estimated 2-3 days of focused work.

**Justification:** The Python OpenCV port is the only blocker, and every MATLAB function has a well-documented Python/OpenCV equivalent. The algorithm logic is simple and procedural (no model weights, no training data), so porting is mainly a translation exercise. Once `road_detection.py` exists, wiring Streamlit and deploying to HuggingFace Spaces is a Quick Win (< 2 hours). The Medium Lift rating accounts for the port itself and testing on a diverse set of satellite tiles to validate that the hard-coded parameters (threshold=150, area=500, seed point) are replaced with robust auto-detection.

A Quick Win alternative (no Python port): record a screen capture of the MATLAB GUI running on sample images and embed it as a GIF in the README. This adds visual proof-of-concept without any coding.
