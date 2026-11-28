# MAP for MY Strata — qwen3.8-flash-next-iq3_s

一张示意图：Strata 服务器（9600X，双 RTX 5060 Ti / 46 GiB RAM）上运行
qwen3.8-flash-next-iq3_s 时，模型权重在 **磁盘 / 内存 / 显存** 三层的实际分布。

- `strata-memory.html` / `strata-memory.png` — 中文版
- `strata-memory-en.html` / `strata-memory-en.png` — English version（详见 [README-en.md](README-en.md)）

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

## 视觉编码器已启用（2026-11-26）

本模型带视觉塔（mmproj），此前只是运行配置未开启。现已启用并换用更快的 Q8_0 mmproj：整条带图请求 3.5 秒（CPU 读图，不占显存）。
开启方法、Q8_0 量化与实测见 [vision-on.md](vision-on.md)。

- `strata-memory-sankey.html` + `strata-memory-sankey-view1.png` / `strata-memory-sankey-view2.png` — 桑基图版(交互):① 权重放置流 ② 每 token 命中瀑布。浏览器打开 HTML 可悬停看数值,PNG 为两个视图的渲染图

## v0.2.23 更新版地图(2026-11-28)

- `strata-memory-v023-sankey.html` + `strata-memory-v023-sankey-view1.png` / `-view2.png`(英文: `-en.html` / `-view1-en.png` / `-view2-en.png`)— v0.2.23/引擎 0.1.70 升级后的桑基图:显存热缓存 17.0→20.0 GiB(10,589 槽, `--mtp-q4 all`),PLE 改走 `--ple-io mmap` 但页缓存被 arena 挤压(Mapped 仅 0.9~4 GiB),匿名 swap 5.6 GiB;实测 47.7~61.2 tok/s
