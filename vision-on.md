# 启用视觉编码器（qwen3.8-flash-next）— 2026-11-26 实测

这台机器其实**一直有视觉编码器**，只是运行配置没开：

- 视觉塔：`/home/test/Strata-data-IQ3-S/mmproj-Qwen3.8-Flash-Next-BF16.gguf`（0.9 GB，27 层 ViT，llama.cpp mtmd 格式）
- 辅助进程：`/home/test/Strata/engine/strata-vision`（llama.cpp mtmd）

> 教训：判断图像是否开启看 `curl :8080/health` 的 `"images": true` 字段——磁盘上存在
> `strata-iq3_s-vision.json` 不代表生效，生效的只有 `serve/server.py --config` 指向的那份。

## 开启方法（运行配置 strata-iq3_s.json，两处都要）

```jsonc
{
  "args": [ ..., "--vision" ],              // 引擎接受带图请求（GENI）
  "vision": {
    "exe":    "/home/test/Strata/engine/strata-vision",
    "mmproj": "/home/test/Strata-data-IQ3-S/mmproj-Qwen3.8-Flash-Next-Q8_0.gguf",
    "model":  "/home/test/Strata-data-IQ3-S/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf",
    "gpu": false                            // CPU 读图
  }
}
```

只有 `vision` 节没有 `--vision` 时，server.py 会在启动时补上这个 flag 并在日志里提示——但配置里最好两处都写全。改完必须**完整重启**（args 只在启动时读一次，`/unload`+`/load` 无效）。

## 为什么用 CPU 读图（gpu:false）

- 官方规则：单卡 <24 GB 默认就是 CPU reader——不动显存，专家缓存保住地盘；本机两张 16.1 GB 只剩 ~1.1/1.3 GiB 空闲
- GPU reader 要占 ~1.3 GB 显存，装不下
- CPU/GPU 编码结果完全一致；CPU 默认每图最多 300 token（`max_tokens` 可调，小字/图表可到 1024）

## Q8_0 mmproj（更快）

```bash
llama-quantize mmproj-Qwen3.8-Flash-Next-BF16.gguf mmproj-Qwen3.8-Flash-Next-Q8_0.gguf Q8_0
```

- 1 秒完成，865 → 588 MiB（"27/334 tensors fallback-quantized" 告警属正常）
- 官方文档称 CPU 场景 embedding 余弦相似度 0.997–0.999，BF16 保留作回退

## 验证（实测）

- `/health` → `"images": true` ✅
- 引擎命令行含 `--vision`；日志出现 `loading the vision encoder ... images on` ✅
- `strata-vision` 实际加载的就是 `mmproj-Qwen3.8-Flash-Next-Q8_0.gguf` ✅
- 真图测试（256×256 红底白色十字，base64 `image_url`，非流式）：模型正确答出
  "背景红色、中间是十字"。**整条请求（编码 + 60 token 生成）3.5 秒**，明显低于文档 CPU reader ~3 秒/图的参考值（那还只是编码部分）

## 备注

- 图像请求必须非流式（该端点不支持流式）
- 首次图像请求才会拉起 strata-vision（惰性启动，闲置 ~120 s 自动退出）
- 之后 Hermes Agent 在这台机器上可直接读图（截图 OCR、看图问答），不再需要 tesseract
