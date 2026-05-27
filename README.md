# LTX 2.3 — Obese Body

ComfyUI workflows that transform people in videos to appear obese/overweight using the **bodypositivity LoRA** on LTX Video 2.3.

Two workflows — same output quality, different identity-preservation mechanisms:

| Workflow | Method | Best for |
|----------|--------|----------|
| **V3** | 3-pass V2V + standard LoRA | Fast, no extra download |
| **IC-LoRA** | IC-LoRA reference conditioning | Better beard/wrinkle preservation |

**Both:** any input aspect ratio → same ratio output at 1920px. Two outputs per run (fat-only + side-by-side comparison).

---

## Workflow 1 — V3 (3-Pass)

`LTX 2.3 - Obese Body V3.json`

```
Source video (any ratio)
    ↓  VAEEncodeTiled + LTXVAudioVAEEncode → LTXVConcatAVLatent
    ↓
[Pass 1 — Motion Retention]   sigmas: 0.3, 0.2, 0.1, 0.0
  Very low noise — locks structure. Face/hair/bg untouched.
    ↓
[Pass 2 — Fat / Style]        sigmas: 0.78, 0.68, 0.58, 0.48, 0.35, 0.0
  bodypositivity LoRA (rank32) reshapes body + face.
    ↓
[Pass 3 — Refinement]         sigmas: 0.2, 0.1, 0.0
  Light cleanup — reduces artifacts, restores fine detail.
    ↓
  LTXVTiledVAEDecode (1×1, no seam)
    ↓
  RealESRGAN x2plus → ImageScaleToMaxDimension(1920px)
    ↓
  fat-only video + original|fat side-by-side
```

---

## Workflow 2 — IC-LoRA

`LTX 2.3 - Obese Body IC-LoRA.json`

```
Source video (any ratio)
    ↓  VAEEncodeTiled + LTXVAudioVAEEncode → LTXVConcatAVLatent
    ↓
[LTXAddVideoICLoRAGuide]
  Injects source frames as reference conditioning.
  Model sees the original person during generation →
  stronger beard/identity preservation at higher sigma.
    ↓
[Single-pass sampler]         sigmas: 0.85, 0.75, 0.65, 0.55, 0.45, 0.35, 0.0
  Higher start sigma than V3 — IC-LoRA handles identity.
    ↓
  LTXVTiledVAEDecode (1×1, no seam)
    ↓
  RealESRGAN x2plus → ImageScaleToMaxDimension(1920px)
    ↓
  fat-only video + original|fat side-by-side
```

### IC-LoRA Extra Download

```bash
# Run from ComfyUI folder
huggingface-cli download TheBurgstall/bodypositivity-ltx2.3 \
  --include "rank128/*" \
  --local-dir models/loras/ltx-2/bodypositivity
```

File lands at: `models/loras/ltx-2/bodypositivity/rank128/lora_weights_step_01000.safetensors`

---

## Required Models (both workflows)

### Checkpoint
| File | Destination |
|------|-------------|
| `ltx-2.3-22b-dev-fp8.safetensors` | `models/checkpoints/` |

```bash
huggingface-cli download Lightricks/LTX-Video --include "ltx-2.3-22b-dev-fp8.safetensors" --local-dir models/checkpoints
```

### LoRAs
| File | Destination | Used by |
|------|-------------|---------|
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` | `models/loras/` | Both |
| `bodypositivity-ltx-2.3-rank32-step02750.safetensors` | `models/loras/` | V3 only |
| `ltx-2/bodypositivity/rank128/lora_weights_step_01000.safetensors` | `models/loras/` | IC-LoRA only |

```bash
# Distilled LoRA (both workflows)
huggingface-cli download Lightricks/LTX-Video-Distilled-Lora --include "ltx-2.3-22b-distilled-lora-384-1.1.safetensors" --local-dir models/loras

# V3 standard LoRA
huggingface-cli download TheBurgstall/bodypositivity-ltx2.3 --include "*.safetensors" --local-dir models/loras

# IC-LoRA rank128 (IC-LoRA workflow only)
huggingface-cli download TheBurgstall/bodypositivity-ltx2.3 --include "rank128/*" --local-dir models/loras/ltx-2/bodypositivity
```

### Text Encoder
| File | Destination |
|------|-------------|
| `gemma_3_12B_it_fp4_mixed.safetensors` | `models/text_encoders/` |

```bash
huggingface-cli download Lightricks/LTX-Video --include "gemma_3_12B_it_fp4_mixed.safetensors" --local-dir models/text_encoders
```

### Upscale Model
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
| [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) | All LTX nodes including `LTXICLoRALoaderModelOnly`, `LTXAddVideoICLoRAGuide` |
| [ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) | `ImageConcatMulti` (side-by-side comparison) |
| Built-in ComfyUI core | `ImageUpscaleWithModel`, `UpscaleModelLoader` |

---

## How to Use

1. **Load the workflow** — drag the `.json` into ComfyUI
2. **Select your video** — click `Load Video (+ Trim)`, pick your file
3. **Queue** — hit Run

Each run produces two output videos automatically — fat-only and side-by-side comparison.

**Test first:** Set `frame_load_cap=120` (5 seconds) before running a full video.

---

## Strength Controls

### V3 — Fat Strength (Pass 2 ManualSigmas)

| Level | Sigmas | Notes |
|-------|--------|-------|
| Subtle | `0.5, 0.4, 0.3, 0.0` | Slight weight gain |
| Moderate | `0.65, 0.55, 0.45, 0.35, 0.0` | Clearly heavier |
| **Default** | `0.78, 0.68, 0.58, 0.48, 0.35, 0.0` | Clearly obese |
| Strong | `0.82, 0.7, 0.6, 0.5, 0.35, 0.0` | Very fat |

### IC-LoRA — Fat Strength (single ManualSigmas)

| Level | Sigmas | Notes |
|-------|--------|-------|
| Subtle | `0.6, 0.5, 0.4, 0.3, 0.0` | Slight weight gain |
| **Default** | `0.85, 0.75, 0.65, 0.55, 0.45, 0.35, 0.0` | Clearly obese |
| Strong | `0.95, 0.82, 0.70, 0.57, 0.44, 0.30, 0.0` | Very fat |

IC-LoRA can use higher sigma than V3 because the reference conditioning preserves identity.

### Motion Lock — V3 Pass 1 ManualSigmas
Keep below 0.4. Default `0.3, 0.2, 0.1, 0.0`.

### LoRA Strength
- V3: bodypositivity LoRA default **1.4** (range 0.8–1.5)
- IC-LoRA: IC-LoRA loader default **1.0** (range 0.8–1.2), guide strength default **1.0**
- Distilled LoRA (both): **0.5** — don't change

---

## Expected Performance (RTX 5060 Ti 16GB)

Processing at 960px. Output at 1920px longest side (RealESRGAN x2).

| Video | V3 Time | IC-LoRA Time |
|-------|---------|--------------|
| 5s (120 frames) | ~8–10 min | ~5–7 min |
| 10s (240 frames) | ~14–17 min | ~10–13 min |
| 20s (480 frames) | ~20–25 min | ~16–20 min |

IC-LoRA is faster — single pass vs three passes.

---

## Troubleshooting

**Not fat enough** — raise sigmas, or raise LoRA strength

**Face / hair / background changing** — lower sigmas; for V3 keep Pass 1 at `0.3, 0.2, 0.1, 0.0`

**Beard / wrinkles disappearing** — try IC-LoRA workflow (stronger identity preservation); or lower Pass 2 sigma in V3

**Mouth not moving** — lower first sigma (less noise = mouth movements survive)

**CUDA out of memory** — restart ComfyUI, then reduce `960` → `832` in the `Limit to 960px` node

**IC-LoRA: lora not found error** — download the rank128 file (see download command above)

---

## License

Workflow JSON: MIT
bodypositivity LoRA: [TheBurgstall/bodypositivity-ltx2.3](https://huggingface.co/TheBurgstall/bodypositivity-ltx2.3)
LTX Video 2.3: [Lightricks Research License](https://huggingface.co/Lightricks/LTX-Video)
