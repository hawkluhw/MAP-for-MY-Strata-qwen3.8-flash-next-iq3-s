# MAP for MY Strata — qwen3.8-flash-next-iq3_s

一张示意图：Strata 服务器（9600X，双 RTX 5060 Ti / 46 GiB RAM）上运行
qwen3.8-flash-next-iq3_s 时，模型权重在 **磁盘 / 内存 / 显存** 三层的实际分布。

- `strata-memory.html` / `strata-memory.png` — 中文版
- `strata-memory-en.html` / `strata-memory-en.png` — English version

数据为 2026-11-25 22:40 的实时读数（`free -h`、`nvidia-smi`、引擎 `/metrics`、目录清单）。

## 要点 / Key numbers

| 层 | 内容 | 大小 |
|---|---|---|
| NVMe | experts.bin 46.8 GiB + PLE n-gram 表 ~25.6 GiB + dense.bin 1.4 GiB | — |
| RAM (46 GiB) | `--resident-experts` 页锁定常驻副本（GPU 装不下的专家） | 36.6 GiB |
| VRAM (2×16.1 GiB) | 热专家缓存 17.0 GiB（8,993 槽） + KV k8v4 170k + 计算图/MTP | — |

效果：decode 稳定 60–65 tok/s（旧 `--mmap-experts` 模式：冷 22–24 / 热 39–43）。
每 token 的读取瀑布：VRAM 专家缓存 → 常驻 RAM 副本（PCIe）→ PLE 行（SSD，仅 ~2–3 MB/s）。

唯一薄弱点：RAM 水位（available ~2.7 GiB）。若变慢先查 `free -g` 与匿名 swap。
