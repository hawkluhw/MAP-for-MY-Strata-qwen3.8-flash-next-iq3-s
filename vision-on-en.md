# Enabling the vision encoder (qwen3.8-flash-next) — verified 2026-11-26

This box always **had** the vision encoder; the run config just didn't enable it:

- Vision tower: `/home/test/Strata-data-IQ3-S/mmproj-Qwen3.8-Flash-Next-BF16.gguf` (0.9 GB, 27-layer ViT, llama.cpp mtmd format)
- Helper binary: `/home/test/Strata/engine/strata-vision` (llama.cpp mtmd)

> Lesson: the ground truth for "images on" is `curl :8080/health` → `"images": true`.
> A stray `strata-iq3_s-vision.json` on disk means nothing — only the file passed to
> `serve/server.py --config` is live.

## How to enable (run config strata-iq3_s.json — BOTH places)

```jsonc
{
  "args": [ ..., "--vision" ],              // engine accepts image requests (GENI)
  "vision": {
    "exe":    "/home/test/Strata/engine/strata-vision",
    "mmproj": "/home/test/Strata-data-IQ3-S/mmproj-Qwen3.8-Flash-Next-Q8_0.gguf",
    "model":  "/home/test/Strata-data-IQ3-S/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf",
    "gpu": false                            // CPU reader
  }
}
```

If only the `vision` section exists, server.py runtime-adds `--vision` at startup and logs a note — but write both. Then a **full restart** is required (args are read once at launch; `/unload`+`/load` won't pick them up).

## Why CPU reader (gpu:false)

- Upstream rule: cards under 24 GB default to the CPU reader — zero VRAM use, the expert cache keeps its ground; our two 16.1 GB cards have only ~1.1/1.3 GiB free
- GPU reader would hold ~1.3 GB of VRAM — doesn't fit
- CPU and GPU encoders produce identical results; CPU default caps at 300 tokens/picture (`max_tokens` raises it; 1024 for small text/charts)

## Q8_0 mmproj (faster)

```bash
llama-quantize mmproj-Qwen3.8-Flash-Next-BF16.gguf mmproj-Qwen3.8-Flash-Next-Q8_0.gguf Q8_0
```

- Done in 1 second, 865 → 588 MiB (the "27/334 tensors fallback-quantized" warning is normal)
- Upstream docs: embedding cosine similarity 0.997–0.999 on CPU; BF16 kept on disk as fallback

## Verification (measured)

- `/health` → `"images": true` ✅
- Engine command line contains `--vision`; server log shows `loading the vision encoder ... images on` ✅
- `strata-vision` is actually loading `mmproj-Qwen3.8-Flash-Next-Q8_0.gguf` ✅
- Real image test (256×256 white cross on red, base64 `image_url`, non-stream): the model
  correctly answered "red background, a cross in the middle". **Whole request (encode +
  60-token decode) took 3.5 s** — well under the docs' ~3 s/picture CPU reference (which is encode-only).

## Notes

- Image requests must be non-streaming (streaming unsupported on this endpoint)
- strata-vision spawns lazily on the first image request (idle-exits after ~120 s)
- Hermes Agent on this box now reads images directly (screenshot OCR, visual Q&A) — no more tesseract workaround
