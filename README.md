# LTX 2.3 — Obese Body V2V (9:16, 1080p)

A ComfyUI workflow that transforms people in videos to appear obese/overweight using video-to-video inference with the **bodypositivity LoRA** on LTX Video 2.3. Works on **multiple people** in the same frame simultaneously.

---

## Preview

| Input | Output |
|-------|--------|
| Original video | Same video, people appear heavier |

The transformation preserves identity (same person, same background) by starting from the original video's latents and partially denoising with the bodypositivity LoRA active.

---

## Required Models

Download all of these to your ComfyUI `models/` folders:

### Checkpoint
| File | Destination | Download |
|------|-------------|----------|
| `ltx-2.3-22b-dev-fp8.safetensors` | `models/checkpoints/` | [Lightricks/LTX-Video-2B-0.9.7-distilled](https://huggingface.co/Lightricks/LTX-Video-2B-0.9.7-distilled) |

```bash
huggingface-cli download Lightricks/LTX-Video-2B-0.9.7 --include "ltx-2.3-22b-dev-fp8.safetensors" --local-dir models/checkpoints
```

### LoRAs
| File | Destination | Download |
|------|-------------|----------|
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` | `models/loras/` | [Lightricks/LTX-Video-2.3-Distilled](https://huggingface.co/Lightricks/LTX-Video-Distilled-Lora) |
| `bodypositivity-ltx-2.3-rank32-step02750.safetensors` | `models/loras/` | [TheBurgstall/bodypositivity-ltx](https://huggingface.co/TheBurgstall/bodypositivity-ltx2.3) |

```bash
huggingface-cli download Lightricks/LTX-Video-Distilled-Lora --include "ltx-2.3-22b-distilled-lora-384-1.1.safetensors" --local-dir models/loras
huggingface-cli download TheBurgstall/bodypositivity-ltx2.3 --include "*.safetensors" --local-dir models/loras
```

### Text Encoder (Gemma)
| File | Destination | Download |
|------|-------------|----------|
| `gemma_3_12B_it_fp4_mixed.safetensors` | `models/text_encoders/` | [Lightricks/LTX-Video](https://huggingface.co/Lightricks/LTX-Video) |

```bash
huggingface-cli download Lightricks/LTX-Video --include "gemma_3_12B_it_fp4_mixed.safetensors" --local-dir models/text_encoders
```

---

## Required Custom Nodes

Install via ComfyUI Manager or manually:

| Node Pack | Required For |
|-----------|-------------|
| [ComfyUI-VideoHelperSuite (VHS)](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | `VHS_LoadVideo` — trim controls |
| [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) | `LTXAVTextEncoderLoader`, `LTXVConditioning`, `LTXVTiledVAEDecode` |
| Built-in ComfyUI core | All other nodes |

---

## How to Use

1. **Load the workflow** — drag `LTX 2.3 - Obese Body V2V (9-16 1080p).json` into ComfyUI
2. **Set your video** — click the `Load Video (+ Trim)` node and select your video file
3. **Set audio source** — set the same video file in the `Load Video (Audio)` node
4. **Queue prompt** — click Run. Output saves to `ComfyUI/output/LTX-2.3/Obese-V2V/`

---

## Trim Controls

The `VHS_LoadVideo` node (top-left) has two trim parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `skip_first_frames` | `0` | Start frame (0 = beginning of video) |
| `frame_load_cap` | `0` | How many frames to load (0 = entire video) |

**Frame to time conversion at 24fps:**
- 120 frames = 5 seconds
- 240 frames = 10 seconds
- 480 frames = 20 seconds

**For long videos** (manual chunking):
1. Set `frame_load_cap = 240` (10 second chunks)
2. Run: `skip_first_frames = 0` → chunk 1
3. Run: `skip_first_frames = 240` → chunk 2
4. Run: `skip_first_frames = 480` → chunk 3
5. Stitch chunks in any video editor (DaVinci Resolve, CapCut, etc.)

---

## Obese Strength Control

Adjust the `ManualSigmas` node to control how extreme the body transformation is:

| Level | Sigma Values | Description |
|-------|-------------|-------------|
| Subtle | `0.421875, 0.0` | Slight weight increase, maximum identity preservation |
| **Moderate** (default) | `0.725, 0.421875, 0.0` | Clear obese transformation, good identity |
| Strong | `0.909375, 0.725, 0.421875, 0.0` | Heavy transformation, some identity drift |
| Extreme | `1.0, 0.909375, 0.725, 0.421875, 0.0` | Maximum effect, more identity drift |

> **Rule:** Higher starting sigma = more body transformation but more deviation from original. Start with default, go stronger if needed.

---

## Workflow Architecture

```
VHS_LoadVideo (trim) ──────────────────────────────────────────────────────┐
                                                                           ↓
LoadVideo (audio) → GetVideoComponents → fps                      ScaleMax (768px)
                                      → audio ──────────────────→    ↓
                                      → fps ──→ LTXVConditioning  ResizeMult(×32)
                                                                       ↓
                                                                  VAEEncodeTiled
                                                                       ↓ (latents)
LowVRAMCheckpointLoader                                         SamplerCustomAdvanced ←─────┐
    → LoraLoaderModelOnly (distilled, 0.5)                             ↑                    │
    → LoraLoaderModelOnly (bodypositivity, 1.0)                        │                    │
    → CFGGuider ──────────────────────────────────────────────── guider│                    │
                                                                        │                    │
LTXAVTextEncoderLoader (Gemma)                              RandomNoise─┤                    │
    → CLIPTextEncode (positive: "bodypositivity, obese...")  KSamplerSelect (euler_anc_cfg_pp)│
    → CLIPTextEncode (negative)                              ManualSigmas (0.725, 0.421875, 0)┘
    → LTXVConditioning ──────→ CFGGuider
                                                                       ↓
                                                              LTXVTiledVAEDecode
                                                                       ↓
                                                              ImageScale (1080×1920)
                                                                       ↓
                                                              CreateVideo (+ audio + fps)
                                                                       ↓
                                                              SaveVideo → output/
```

---

## Optimization Tips

### Speed (RTX 5060 Ti 16GB or similar)

| Optimization | Impact | Setting |
|-------------|--------|---------|
| **LowVRAMCheckpointLoader** | Prevents OOM with 27GB model on 16GB VRAM | Already used in workflow |
| **SamplerCustomAdvanced** | 3-4x faster than LTXVLoopingSampler | Already used |
| **3-step sigmas** | ~3 model passes total | Default: `0.725, 0.421875, 0.0` |
| **768px processing** | Avoids VRAM spike at high res | Already set |
| **1080p upscale at output** | Upscale after decode, not during | Already set (ImageScale node) |
| **euler_ancestral_cfg_pp** | Best quality-per-step ratio | Already set |
| **FP8 checkpoint** | Half the VRAM of FP16 | ltx-2.3-22b-dev-fp8 |

### Expected performance on RTX 5060 Ti (16GB)
- 5-second clip (120 frames, 768px) → ~90-150 seconds (vs ~900s before optimization)
- 10-second clip (240 frames) → ~180-300 seconds

### If still slow
- Reduce to 2-step sigmas: `0.421875, 0.0`
- Reduce frame count: set `frame_load_cap = 120` (5s chunks)
- Ensure CUDA is used: check Task Manager GPU utilization during generation

---

## Troubleshooting

**Different person in output / identity drift**
- Lower the starting sigma (e.g., `0.421875, 0.0` instead of `0.725, ...`)
- bodypositivity LoRA at 1.0 is intentional for strong effect — reduce to 0.7-0.8 if too extreme

**OOM / out of memory error**
- The workflow uses `LowVRAMCheckpointLoader` — ensure you're not accidentally using `CheckpointLoaderSimple`
- Reduce `frame_load_cap` to process shorter chunks (120 = 5s)

**Audio not in output**
- Make sure the same video file is loaded in BOTH `VHS_LoadVideo` AND `Load Video (Audio)` nodes
- `GetVideoComponents` extracts fps and audio track

**Black or corrupt output frames**
- Usually caused by sigma values starting too high (>1.0). Stay within the presets in the Obese Strength table.

---

## Parameters Reference

| Node | Key Parameter | Default | Notes |
|------|--------------|---------|-------|
| VHS_LoadVideo | skip_first_frames | 0 | Start frame |
| VHS_LoadVideo | frame_load_cap | 0 | 0 = all frames |
| VHS_LoadVideo | force_rate | 24 | Output FPS |
| LoraLoaderModelOnly (distilled) | strength | 0.5 | Enables 3-9 step inference |
| LoraLoaderModelOnly (bodypositivity) | strength | 1.0 | Obese effect intensity |
| ManualSigmas | sigmas | `0.725, 0.421875, 0.0` | Transformation strength |
| CFGGuider | cfg | 1 | Keep at 1 for distilled |
| ImageScaleToMaxDimension | max_dimension | 768 | Processing resolution cap |
| ImageScale | width/height | 1080/1920 | Final output resolution |
| LTXVTiledVAEDecode | tile_sample_min_width/height | 2×2 tiles, overlap 6 | VRAM-efficient decode |

---

## License

Workflow JSON: MIT  
bodypositivity LoRA: [TheBurgstall/bodypositivity-ltx2.3](https://huggingface.co/TheBurgstall/bodypositivity-ltx2.3) — check HuggingFace for license  
LTX Video 2.3 model: [Lightricks Research License](https://huggingface.co/Lightricks/LTX-Video)
