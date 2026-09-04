### 1. WebODM (Baseline Photogrammetry)
*   **Source:** https://webodm.org/
*   **Core Concept:** Open-source photogrammetry software that stitches 2D drone images into 3D models using dense matching.
*   **Relevance / Critical Weakness:** Requires a grid flight pattern with 80% image overlap. It completely fails to reconstruct occluded surfaces (like the sides of buildings) in a single-pass drone flight. 
*   **Hardware Constraint Check:** Takes hours of CPU/RAM processing time. Fails the "near real-time" problem statement requirement.
*   **PPT Slide Mapping:** Slide 2: The Status Quo (Why traditional methods fail).

### 2. Structure-From-Motion (COLMAP)
*   **Source:** https://www.cv-foundation.org/openaccess/content_cvpr_2016/papers/Schonberger_Structure-From-Motion_Revisited_CVPR_2016_paper.pdf
*   **Core Concept:** Extracts camera positions and a sparse 3D point cloud from unordered 2D images.
*   **Relevance to Project:** We will use COLMAP purely to track where the drone camera is in 3D space (camera poses) to anchor our AI rendering.
*   **Hardware Constraint Check:** We must restrict COLMAP to *sparse* reconstruction only; running its dense matching module will bottleneck the HP Victus 15.
*   **PPT Slide Mapping:** Slide 4: Architecture (Camera Pose Estimation).

### 3. Depth Anything V2 (Monocular Metric Depth Estimation)
*   **Source:** https://github.com/DepthAnything/Depth-Anything-V2
*   **Core Concept:** An AI vision model trained on 595K synthetic labeled images and 62M+ real unlabeled images that predicts the absolute depth of pixels from a single RGB frame.
*   **Relevance to Project:** Solves the single-pass blind spot issue. It guesses the geometry of unseen building facades and terrain, replacing the need for multiple drone angles. 
*   **Hardware Constraint Check:** Comes in scales from 25M to 1.3B parameters. We must use the smallest, most lightweight version and quantize it to run smoothly on the laptop GPU.
*   **PPT Slide Mapping:** Slide 4: Architecture & Slide 5: Key Challenges (Solves Challenge i: Limited viewing angles).

### 4. SuGaR (Surface-Aligned Gaussian Splatting)
*   **Source:** https://github.com/Anttwo/SuGaR 
*   **Core Concept:** Optimizes 3D Gaussian Splatting by aligning Gaussians with the scene's surface, allowing for precise and extremely fast 3D mesh extraction using Poisson reconstruction.
*   **Relevance to Project:** Takes the sparse points from COLMAP and depth maps from Depth Anything to render the scene. Unlike standard 3DGS, SuGaR actually lets us export an editable 3D mesh within minutes, satisfying the PS requirement for "measurement and analysis purposes". 
*   **Hardware Constraint Check:** VRAM intensive. We must cap Gaussian limits or downscale the drone video to 1080p before feeding it into the model.
*   **PPT Slide Mapping:** Slide 3: Proposed Solution & Slide 4: Architecture.

### 5. [OpenCV Blur Filter & YOLOv8 Object Masking]
*   **Source:** [Find a tutorial or OpenCV documentation link]
*   **Core Concept:** A Python preprocessing script. OpenCV calculates the variance of the Laplacian to drop blurred frames. YOLOv8 detects and masks moving objects (vehicles/humans).
*   **Relevance to Project:** Directly solves Key Challenges (ii) Motion blur and (iv) Dynamic objects. It prevents garbage data from corrupting the Gaussian Splatting engine.
*   **Hardware Constraint Check:** YOLOv8 Nano (YOLOv8n) must be used to ensure the masking process runs fast on the laptop GPU before the heavier 3D pipelines start.
*   **PPT Slide Mapping:** Slide 4: Architecture (Preprocessing Node) and Slide 5: Key Challenges.

### 6. EXIF Telemetry Fusion with COLMAP
*   **Source:** [Find documentation on COLMAP model alignment or ExifTool]
*   **Core Concept:** Extracting embedded GPS/altitude data from the drone video metadata and using it to scale and orient the COLMAP sparse point cloud to absolute Earth coordinates.
*   **Relevance to Project:** Solves Key Challenge (v) GPS inaccuracies and (viii) Maintaining metric accuracy without GCPs. It ensures a 10-meter building in reality measures exactly 10 meters in the web viewer.
*   **Hardware Constraint Check:** Purely mathematical matrix transformations in NumPy; extremely low compute cost.
*   **PPT Slide Mapping:** Slide 4: Architecture (Telemetry Sync) and Slide 5: Key Challenges.


### 7. Three.js 3D Gaussian Splat Web Viewer
*   **Source:** https://github.com/mkkellogg/GaussianSplats3D
*   **Core Concept:** An open-source JavaScript (WebGL/WebGPU) library that renders millions of 3D Gaussian splats natively inside a web browser.
*   **Relevance to Project:** The problem statement demands the model be suitable for "visualization, measurement, and analysis." This is our frontend. We compress the splat data into a highly optimized format (like `.ksplat`) and host it via Vercel so the judges can interact with the 3D map using just a URL.
*   **Hardware Constraint Check:** The rendering load is shifted to the browser. By converting standard `.ply` files to `.ksplat`, we ensure the web page loads fast and doesn't crash the end-user's device.
*   **PPT Slide Mapping:** Slide 4: Architecture (Deployment Node) and Slide 6: Technology Stack.

### 8. Metric Scaling without GCPs (COLMAP model_aligner)
*   **Source:** https://colmap.github.io/faq.html
*   **Core Concept:** A mathematical alignment tool within COLMAP that uses the drone's embedded GPS/EXIF data to scale, rotate, and translate the arbitrary 3D point cloud into absolute real-world Earth coordinates.
*   **Relevance to Project:** Solves Key Challenge (viii): "Maintaining metric accuracy without Ground Control Points." Because we anchor the camera poses to the flight metadata before Gaussian Splatting, a 10-meter road in the drone video will measure exactly 10 meters in our 3D web viewer.
*   **Hardware Constraint Check:** Zero heavy compute required. It is simply a matrix transformation applied to the coordinates before they are passed into the AI rendering engine.
*   **PPT Slide Mapping:** Slide 5: Key Challenges.

---

## Phase 2: Key Challenges Solution Matrix (For Slide 5)

| PS Challenge | Our Solution (The "How") | Pipeline Node |
| :--- | :--- | :--- |
| **(i) Limited viewing angles** | Depth Anything v2 infers unseen geometry (like building back-walls) by guessing depth from visible edges. | AI Depth Prior Generation |
| **(ii) Motion blur / compression** | OpenCV calculates Laplacian variance to automatically drop blurry frames before they corrupt the model. | Preprocessing |
| **(iii) Variable illumination** | 3D Gaussian Splatting utilizes spherical harmonics to bake view-dependent lighting directly into the splats. | 3DGS Rendering |
| **(iv) Dynamic objects** | YOLOv8n semantic segmentation masks out moving entities (cars/humans) so the 3D model ignores them. | Preprocessing |
| **(v) GPS inaccuracies / noise** | COLMAP extracts highly accurate relative camera poses, overriding noisy micro-movements in the GPS data. | Camera Pose Estimation |
| **(vi) Near real-time processing** | We bypass slow dense-matching and NeRFs entirely, using sparse SfM and 3DGS for instant rasterization. | Global Pipeline / 3DGS |
| **(vii) Occluded surfaces** | Monocular Metric Depth Estimation mathematically projects a structural mesh for surfaces hidden from the drone. | AI Depth Prior Generation |
| **(viii) Metric accuracy w/o GCPs** | COLMAP's `model_aligner` anchors the sparse 3D point cloud strictly to the drone's EXIF GPS/altitude metadata. | Telemetry Sync |


## Things we'll be using for SIH26158 

### COLMAP 
Extracting Image Details:
Tracking the Drone's Path:
Building a Basic 3D Skeleton:
Making a Solid 3D Structure:
Feeding Modern AI Systems:

