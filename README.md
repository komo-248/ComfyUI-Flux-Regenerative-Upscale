# ComfyUI – Flux Regenerative Upscale Workflow

A ComfyUI workflow for AI-driven regenerative video upscaling using FLUX.1-fill-dev with ControlNet guidance, signal processing pre/post chains, and RIFE frame interpolation. Rather than simply enlarging pixels, this workflow uses a diffusion model to reconstruct and hallucinate detail guided by the original frame's structure — producing results that go beyond what a standard upscaler can recover.

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Required Models](#required-models)
- [Required Custom Nodes](#required-custom-nodes)
- [Workflow Parameters](#workflow-parameters)
  - [Input / Output](#input--output)
  - [FLUX Generation](#flux-generation)
  - [ControlNet Guidance](#controlnet-guidance)
  - [Signal Processing Chain](#signal-processing-chain)
  - [Post-Processing](#post-processing)
  - [Frame Interpolation](#frame-interpolation)
- [Signal Processing Chain Detail](#signal-processing-chain-detail)
- [Installation](#installation)
- [Usage](#usage)

---

## Overview

Standard upscalers (Real-ESRGAN, waifu2x) enlarge existing pixels using learned filters. They can sharpen and denoise but cannot add detail that was never in the source. This workflow takes a different approach:

1. **Extracts structural signals** from each frame (lineart, edge maps)
2. **Constructs a composite mask** that tells the diffusion model where detail is missing or degraded
3. **Runs FLUX.1-fill-dev inpainting** guided by ControlNet to regenerate those regions
4. **Post-processes the output** through a signal chain to restore color balance and suppress artifacts
5. **Interpolates** the result with RIFE for smooth motion output

The result is a frame that has been structurally reconstructed by a diffusion model rather than just mathematically enlarged — detail is synthesized, not interpolated.

---

## How It Works

```
Input Video (VHS_LoadVideo)
        │
        ├──────────────────────────────────────────────────────┐
        │                                                      │
        ▼                                                      ▼
[Signal Processing Chain]                            [Original Frame Preview]
  JPEG artifact removal (disable)
  → Image Filter (sharpness/contrast boost)
  → Manga2Anime LineArt Preprocessor
  → ImageInvert
  → Canny Edge (low threshold: 30/85)             
  → ImageInvert                                   
  → JPEG artifact removal (enable, strength 10)   
  → Image Filter Adjustments                      
  → Canny Edge (high threshold: 170/255)          
  → ImageInvert                                   
  → Image Filter Adjustments                      
  → ImageBlend (multiply, 0.85) ──────────────────────────────────► Composite Mask
  → ImageBlend (multiply, 0.35)                                           │
  → JPEG artifact removal (enable, strength 5)                            │
  → Image Filter Adjustments (final signal)                               │
  → Bilateral Filter (σ=3)                                                │
  → JPEG artifact removal (enable, strength 1)                            │
        │                                                                  │
        ▼                                                                  ▼
[FLUX.1-fill-dev Inpainting] ◄──── ControlNet (Union Pro, strength 0.55, end 0.75)
  Model: flux1-fill-dev                           AIO Preprocessor (LineartStandard, 1920px)
  CLIP: clip_l + t5xxl_fp16
  VAE: ae.safetensors
  Guidance: 2.0
  Differential Diffusion
  InpaintModelConditioning
        │
        ▼
[UltimateSDUpscale]
  Scale: 1.5x
  Tile: 512×512, padding 32
  Sampler: DPM++ 2M Karras
  Denoise: 0.01 (structure-preserving)
        │
        ▼
[Post-Processing]
  Image Filter Adjustments (brightness/contrast normalization)
  → Bilateral Filter (σ=1)
        │
        ├── Preview (Upscaled Frames)
        ▼
[RIFE VFI Frame Interpolation]
  Model: rife47
  Multiplier: 2x
        │
        ▼
[ImageScale → 960×720 (Lanczos)]
        │
        ▼
[VHS_VideoCombine → MP4, CRF 19, H.264]
```

---

## Required Models

Download and place in your ComfyUI `models/` subdirectories as noted:

| Model | Folder | Purpose |
|-------|--------|---------|
| `flux1-fill-dev.safetensors` | `models/unet/` | FLUX.1-fill-dev inpainting UNET |
| `clip_l.safetensors` | `models/clip/` | CLIP-L text encoder |
| `t5xxl_fp16.safetensors` | `models/clip/` | T5-XXL text encoder |
| `ae.safetensors` | `models/vae/` | FLUX VAE |
| `FLUX.1-dev-ControlNet-Union-Pro.safetensors` | `models/controlnet/` | ControlNet guidance |
| `realesr-animevideov3.pth` | `models/upscale_models/` | Real-ESRGAN upscale model |
| `rife47.pth` | `models/rife/` | RIFE frame interpolation |

**Model sources:**
- FLUX.1-fill-dev: [black-forest-labs/FLUX.1-fill-dev](https://huggingface.co/black-forest-labs/FLUX.1-fill-dev) (requires HuggingFace access)
- FLUX ControlNet Union Pro: [Shakker-Labs/FLUX.1-dev-ControlNet-Union-Pro](https://huggingface.co/Shakker-Labs/FLUX.1-dev-ControlNet-Union-Pro)
- Real-ESRGAN animevideov3: [xinntao/Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN/releases)
- RIFE 4.7: [hzwer/ECCV2022-RIFE](https://github.com/hzwer/ECCV2022-RIFE)

> FLUX.1-fill-dev requires ~24GB VRAM for full precision. Use `fp8` quantization if running on 16GB or less.

---

## Required Custom Nodes

Install via [ComfyUI Manager](https://github.com/ltdrdata/ComfyUI-Manager) or clone manually into `ComfyUI/custom_nodes/`:

| Node Pack | Nodes Used |
|-----------|-----------|
| [ComfyUI-VideoHelperSuite (VHS)](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | `VHS_LoadVideo`, `VHS_VideoCombine`, `VHS_BatchManager`, `VHS_VideoInfo` |
| [ComfyUI-Advanced-ControlNet](https://github.com/Kosinkadink/ComfyUI-Advanced-ControlNet) | `ACN_AdvancedControlNetApply` |
| [ComfyUI_UltimateSDUpscale](https://github.com/ssitu/ComfyUI_UltimateSDUpscale) | `UltimateSDUpscale` |
| [ComfyUI-RIFE-VFI](https://github.com/Fannovel16/ComfyUI-Frame-Interpolation) | `RIFE VFI` |
| [ComfyUI-Image-Filters](https://github.com/spacepxl/ComfyUI-Image-Filters) | `BilateralFilter`, `Image Filter Adjustments`, `JPEG artifacts removal FBCNN` |
| [comfyui_controlnet_aux](https://github.com/Fannovel16/comfyui_controlnet_aux) | `AIO_Preprocessor`, `CannyEdgePreprocessor`, `Manga2Anime_LineArt_Preprocessor` |
| [ComfyUI-FluxTrainer / Flux nodes](https://github.com/kijai/ComfyUI-FluxTrainer) | `FluxGuidance`, `DifferentialDiffusion` |

---

## Workflow Parameters

### Input / Output

| Node | Parameter | Value | Notes |
|------|-----------|-------|-------|
| `VHS_LoadVideo` | Video input | — | Load source video |
| `VHS_VideoCombine` | Format | `video/h264-mp4` | H.264 MP4 output |
| `VHS_VideoCombine` | CRF | `19` | Good quality/size balance |
| `VHS_VideoCombine` | Frame rate | `24` | Set to match source |
| `ImageScale` | Output resolution | 960×720 | Lanczos downscale after upscale |

---

### FLUX Generation

| Node | Parameter | Value | Notes |
|------|-----------|-------|-------|
| `UNETLoader` | Model | `flux1-fill-dev.safetensors` | Fill/inpainting variant |
| `DualCLIPLoader` | CLIP models | `clip_l` + `t5xxl_fp16` | Required dual encoders for FLUX |
| `VAELoader` | VAE | `ae.safetensors` | FLUX native VAE |
| `FluxGuidance` | Guidance scale | `2.0` | Low guidance preserves source structure |
| `CLIPTextEncode` | Positive prompt | `High-quality anime illustration, dynamic, crisp, sharp, linework, cel-shade, Vivid colors, deep shadows, Intricate details, 8k resolution, masterful composition, sharp focus` | Tune to your content style |
| `DifferentialDiffusion` | — | enabled | Applies spatially-varying noise based on mask — regenerates degraded areas more aggressively |
| `InpaintModelConditioning` | — | enabled | Conditions FLUX on the masked region for inpainting |

---

### ControlNet Guidance

| Node | Parameter | Value | Notes |
|------|-----------|-------|-------|
| `ControlNetLoader` | Model | `FLUX.1-dev-ControlNet-Union-Pro.safetensors` | Multi-mode ControlNet |
| `AIO_Preprocessor` | Type | `LineartStandardPreprocessor` | Extracts clean lineart at 1920px |
| `ACN_AdvancedControlNetApply` | Strength | `0.55` | Moderate guidance — preserves FLUX freedom to hallucinate detail |
| `ACN_AdvancedControlNetApply` | Start | `0.0` | Active from start of diffusion |
| `ACN_AdvancedControlNetApply` | End | `0.75` | Released at 75% — allows final steps to blend freely |

---

### Signal Processing Chain

See [Signal Processing Chain Detail](#signal-processing-chain-detail) below for a full breakdown of each node's role in the mask construction pipeline.

| Node | Key Settings | Role |
|------|-------------|------|
| `JPEG artifacts removal FBCNN` (Node 246) | disabled | Pass-through — baseline frame |
| `Image Filter Adjustments` (Node 255) | contrast +2, sharpness +5 | Enhance structure before lineart extraction |
| `Manga2Anime_LineArt_Preprocessor` | 512px | Extract anime-style lineart |
| `CannyEdgePreprocessor` (Node 121) | low: 30/85, 768px | Soft edge map — catches gradients |
| `CannyEdgePreprocessor` (Node 276) | high: 170/255, 512px | Hard edge map — structural outlines only |
| Multiple `ImageBlend` (multiply) | 0.35, 0.60, 0.85 | Layer signal maps into composite mask |
| `BilateralFilter` (Node 273) | σd=3, σr=0.25 | Edge-preserving smooth of final mask |
| `JPEG artifacts removal FBCNN` (Node 282) | strength 1 | Final light artifact removal before inpainting |

---

### Post-Processing

| Node | Parameter | Value | Notes |
|------|-----------|-------|-------|
| `UltimateSDUpscale` | Scale | `1.5x` | Tile-based upscale after FLUX regeneration |
| `UltimateSDUpscale` | Tile size | 512×512 | With 32px padding |
| `UltimateSDUpscale` | Sampler | `DPM++ 2M Karras` | |
| `UltimateSDUpscale` | Denoise | `0.01` | Near-zero — preserves FLUX output, just sharpens |
| `Image Filter Adjustments` (Node 271) | brightness +0.05, contrast +1.05 | Normalize FLUX output levels |
| `BilateralFilter` (Node 292) | σd=1, σr=0.25 | Light edge-preserving smoothing on final output |

---

### Frame Interpolation

| Node | Parameter | Value | Notes |
|------|-----------|-------|-------|
| `RIFE VFI` | Model | `rife47.pth` | RIFE 4.7 — best temporal consistency |
| `RIFE VFI` | Multiplier | `2x` | Doubles frame count |
| `RIFE VFI` | Scale | `2` | Internal scale factor |
| `RIFE VFI` | Fast mode | enabled | |
| `RIFE VFI` | Ensemble | enabled | Averages forward/backward passes for smoother results |

---

## Signal Processing Chain Detail

The most complex part of this workflow is the mask construction chain. Its purpose is to identify where the source frame has degraded, blurry, or missing detail — and produce a mask that tells FLUX exactly where to regenerate.

**Why so many steps?**

A single edge detector or lineart extractor is noisy — it misses some degraded areas and over-fires on others. By combining multiple signal extractors at different sensitivities and blending them with multiply operations, the composite mask is:
- High-confidence in areas where multiple detectors agree there is structural detail
- Near-zero in areas where the frame is already clean (protecting them from unnecessary regeneration)

**Chain breakdown:**

```
Raw frame
  → FBCNN disabled (pass-through baseline)
  → Contrast/sharpness boost (Node 255: contrast=2, sharpness=5)
      Purpose: Amplify degraded detail so the lineart extractor can find it
  → Manga2Anime LineArt (512px)
      Purpose: Anime-optimized lineart extraction
  → Invert
      Purpose: Black lines on white → white signal on black for multiply blending
  → Canny (soft: 30/85, 768px)
      Purpose: Wide-net edge detection — catches subtle gradients not in lineart
  → Invert
  → FBCNN enabled, strength=10 (Node 253)
      Purpose: Simulate heavy JPEG degradation signal to locate artifact-prone regions
  → Image Filter (Node 255: further processing)
  → Canny (hard: 170/255, 512px) → Invert
      Purpose: Tight structural outline only — high confidence edges
  → ImageBlend multiply 0.85 (hard canny × soft blend)
      Purpose: Weight hard edges heavily in the final mask
  → ImageBlend multiply 0.60 (FBCNN signal × edge blend)
      Purpose: Overlap artifact-prone regions with structural edges
  → FBCNN enabled, strength=5 (Node 270)
  → ImageBlend multiply 0.35 (light blend)
      Purpose: Add low-confidence regions at reduced weight
  → Image Filter Adjustments (Node 286: final signal shaping)
  → Bilateral Filter σd=3 (Node 273)
      Purpose: Smooth mask edges without crossing structural boundaries
  → FBCNN strength=1 (Node 282)
      Purpose: Final light cleanup before passing to InpaintModelConditioning
```

The output is a grayscale mask where bright regions = regenerate, dark regions = preserve.

---

## Installation

1. **Install ComfyUI** — [ComfyUI installation guide](https://github.com/comfyanonymous/ComfyUI)
2. **Install ComfyUI Manager** — [ltdrdata/ComfyUI-Manager](https://github.com/ltdrdata/ComfyUI-Manager)
3. **Install all custom nodes** listed in [Required Custom Nodes](#required-custom-nodes) via Manager
4. **Download all models** listed in [Required Models](#required-models) and place in correct subdirectories
5. **Load the workflow** — drag `Flux_Regen_Upscale.json` onto the ComfyUI canvas or use Load

---

## Usage

1. Open ComfyUI and load `Flux_Regen_Upscale.json`
2. In the `VHS_LoadVideo` node, select your input video
3. Adjust `VHS_VideoCombine` frame rate to match your source
4. Optionally edit the positive prompt in `CLIPTextEncode` to match your content style
5. Set `VHS_BatchManager` frame batch size based on available VRAM
6. Queue the prompt

> **VRAM guidance:** FLUX.1-fill-dev is large. On 24GB run batch size 1–2. On 16GB use fp8 quantization and batch size 1. The workflow is designed for batch processing — it will process your video in frame batches and combine them automatically.
