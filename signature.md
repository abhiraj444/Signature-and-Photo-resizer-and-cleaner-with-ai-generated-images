# Project Specification: SignVec (Signature & Passport Photo Processor)

This document outlines the architectural design, user workflows, mathematical formulations, and external API integrations for **SignVec**. It is designed to serve as a comprehensive blueprint for developer maintenance, future refactoring, and feature extensions.

---

## 1. High-Level System Architecture

SignVec is a zero-dependency, single-file HTML5 web application designed to run entirely client-side. It serves two distinct utility pipelines:
1. **Signature Vectorizer Mode**: Clean, level, erase smudges, vectorize, and export handwritten signatures into scalable vectors (SVG) or formatted, transparent raster images (PNG/JPG).
2. **Passport/Portrait Photo Mode**: Level, crop, and scale portrait photographs to exact dimensional boundaries (mm, inch, px) and strict target file size (KB) constraints, with an optional multimodal AI-powered passport attire generation grid.

### Stage-Based Single-Page App (SPA) Model
The application manages its state and UI progressively through five navigation stages. Transition states are handled on the main thread by toggling the CSS `.visible` class across layout panels and updating progress pills.

```
[ Stage 0: Upload ] ──► [ Stage 1: Crop & Rotate ] ──► [ Stage 2: Clean/Erase ]* ──► [ Stage 3: Vector Preview ]* ──► [ Stage 4: Export ]
                                                     (*Stages bypassed in Photo Mode)
```

---

## 2. Core Operational Modes & Pipelines

### 2.1 Signature Vectorizer Mode
Designed to convert noisy physical paper signature photos into lightweight, clean assets.
* **Stage 0 (Upload)**: File ingestion (PNG, JPG, WebP).
* **Stage 1 (Crop & Straighten)**: Alignment and boundary framing.
* **Stage 2 (Clean & Threshold)**: Dominant background detection, binarization, morphological cleaning, and interactive smudge lasso erasure.
* **Stage 3 (Vectorization)**: Marching squares boundary contour tracing, Chaikin smoothing, and outline stroke-weight adjustments.
* **Stage 4 (Export)**: Canvas vector rasterization or SVG export. Dimensions can be scaled to exact px, mm, or inch sizes with dynamic target KB bisection size limits.

### 2.2 Portrait / Photo Mode
Designed for passport photo compliance.
* **Stage 0 (Upload)**: Face photograph ingestion.
* **Stage 1 (Crop & Straighten)**: Standard aspect ratio framing presets (`1:1`, `3.5:4.5`, `3:4`). Optional trigger for Google Gemini AI Multimodal attire grid generation.
* **Stage 2 & 3**: Bypassed automatically.
* **Stage 4 (Export)**: Output configuration with standard passport chips, target size constraint limit optimization, and no-letterbox stretch-fill scaling.

---

## 3. Trigonometric Image Straightening & Cropping Algorithm

To prevent image shearing or coordinate clipping when an image is rotated on Stage 1, a two-pass coordinate mapping system is implemented.

### 3.1 Display Rendering
When the user adjusts the level slider ($\theta \in [-180^\circ, 180^\circ]$), the original unrotated image is drawn centered on a dynamically sized virtual bounding canvas. The width ($rotW$) and height ($rotH$) of the rotated boundaries are calculated using the absolute values of the trigonometric sine and cosine functions:

$$rotW = |w \cdot \cos(\theta)| + |h \cdot \sin(\theta)|$$

$$rotH = |w \cdot \sin(\theta)| + |h \cdot \cos(\theta)|$$

A low-resolution display canvas renders this rotated preview, updating the source image of the cropping DOM viewport (`cropImg`).

### 3.2 High-Resolution Crop Translation
The crop selector box coordinates are recorded relative to the rotated screen-space display boundaries. To perform a mathematically exact crop without losing quality, the app maps these display coordinates back to a high-resolution, rot-bounded canvas generated from the natural, unscaled dimensions of the original image:

$$\text{Scale}_x = \frac{\text{rotW}_{\text{natural}}}{\text{DisplayWidth}}$$

$$\text{Scale}_y = \frac{\text{rotH}_{\text{natural}}}{\text{DisplayHeight}}$$

$$ix = (b_x - \text{DisplayLeft}) \cdot \text{Scale}_x$$

$$iy = (b_y - \text{DisplayTop}) \cdot \text{Scale}_y$$

$$iw = b_w \cdot \text{Scale}_x$$

$$ih = b_h \cdot \text{Scale}_y$$

These mapped coordinates ($ix, iy, iw, ih$) are used to crop the high-resolution rotated canvas directly into `state.croppedCanvas`.

---

## 4. Interactive Mask Processing & Smudge Eraser

### 4.1 Lightweight Preview Processing
Running 30x30 nested morphological loops for erosion/dilation or calculating active pixel arrays on multi-megapixel images in real time can trigger browser main-thread freezes. To avoid this:
1. Upon exiting Step 1, a processing preview canvas (`state.previewCanvas`) is generated, capping the maximum dimension at `800px` while preserving aspect ratios.
2. All real-time Stage 2 slider changes (Sensitivity, Noise, Edge Soften) and manual smudge erasures are calculated instantly on this preview canvas.

### 4.2 Binarization & Dominant Background Detection
Binarization isolates ink from paper by finding the peak of the grayscale histogram in the upper 60% of the luminance spectrum, representing the dominant paper color. Pixels deviating from this peak beyond the Sensitivity threshold ($T$) are flagged as foreground ink.

### 4.3 Persistent Lasso Eraser
Manual erasures are stored as an array of coordinate polygon arrays (`state.eraserPolygons`).
* **Interactive Coordinate Mapping**: The user draws a lasso path on screen. Bounding client rect calculations map the screen coordinates $(X_{\text{screen}}, Y_{\text{screen}})$ to the preview canvas pixel dimensions:

$$X_{\text{canvas}} = (X_{\text{screen}} - \text{Rect}_{\text{left}}) \cdot \frac{\text{CanvasWidth}}{\text{Rect}_{\text{width}}}$$

$$Y_{\text{canvas}} = (Y_{\text{screen}} - \text{Rect}_{\text{top}}) \cdot \frac{\text{CanvasHeight}}{\text{Rect}_{\text{height}}}$$

* **Re-draw Loop**: Each time a range slider is adjusted, `computeMask()` recalculates the raw binarized canvas and then sequentially paints all saved `state.eraserPolygons` with `#ffffff` (white background color) on top, preserving manual changes.

---

## 5. Contour Extraction & Vectorization Math

### 5.1 Marching Squares Boundary Tracing
Contours are extracted from the binarized preview mask via a Marching Squares boundary-tracing algorithm. The canvas is converted into a binary 2D grid where foreground ink is evaluated as `1` and background paper is `0`. Active edge pixels with neighboring `0` blocks trigger the `traceBoundary()` loop, creating a list of closed loops.

### 5.2 Ramer-Douglas-Peucker (RDP) Simplification
To reduce SVG file size, raw marching loops are simplified using the RDP recursive algorithm. Points deviating from a straight line segment by less than a distance threshold $\epsilon = 1.2$ pixels are dropped:

$$d = \frac{|(y_2 - y_1)x_p - (x_2 - x_1)y_p + x_2 y_1 - y_2 x_1|}{\sqrt{(y_2 - y_1)^2 + (x_2 - x_1)^2}}$$

### 5.3 Chaikin's Corner-Cutting Smoothing
Simplified vertices undergo Chaikin smoothing. For each segment from $Q$ to $R$, new points are cut at $3/4$ and $1/4$ ratios along the vector:

$$P = 0.75Q + 0.25R$$

$$S = 0.25Q + 0.75R$$

This process is repeated $N$ times (derived from the "Path Smoothing" range slider) to generate continuous curves.

### 5.4 Tight viewport ViewBox Wrapping
To prevent output vectors from rendering on a large empty canvas with the signature offset in a corner, a bounding box loop evaluates the absolute limits of all processed path coordinates:

$$\text{minX} = \min(x_i) - 4, \quad \text{minY} = \min(y_i) - 4$$

$$\text{maxX} = \max(x_i) + 4, \quad \text{maxY} = \max(y_i) + 4$$

The viewBox of the generated transparent SVG element is set precisely to:

```xml
viewBox="minX minY (maxX - minX) (maxY - minY)"
```

---

## 6. Stage 4 Export & Bisection KB Optimizer

### 6.1 Direct Vector Canvas Scaling
Stage 4 renders the final export canvas by mapping the smoothed vector coordinate paths directly to the user-specified export resolution, stretching or scaling the coordinates proportionally:

$$\text{scaleX} = \frac{\text{ExportInnerWidth}}{\text{cropW}}$$

$$\text{scaleY} = \frac{\text{ExportInnerHeight}}{\text{cropH}}$$

$$X_{\text{export}} = \text{dx} + (X_{\text{vector}} - \text{minX}) \cdot \text{scaleX}$$

$$Y_{\text{export}} = \text{dy} + (Y_{\text{vector}} - \text{minY}) \cdot \text{scaleY}$$

$$LineWidth = \text{StrokeWeight} \cdot \min(\text{scaleX}, \text{scaleY})$$

This method guarantees sharp rendering at any output resolution (such as high-DPI prints) and bypasses letterboxing.

### 6.2 Bisection File Size (KB) Optimizer
If the user enables the **Limit File Size (KB)** module, the application runs a synchronous, monotonic feedback loop over a unified scaling parameter $t \in [0.0, 1.0]$.

```
Input [Min KB, Max KB] ──► Target Midpoint (M)
Initialize: low = 0.0, high = 1.0
Loop (15 Iterations):
   t = (low + high) / 2
   Map t ──► scale (0.05 to 4.0) & quality (0.15 to 0.98)
   Generate canvas blob ──► Evaluate size (S)
   If S is in [Min KB, Max KB] ──► Break (Match Found)
   If S > Max KB ──► high = t
   Else ──► low = t
Output closest matching candidate blob
```

Because pixel dimensions and JPEG quality parameters scale monotonically with $t$, this bisection search converges on the target range within 15 iterations.

---

## 7. Gemini Multimodal AI Integration (Passport Grid)

SignVec integrates client-side REST communications with the Google AI Studio developer platform to generate attire variations for professional portrait photos [✍️].

```
               [ Input API Key ]
                       │
             [ Cropped/Full Image ]
                       │
       ┌───────────────┴───────────────┐
       ▼                               ▼
[ Use Cropped? Yes ]           [ Use Cropped? No ]
Extract cropped canvas         Generate uncropped master
bytes                          image bytes
       └───────────────┬───────────────┘
                       │
         [ Base64 Image Ingestion ]
                       │
          [ Multimodal Prompt Payload ]
                       │
    [ POST: gemini-3.1-flash-image ]
                       │
           [ JSON Content Parsing ]
                       │
         [ 3x3 PNG Grid Generation ]
                       │
          [ Crop and Reload Grid ]
```

### 7.1 API Connection Details
* **Endpoint**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image:generateContent?key=${API_KEY}`
* **Method**: `POST`
* **Content-Type**: `application/json`

### 7.2 Request Payload Structure
The payload contains the Base64-encoded image and the detailed layout instructions:

```json
{
  "contents": [
    {
      "parts": [
        {
          "inlineData": {
            "mimeType": "image/png",
            "data": "<BASE64_IMAGE_BYTES>"
          }
        },
        {
          "text": "Using the provided reference image, create a 3×3 grid containing 9 different passport-style portrait photographs... <CONTINUED_DETAILED_PROMPT>"
        }
      ]
    }
  ],
  "generationConfig": {
    "responseModalities": ["IMAGE"]
  }
}
```

### 7.3 Response Parsing & Crop-Reload Loop
On a successful request, the returned JSON structure is parsed as follows:

```javascript
const base64Data = jsonResponse.candidates[0].content.parts[0].inlineData.data;
```

The returned Base64 PNG image (representing the 3×3 grid of 9 passport-style photos) is loaded into an HTML `Image` object. When the user clicks **Crop a Photo from Grid**:
1. The new 3×3 grid image is set as `state.originalImage`.
2. The custom rotation values, manual erasures, and size parameters are reset.
3. Stage 1 (Crop) is reloaded, allowing the user to select and frame any one of the 9 generated portraits for compliant export.

---

## 8. Developer Maintenance & Modularity Guide

When modifying this application, adhere to the following architectural guidelines to maintain code safety:

* **Use the Safe Listener Wrapper**: Avoid registering event listeners directly on globally queried elements. Use the `listen(id, eventName, handler)` helper function. This ensures that if an HTML element is edited or removed, the missing ID will trigger a console warning rather than crashing the script.
* **Keep Preview Tasks Lightweight**: Do not run computationally heavy pixel-manipulation operations (like morphological filters) on the high-resolution `state.croppedCanvas`. Always perform these operations on `state.previewCanvas` (capped at `800px`) to keep the UI responsive.
* **Synchronous Vector Calculations**: Keep the Stage 3 vector rendering calculations synchronous. Doing so ensures that `state.processedContours` and `state.vectorBounds` are immediately populated before Step 4 initiates its drawing routines, preventing render mismatches.