---
date: 2020-03-09
title: An introduction to temporal antialiasing
tags: ["Computer Graphics", "Notes", "Real-Time Rendering"]
--- 
{{< figure library="true" src="TAAUncharted4.jpg" title="From _Temporal Antialiasing in Uncharted 4_ in SIGGRAPH 2016" lightbox="true" >}}

Recently found this preprint EGSR 2020 paper [_A Survey of Temporal Antialiasing Techniques_](http://behindthepixels.io/assets/files/TemporalAA.pdf) by Lei Yang, Shiqiu Liu, Marco Salvi online. I'm somewhat interested in the TAA technique and worked on it in my Real time ray tracing project before. This post will serve as my summary to the paper. Without extra caption, all the pictures in this post are credited to the orignal paper.  

### What is TAA? 
Temporal Antialiasing is also defined as temporally-amortized supersampling, is a widely used antialiasing technique in today's game engine. Aliasing has been converning the game developers for a long time, TAA is the technique to do the [antialiasing](https://en.wikipedia.org/wiki/Anti-aliasing) by **using data gathered across multiple frames to do the spatial antialiasing**. It has two main components: sample accumulation, and history validation.  

TAA aims to resolve subpixel details that is missing in single sampled shading. By reprojecting shading result from previous frames, TAA amortizes the cost shading multiple samples per pixel over consecutive frames. Usually implemented as a single post-processing pass.  

### 1. Algorithm overview

{{< figure library="true" src="figure2.png" title="Figure 2: Conceptual illustration of how TAA amortizes spatial supersampling over multiple frames" lightbox="true" >}}

The key of the algorithm is to reuse the pixel information from previous frames. Assuming a number of samples (<span style="color:yellow">**yellow dot**</span>) were gathered for each pixel prior to frame N, and have been averaged and stored in the history buffer as a single color value per pixel (<span style="color:green">green dot</span>). 

For each pixel in frame N, map its center location (<span style="color:orange">orange dot</span>) to the previous frame N-1. Resample history buffer at that location to obtain the history color for that pixel.
For frame N, shade a new sample (<span style="color:blue">blue dot</span>) at a jittered location, blend the result with the resampled history color. This produce the output pixel color of frame N, also becomes the history of frame N+1.

{{< figure library="true" src="figure3.png" title="Figure 3: Schematic diagram of a typical TAA implementation. Blue blocks are TAA components, green blocks are stages of the render engine." lightbox="true" >}}

### 2. Accumulating temporal samples
Spatial antialiasing requires convolving the continuous shading signal with a low-pass pixel filter to suppress excessive high-frequency components before sampling the signal. In practice, people supersample the continuous signal before applying the low-pass filter on the discrete samples of the signal. TAA amortizes the cost of supersampling by generating, aligning, and accumulating these spatial samples over multiple frames.

### 3. Data reprojection between multiple frames
With scene motion between frames, each pixel needs to compute its corresponding location in the previous frame to fetch history data. During scene rasterization, geometry is transformed twice, once using previous frames' data and once using current frames'data. The offsets between the current and previous location of every pixel is stored into a **motion vector** texture, which later used by the TAA algorithm.

In order to save framebuffer, some engine only compute and store motion vectors for animated object. For pixels are not a part of it, their location in the previous frame is :  
Eq.(1):$$p_{n-1} = M_{n-1} M_{n}^{-1}p_{n}$$
where $p_{n} = (\frac{2x}{w}-1, \frac{2y}{h}-1, z, 1)$ is 3D [clip-space](https://en.wikipedia.org/wiki/Clip_coordinates) location of the current frame pixel in [homogeneous coordinates](https://en.wikipedia.org/wiki/Homogeneous_coordinates), and $M_{n-1}$ and $M_{n}$ are previous and current frame's view-projection matrices. The resulting position is obtained after a perspective division.  

Eq.(2):
$$f_{n}(p) = \alpha s_{n}(p) + (1- \alpha) f_{n-1}(\pi(p))$$
where $f_{n}(p)$ is frame n's color output at pixel p, $\alpha$ is the blending factor, $s_{n}(p)$ is frame n's new sample color at pixel p, and $f_{n-1}(\pi(p))$ is the reprojected ouput (history color) from previous frame, using the reprojecting operator $\pi$ and resampling.

### 4. Validate history data
However, the history frame data are not always reliable and trsutworty, due to the motion of the scene, change of lighting, etc. We need a way to validate them.

#### 4.1 History rejection 
This is done be setting $\alpha$ in eq.2 to 1. Two source to determine the confidence of the history: geometry data (depth, normal, object ID and motion vector), and color data. Geometry data is typically used to identify invalid history data reprojected from mismatching surfaces due to occlusion changes. 
{{< figure library="true" src="figure5.png" title="Figure 5: Checking for disocclusion changes. Pixel coordinate $p_{1}$ in frame t is reprjected to coordinats $\pi_{t-1}(p_{1})$ in frame t-1, where it is occluded by another piece of geometry. By checking geometry data (depth, surface normal, or object ID), the occlusion can be identified and properly handled." lightbox="true" >}}

#### 4.2 History rectification 
Simpley rejecting stale or invlaid history data leads to increased temporal artifacts. Rectified history blend with current frame, leading to more visually acceptable results.

{{< figure library="true" src="figure6.png" title="Figure6: Common history rectification techniques in 2D chromaticity space for simplicity" lightbox="true" >}}

Because the current frame samples are often sparse and aliased, pull information from 3x3 neighbourhood of each pixel. The [convex hull](https://en.wikipedia.org/wiki/Convex_hull) of the neighborhood samples in color space represent the extent of the colors we expect around the center pixel. If the history color falls inside the convex hull, it is assumed to be consistent with the current frame data and can be reused safely. If it falls outside the convex hull, we need to rectify it to make it consistent with the current frame data.  

We do so by connecting history color with the current-frame sample color, and clip the line segment
against the convex hull. The intersection point is the best estimate
of history that is consistent with the current frame samples.  

In practice, computing a convex hull and the ray-hull intersection per pixel can be prohibitively expensive. A common approximation to this is to compute an axis-aligned bounding box (AABB)
of the colors using a min/max filter, and either clip or clamp the
history color against the AABB (Figure 6(c)).


### 5. Conclusion 
This post contains a brief introduction for the TAA, summarized from the paper _A Survey of Temporal Antialiasing Techniques_. The past decade has witnessed great advances and success of temporal antialiasing and upsampling techniques in the gaming industry.In particular, this post discuss how spatial samples are gathered and accumulated temporally, how history data are validated.  

Robustness and reliability of history rectification needs to be further improved to fully utilize the temporal data available. In the near future, we expect advances in both analytical and machine learningbased solutions to surpass today’s best techniques in this area, and bring the image quality of TAA to the next level.
