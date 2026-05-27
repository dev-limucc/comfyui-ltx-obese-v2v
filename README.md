# LTX 2.3 — Obese Body V3

ComfyUI workflow that transforms people in videos to appear obese/overweight using the **bodypositivity LoRA** on LTX Video 2.3.

**One universal workflow — works on any input aspect ratio (9:16, 16:9, square, etc.). Output ratio always matches input.**

---

## Pipeline

Three-pass V2V with RealESRGAN upscale (based on Discord community method — separate motion from appearance):

```
Source video (any ratio)
    ↓  VAEEncodeTiled + LTXVAudioVAEEncode
    ↓  LTXVConcatAVLatent
    ↓
[Pass 1 — Motion Retention]   sigmas: 0.3, 0.2, 0.1, 0.0
  Very low noise — locks structure.
  Face, hair, background untouched.
    ↓
[Pass 2 — Fat / Style]        sigmas: 0.7, 0.62, 0.54, 0.45, 0.35, 0.0
  Bodypositivity LoRA reshapes body, face, neck.
  Works on the clean Pass 1 latent.
    ↓
[Pass 3 — Refinement]         sigmas: 0.2, 0.1, 0.0
  Light cleanup — reduces artifacts, restores fine detail.
    ↓
  LTXVTiledVAEDecode (1×1, no seam)
    ↓
[RealESRGAN x2plus]
  Neural upscale — sharp eyes, mouth, beard, wrinkles.
    ↓
  ImageScaleToMaxDimension(1920px) — preserves input ratio
    ↓
  Two outputs saved automatically:
    1. Fat video only       → ComfyUI/output/LTX-2.3/Obese-V3/
    2. Original | Fat (SxS) → ComfyUI/output/LTX-2.3/Compare-V3/
```

**Audio** is processed through `LTXVAudioVAEEncode → LTXVConcatAVLatent` for audio-aware generation, but the **original untouched audio** is routed directly to `CreateVideo` for perfect lip sync.

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

### Upscale Model (for RealESRGAN)
| File | Destination |
|------|-------------|
| `RealESRGAN_x2plus.pth` | `models/upscale_models/` |

```bash
wget https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.1/RealESRGAN_x2plus.pth -P models/upscale_models/
```

---

## Required Custom Nodes

| Node Pack | Required For |
|-----------|-------------|
| [ComfyUI-VideoHelperSuite](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | `VHS_LoadVideo`, `CreateVideo`, `SaveVideo` |
| [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) | All LTX-specific nodes |
| [ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) | `ImageConcatMulti` (side-by-side comparison) |
| Built-in ComfyUI core | `ImageUpscaleWithModel`, `UpscaleModelLoader` |

---

## How to Use

1. **Load the workflow** — drag `LTX 2.3 - Obese Body V3.json` into ComfyUI
2. **Select your video** — click `Load Video (+ Trim)`, pick your file
3. **Queue** — hit Run

Each run produces two output videos automatically — fat-only and side-by-side comparison.

**Test first:** Set `frame_load_cap=120` (5 seconds) before running a full video.

---

## Strength Controls

### Fat Strength — Pass 2 ManualSigmas

| Level | Sigmas | Notes |
|-------|--------|-------|
| Subtle | `0.5, 0.4, 0.3, 0.0` | Slight weight gain |
| Moderate | `0.65, 0.55, 0.45, 0.35, 0.0` | Clearly heavier |
| **Default** | `0.7, 0.62, 0.54, 0.45, 0.35, 0.0` | Clearly obese |
| Strong | `0.82, 0.7, 0.6, 0.5, 0.35, 0.0` | Very fat |
| Extreme | `0.95, 0.82, 0.7, 0.6, 0.5, 0.0` | Maximum, some face drift |

### Motion Lock — Pass 1 ManualSigmas
Keep below 0.4. Default `0.3, 0.2, 0.1, 0.0`.

### LoRA Strength
Default: **1.35**. Range 0.8–1.5. Raise this before raising sigmas.

---

## Trim Controls

| Parameter | Default | Description |
|-----------|---------|-------------|
| `frame_load_cap` | `0` | Frames to load (0 = full video) |
| `skip_first_frames` | `0` | Start frame offset |

At 24fps: 120 = 5s · 240 = 10s · 480 = 20s

---

## Expected Performance (RTX 5060 Ti 16GB)

Processing at 960px. Output at 1920px longest side (RealESRGAN x2).

| Video | Time |
|-------|------|
| 5s (120 frames) | ~8–10 min |
| 10s (240 frames) | ~14–17 min |
| 20s (480 frames) | ~20–25 min |

---

## Troubleshooting

**Not fat enough** — raise LoRA strength toward 1.4–1.5, or raise Pass 2 first sigma

**Face / hair / background changing** — lower Pass 2 first sigma, keep Pass 1 at `0.3, 0.2, 0.1, 0.0`

**Beard / wrinkles disappearing** — already handled in prompts (`same beard, same wrinkles, skin texture` in positive; `smooth skin, removed wrinkles` in negative)

**Mouth not moving** — lower Pass 2 first sigma (less noise = mouth movements survive)

**CUDA out of memory** — restart ComfyUI, then reduce `960` → `832` in the `ImageScaleToMaxDimension` node (processing res node, not the output one)

**Tile seam / blurry line** — tiled decode is set to 1×1 (no tiling) — should not occur

---

## License

Workflow JSON: MIT
bodypositivity LoRA: [TheBurgstall/bodypositivity-ltx2.3](https://huggingface.co/TheBurgstall/bodypositivity-ltx2.3)
LTX Video 2.3: [Lightricks Research License](https://huggingface.co/Lightricks/LTX-Video)
