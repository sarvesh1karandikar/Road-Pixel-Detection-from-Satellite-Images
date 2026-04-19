# SETUP.md — Environment Requirements

This project is implemented in **MATLAB** using the GUIDE rapid GUI framework. There is no Python code; setup consists of ensuring the correct MATLAB version and toolboxes are installed.

---

## MATLAB Version

| Requirement | Value |
|-------------|-------|
| Minimum version | MATLAB R2013b |
| Tested version | MATLAB R2013b |
| Maximum compatible | MATLAB R2021b (GUIDE removed in R2022a — see note below) |

> **Note on MATLAB R2022a+:** The GUI components (`.fig` files and GUIDE-based `.m` files) use MATLAB GUIDE, which was removed in R2022a. On R2022a or later you will see a compatibility warning and may need to convert the GUIs to App Designer using `guide2appdesigner`. The underlying algorithm code (threshold, morphological ops, K-means) works on any version.

---

## Required Toolboxes

### Pipeline 1 — Connected Component Analysis

| Toolbox | Functions Used |
|---------|---------------|
| **Image Processing Toolbox** | `rgb2gray`, `bwmorph`, `bwconncomp`, `regionprops`, `labelmatrix`, `imclose`, `imdilate`, `imfuse`, `imwrite`, `imresize` |

### Pipeline 2 — Morphological + K-Means

| Toolbox | Functions Used |
|---------|---------------|
| **Image Processing Toolbox** | `imfilter`, `fspecial`, `histeq`, `imopen`, `imshow`, `im2double`, `imadd`, `imresize` |
| **Statistics and Machine Learning Toolbox** | `kmeans` |

### Adaptive Thresholding (optional)

| Toolbox | Functions Used |
|---------|---------------|
| **Image Processing Toolbox** | `mat2gray`, `imfilter`, `fspecial`, `medfilt2`, `im2bw` |

---

## Verifying Your Toolboxes

Run in the MATLAB Command Window:

```matlab
% Check Image Processing Toolbox
ver('images')

% Check Statistics and Machine Learning Toolbox
ver('stats')

% Quick smoke test
I = imread('cameraman.tif');
BW = imbinarize(I);
disp('Image Processing Toolbox OK')
```

---

## Input Image Format

- **Format:** JPEG (`.jpg`)
- **Colour space:** RGB (converted to greyscale internally)
- **Recommended resolution:** Any — images are auto-resized to **256 x 256** pixels internally
- **Content:** Nadir-view (straight-down) satellite imagery; roads should appear as bright linear features

---

## File Structure

```
Road-Pixel-Detection-from-Satellite-Images/
|
+-- Connected Component Analysis/
|   +-- ProjectGUI.m          # Main GUIDE app entry point (Pipeline 1)
|   +-- ProjectGUI.fig         # GUIDE layout file
|   +-- main2.m ... main8.m   # Pipeline step callbacks
|   +-- adaptivethreshold.m   # Optional local thresholding utility
|   +-- output10-17.jpg       # Sample output images
|
+-- Morphological Operation/
|   +-- main1.m               # Main GUIDE app entry point (Pipeline 2)
|   +-- main1.fig ... main8.fig
|   +-- main2.m ... main8.m   # Pipeline step callbacks
|   +-- thinning.m            # Custom Canny edge detector
|   +-- J1-17.jpg             # Intermediate result images
|
+-- ProjectDocument.docx      # Original project report
+-- README.md
+-- CLAUDE.md
+-- SETUP.md                  # This file
```

---

## Python Port (Future)

This project is MATLAB-only. A Python/OpenCV port is planned to enable a browser-based demo. When that port is complete, a `requirements.txt` will be added with:

```
opencv-python>=4.8
scikit-image>=0.21
scikit-learn>=1.3
numpy>=1.24
Pillow>=10.0
streamlit>=1.28      # for the web demo
```
