# LTX 2.3 — Obese Body V2V

ComfyUI workflows that transform people in videos to appear obese/overweight using the **bodypositivity LoRA** on LTX Video 2.3.

Two versions, two aspect ratios — pick what fits your video:

| Workflow | Output | Use For | Version |
|----------|--------|---------|---------|
| `LTX 2.3 - Obese Body V2V (9-16 1080p).json` | 1080×1920 portrait | TikTok, Reels, Shorts | V2 |
| `LTX 2.3 - Obese Body V2V (16-9 1080p).json` | 1920×1080 landscape | YouTube, streams, clips | V2 |
| `LTX 2.3 - Obese Body V3 (9-16 1080p).json` | 1080×1920 portrait | TikTok, Reels, Shorts | V3 |
| `LTX 2.3 - Obese Body V3 (16-9 1080p).json` | 1920×1080 landscape | YouTube, streams, clips | V3 |

---

## V2 vs V3 — Which to Use?

### V2 — Two-Pass (fast, solid)
```
Encode → Pass 1 (motion lock) → Pass 2 (fat style) → VAEDecode → lanczos upscale → 1080p
```
- ~10–15 min for a 20s clip on RTX 5060 Ti
- Good quality, stable on 16GB VRAM
- Best for quick runs and testing settings

### V3 — Three-Pass + RealESRGAN (slower, sharper)
```
Encode → Pass 1 (motion lock) → Pass 2 (fat style) → Pass 3 (refinement) → VAEDecode → RealESRGAN x2 → 1080p
```
- ~20–25 min for a 20s clip on RTX 5060 Ti
- Pass 3 cleans up artifacts left by the fat transformation
- RealESRGAN x2plus replaces lanczos — eyes, mouth, hair visibly sharper
- Best for final output you actually want to share

**Start with V2 to get your settings right, then run V3 for the final version.**

---

## How It Works

Both versions use a multi-pass video-to-video pipeline based on the community method: **separate motion from appearance**.

The problem with single-pass V2V: you can't have both high denoising (needed for fat transformation) and low denoising (needed to preserve face/hair/background) at the same time. Multi-pass solves this by splitting those jobs across separate sampler runs.

### V2 Pipeline
```
Source video
    ↓  VAEEncodeTiled + LTXVAudioVAEEncode
    ↓  LTXVConcatAVLatent  (video + audio joined into one latent)
    ↓
[Pass 1 — Motion Retention]   sigmas: 0.3, 0.2, 0.1, 0.0
  Very low noise — locks in structure.
  Face, hair, background essentially untouched.
    ↓
[Pass 2 — Fat / Style]        sigmas: 0.7, 0.62, 0.54, 0.45, 0.35, 0.0
  Bodypositivity LoRA reshapes the body.
  Works on the already-clean Pass 1 latent → more effective than single pass.
    ↓
  LTXVSeparateAVLatent
  Original audio → CreateVideo (bypasses audio VAE decode for perfect lip sync)
    ↓
  LTXVTiledVAEDecode → ImageScale (lanczos) → 1080p → SaveVideo
```

### V3 Pipeline
```
... same as V2 through Pass 2 ...
    ↓
[Pass 3 — Refinement]         sigmas: 0.2, 0.1, 0.0
  Very light cleanup pass — tightens details, reduces artifacts
  from the fat transformation without undoing the effect.
    ↓
  LTXVTiledVAEDecode
    ↓
[RealESRGAN x2plus]
  Neural upscaler — reconstructs eye, mouth, hair detail.
  Much sharper than lanczos at the same resolution.
    ↓
  ImageScale → exact 1080p crop → SaveVideo
```

**Audio** goes through `LTXVAudioVAEEncode → LTXVConcatAVLatent` so the model has audio context during generation — but the **original untouched audio** is routed directly to `CreateVideo` for perfect lip sync.

**Tile decode orientation** prevents seam artifacts through faces:
- Portrait workflows: 1 horizontal × 2 vertical tiles (no vertical seam)
- Landscape workflows: 2 horizontal × 1 vertical tiles (no horizontal seam)

---

## Required Models

### Checkpoint
| File | Destination |
|------|-------------|
| `ltx-2.3-22b-dev-fp8.safetensors` | `models/checkpoints/` |

```bash
huggingface-cli download Lightricks/LTX-Video --include "ltx-2.3-22b-dev-fp8.safetensors" --local-dir models/checkpoints
```

### LoRAs
| File | Destination |
|------|-------------|
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` | `models/loras/` |
| `bodypositivity-ltx-2.3-rank32-step02750.safetensors` | `models/loras/` |

```bash
huggingface-cli download Lightricks/LTX-Video-Distilled-Lora --include "ltx-2.3-22b-distilled-lora-384-1.1.safetensors" --local-dir models/loras
huggingface-cli download TheBurgstall/bodypositivity-ltx2.3 --include "*.safetensors" --local-dir models/loras
```

### Text Encoder
| File | Destination |
|------|-------------|
| `gemma_3_12B_it_fp4_mixed.safetensors` | `models/text_encoders/` |

```bash
huggingface-cli download Lightricks/LTX-Video --include "gemma_3_12B_it_fp4_mixed.safetensors" --local-dir models/text_encoders
```

### V3 Only — Upscale Model
| File | Destination |
|------|-------------|
| `RealESRGAN_x2plus.pth` | `models/upscale_models/` |

```bash
wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.1/RealESRGAN_x2plus.pth -P models/upscale_models/
```

---

## Required Custom Nodes

| Node Pack | Required For | V2 | V3 |
|-----------|-------------|----|----|
| [ComfyUI-VideoHelperSuite](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | `VHS_LoadVideo`, `CreateVideo`, `SaveVideo` | ✅ | ✅ |
| [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) | All LTX-specific nodes | ✅ | ✅ |
| Built-in ComfyUI core | `ImageUpscaleWithModel`, `UpscaleModelLoader` | — | ✅ |

---

## How to Use

1. **Load the workflow** — drag the `.json` into ComfyUI or use Load → browse
2. **Select your video** — click `Load Video (+ Trim)`, pick your file
3. **Queue** — hit Run

Output saves to:
- V2: `ComfyUI/output/LTX-2.3/Obese-V2V/`
- V3: `ComfyUI/output/LTX-2.3/Obese-V3/`

Use **9-16** workflows for portrait/vertical video, **16-9** for landscape/horizontal.

---

## Strength Controls

### Fat Strength — Pass 2 ManualSigmas

Higher starting sigma = fatter body but more risk of face drift. Lower = more preserved but less fat.

| Level | Sigmas | Notes |
|-------|--------|-------|
| Subtle | `0.5, 0.4, 0.3, 0.0` | Slight weight gain, maximum identity preservation |
| Moderate | `0.65, 0.55, 0.45, 0.35, 0.0` | Clearly heavier |
| **Default** | `0.7, 0.62, 0.54, 0.45, 0.35, 0.0` | Clearly obese, good preservation |
| Strong | `0.82, 0.7, 0.6, 0.5, 0.35, 0.0` | Very fat, slight face drift |
| Extreme | `0.95, 0.82, 0.7, 0.6, 0.5, 0.0` | Maximum effect, noticeable drift |

### Motion Lock — Pass 1 ManualSigmas

Keep this low. Going above 0.4 starts defeating the purpose.

| Setting | Sigmas |
|---------|--------|
| **Default** | `0.3, 0.2, 0.1, 0.0` |
| Slightly firmer | `0.4, 0.3, 0.2, 0.0` |

### LoRA Strength
Default: **1.3**. Range: 0.8–1.5. Raise this before raising sigmas if not fat enough.

---

## Trim Controls

| Parameter | Default | Description |
|-----------|---------|-------------|
| `frame_load_cap` | `0` | Frames to load (0 = full video) |
| `skip_first_frames` | `0` | Start frame offset |

At 24fps: 120 = 5s · 240 = 10s · 480 = 20s

**Always test with `frame_load_cap=120` (5s) first** before running the full video.

---

## Expected Performance (RTX 5060 Ti 16GB)

Processing at 960px (longest dimension). Both versions respect the 16GB VRAM limit.

| Video | V2 time | V3 time |
|-------|---------|---------|
| 5s (120 frames) | ~5–7 min | ~8–10 min |
| 10s (240 frames) | ~10–12 min | ~14–17 min |
| 20s (480 frames) | ~15–20 min | ~20–25 min |

**If you get OOM:** lower `960` → `832` in the `ImageScaleToMaxDimension` node.

---

## Troubleshooting

**Not fat enough**
- Raise LoRA strength toward 1.4–1.5 first
- Then raise Pass 2 first sigma (e.g. `0.82` start)

**Face / hair / background changing**
- Lower Pass 2 first sigma (e.g. `0.65` start)
- Keep Pass 1 at `0.3, 0.2, 0.1, 0.0`

**Mouth not moving / lip sync off**
- Already fixed: original audio bypasses VAE decode
- If mouth looks visually frozen: lower Pass 2 first sigma (less noise = mouth movements survive)

**CUDA out of memory**
- Restart ComfyUI fully to clear VRAM after a failed run
- Reduce `ImageScaleToMaxDimension` from 960 → 832
- Reduce `frame_load_cap` to process shorter chunks

**Tile seam artifacts (blurry line across frame)**
- Portrait: should be 1h×2v in `LTXVTiledVAEDecode` — check it hasn't been changed
- Landscape: should be 2h×1v

---

## License

Workflow JSON: MIT
bodypositivity LoRA: [TheBurgstall/bodypositivity-ltx2.3](https://huggingface.co/TheBurgstall/bodypositivity-ltx2.3)
LTX Video 2.3: [Lightricks Research License](https://huggingface.co/Lightricks/LTX-Video)
