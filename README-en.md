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
