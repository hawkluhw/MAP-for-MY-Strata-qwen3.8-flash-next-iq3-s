# MAP for MY Strata — qwen3.8-flash-next-iq3_s (English)

One diagram: how the model's weights are actually distributed across **disk / RAM / VRAM**
while Strata serves qwen3.8-flash-next-iq3_s on the 9600X box (dual RTX 5060 Ti, 46 GiB RAM).

- `strata-memory-en.html` / `strata-memory-en.png` — this version
- `strata-memory.html` / `strata-memory.png` — 中文版

All numbers are live readings from 2026-11-25 22:40 (`free -h`, `nvidia-smi`, the engine's
`/metrics`, and directory listings).

## Key numbers

| Tier | Contents | Size |
|---|---|---|
| NVMe | experts.bin 46.8 GiB + PLE n-gram table ~25.6 GiB + dense.bin 1.4 GiB | — |
| RAM (46 GiB) | page-locked resident copy of the experts the GPUs cannot hold (`--resident-experts`) | 36.6 GiB |
| VRAM (2×16.1 GiB) | hot-expert cache 17.0 GiB (8,993 slots) + KV k8v4 @170k ctx + CUDA graphs / MTP drafter | — |

Result: decode is a stable **60–65 tok/s** (the previous `--mmap-experts` mode: 22–24 cold /
39–43 warm). Per generated token the hit waterfall is: VRAM expert cache → resident RAM copy
(over PCIe) → PLE n-gram rows (SSD, only ~2–3 MB/s).

Why total placement (17.0 + 36.6 GiB) exceeds the 48.2 GiB of runtime weights: the two tiers
overlap deliberately — the hottest experts live in both VRAM and RAM, so routing almost never
reaches a cold expert.

## How it was set up

```
--resident-experts   # replaced --mmap-experts in strata-iq3_s.json
```

`--resident-experts` is the engine-level low-RAM mode; `--low-ram` is only a setup/installer
option, never an engine flag. The mode applies silently at model start — confirm it via
`arena_mib` in `/metrics` and anonymous `used` in `free -h` (VmRSS and logs lie).

## The single weak point

RAM headroom is gone: `available` was ~2.7 GiB and ~2.7 GiB of anonymous swap is in use.
If it ever slows down, check `free -g` for a shrinking `available` and growing swap;
the fallback is `--resident-budget-gib N` (pin only the hottest N GiB) or reverting the config.

## Vision encoder enabled (2026-11-26)

The model ships with a vision tower (mmproj); the run config simply had it off. Now enabled, with a faster Q8_0 mmproj: a full image request takes 3.5 s (CPU reader, zero VRAM).
How-to, quantization and measurements: [vision-on-en.md](vision-on-en.md).

- `strata-memory-sankey.html` + `strata-memory-sankey-view1.png` / `strata-memory-sankey-view2.png` — interactive sankey version: (1) weights placement flow, (2) per-token hit waterfall. Open the HTML in a browser for hover tooltips; PNGs are renders of both views

## Updated map for v0.2.23 (2026-11-28)

- `strata-memory-v023-sankey-en.html` + `strata-memory-v023-sankey-view1-en.png` / `-view2-en.png`(中文: `strata-memory-v023-sankey.html` 等)— sankey map after the v0.2.23 / engine 0.1.70 upgrade: VRAM hot cache 17.0→20.0 GiB (10,589 slots via `--mtp-q4 all`), PLE now on `--ple-io mmap` but its page cache is squeezed by the pinned arena (Mapped only 0.9–4 GiB), anonymous swap 5.6 GiB; measured 47.7–61.2 tok/s
