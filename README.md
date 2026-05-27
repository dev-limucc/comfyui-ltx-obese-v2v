# LTX 2.3 — Obese Body V2V

ComfyUI workflows that transform people in videos to appear obese/overweight using a **two-pass video-to-video pipeline** with the **bodypositivity LoRA** on LTX Video 2.3.

Two variants included:

| Workflow | Output | Use For |
|----------|--------|---------|
| `LTX 2.3 - Obese Body V2V (9-16 1080p).json` | 1080×1920 portrait | TikTok, Reels, Shorts |
| `LTX 2.3 - Obese Body V2V (16-9 1080p).json` | 1920×1080 landscape | YouTube, streams, clips |

---

## How It Works

Two-pass pipeline (based on Discord community method — separate motion from appearance):

```
Source video
    ↓  VAEEncodeTiled + AudioVAEEncode
    ↓  LTXVConcatAVLatent  (video + audio joined)
    ↓
[Pass 1 — Motion Retention]   sigmas: 0.3 → 0.2 → 0.1 → 0
  Very low noise — locks in motion structure.
  Face, hair, background stay intact.
    ↓
[Pass 2 — Fat / Style]        sigmas: 0.9 → 0.8 → 0.7 → 0.6 → 0.5 → 0
  Bodypositivity LoRA reshapes the body.
  Works on the clean pass-1 latent → more effective than single pass.
    ↓
  LTXVSeparateAVLatent → LTXVAudioVAEDecode (audio preserved)
    ↓
  LTXVTiledVAEDecode → ImageScale → CreateVideo (with audio) → SaveVideo
```

**Why two passes?** A single pass forces a tradeoff: high sigma = fat but face changes, low sigma = face preserved but not fat. Two passes separates those concerns — Pass 1 locks the structure at low sigma, Pass 2 can then go aggressive with the LoRA without destroying the face.

**Audio** is processed through the joint AV latent pipeline (`LTXVAudioVAEEncode → LTXVConcatAVLatent`), so the model is aware of audio context during generation. Audio is preserved in the output.

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

---

## Required Custom Nodes

| Node Pack | Required For |
|-----------|-------------|
| [ComfyUI-VideoHelperSuite](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | `VHS_LoadVideo`, `CreateVideo`, `SaveVideo` |
| [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) | `LTXVTiledVAEDecode`, `LTXVAudioVAEEncode/Decode`, `LTXVConcatAVLatent`, `LTXVSeparateAVLatent`, `LTXAVTextEncoderLoader`, `LTXVConditioning`, `LowVRAMCheckpointLoader` |

---

## How to Use

1. **Load the workflow** — drag the `.json` file into ComfyUI (or use Load → browse)
2. **Select your video** — click the `Load Video (+ Trim)` node, pick your file
3. **Queue** — hit Run. Output saves to `ComfyUI/output/LTX-2.3/Obese-V2V/`

Use the **9-16** workflow for portrait/vertical video, **16-9** for landscape/horizontal.

---

## Strength Controls

### Fat Strength — Pass 2 ManualSigmas

| Level | Sigmas | Notes |
|-------|--------|-------|
| Subtle | `0.5, 0.4, 0.3, 0.0` | Slight weight gain, maximum preservation |
| Moderate | `0.75, 0.65, 0.55, 0.45, 0.0` | Clearly heavier |
| **Strong (default)** | `0.9, 0.8, 0.7, 0.6, 0.5, 0.0` | Clearly obese |
| Extreme | `0.95, 0.85, 0.75, 0.65, 0.5, 0.0` | Maximum fat, some face drift |

### Motion Lock — Pass 1 ManualSigmas
Keep this **low**. Raising it above 0.4 defeats the purpose of two-pass.

| Level | Sigmas |
|-------|--------|
| **Default** | `0.3, 0.2, 0.1, 0.0` |
| Slightly stronger lock | `0.4, 0.3, 0.2, 0.0` |

### LoRA Strength (Obese LoRA node)
Default: **1.15**. Range: 0.8–1.3. Higher = fatter but more artifact risk.

---

## Trim Controls

| Parameter | Default | Description |
|-----------|---------|-------------|
| `frame_load_cap` | `0` | Frames to load (0 = entire video) |
| `skip_first_frames` | `0` | Start frame offset |

At 24fps: 120 frames = 5s · 240 = 10s · 480 = 20s

**Test first:** Use `frame_load_cap=120` (5 seconds) to verify results before running the full video.

---

## Expected Performance (RTX 5060 Ti 16GB)

| Video length | Approx time |
|-------------|-------------|
| 5s (120 frames) | ~5–7 min |
| 10s (240 frames) | ~8–12 min |
| 20s (480 frames) | ~15–20 min |

Processing resolution: **960px** (longest dimension) → upscaled to 1080p at output.

---

## Troubleshooting

**Not fat enough**
- Raise Pass 2 first sigma (e.g. `0.95` start)
- Raise LoRA strength toward 1.2–1.3

**Face / hair changing**
- Lower Pass 2 first sigma (e.g. `0.75` start)
- Keep Pass 1 sigmas low (`0.3, 0.2, 0.1, 0.0`)

**Background filling with extra people**
- Already handled in negative prompt — if it recurs, lower Pass 2 start sigma

**OOM / out of memory**
- Reduce processing resolution: change `960` → `832` in the `ImageScaleToMaxDimension` node
- Reduce `frame_load_cap` to process shorter chunks

**Blurry decode / tile seam**
- Portrait uses **1h×2v** tiling (no vertical seam through face)
- Landscape uses **2h×1v** tiling (no horizontal seam through frame)
- These are already set correctly per workflow

---

## License

Workflow JSON: MIT
bodypositivity LoRA: [TheBurgstall/bodypositivity-ltx2.3](https://huggingface.co/TheBurgstall/bodypositivity-ltx2.3)
LTX Video 2.3: [Lightricks Research License](https://huggingface.co/Lightricks/LTX-Video)
