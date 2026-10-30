# `models/` — 内置模型（不进 GPL，见 `MODEL-LICENSE.md`）

**文件名固定、不带版本号**（2026-10-29 起）。应用代码只认这四个名字
（`infer::MODEL_DENOISE` / `infer::aec::MODEL_AEC` / `infer::tse::MODEL_TSE` /
`infer::tse::MODEL_TSE_REF`）⇒ **换模型 = 换文件**，一行代码都不用改。

| 文件 | 接口（每 hop 10 ms / 48 kHz） | cache |
|---|---|---|
| `purevox_denoise.onnx` | `spec_in (1,2,481) + cache_in` → `spec_out + cache_out` | 45,988 |
| `purevox_aec.onnx` | `mic_spec + far_spec + cache_in` → `mic_spec_out + cache_out` | 91,920 |
| `purevox_tse.onnx` | `spec_in + keys (1,64,143) + gb (1,144) + cache_in` → `spec_out + cache_out` | 35,368 |
| `purevox_tse_ref_encoder.onnx` | `ref_spec (1,2,1001,481)` → `keys + gb`（每句一次） | — |

- **`cache_in` 维度随模型变**：应用侧运行时用 `cache_dim()` 从 session 输入形状读，不写死；
  但**模型池按路径缓存 session**，所以**替换文件后必须重启应用**才生效。
- **谱接口**：图内只有网络，输入/输出都是 `(1,2,481)` 复数谱；960/480 周期 Hann 分析与合成、
  OLA（逐样本除 Σw²）、1 hop 滞后都在应用侧（`src-tauri/src/infer/spec.rs`，三个模型共用一份）。
- **TSE 的两个文件必须成对**：`ref_encoder` 里是键投影与 FiLM 的权重，跟主图不配套时
  keys/gb 形状对不上。应用侧加载时按 metadata `model` + keys/gb 形状校验，不配套直接报错
  （硬失败），不会退化成直通。
- **替换流程**：成对文件必须同批替换；替换后重启应用。
- 每个文件里的 ONNX metadata 记着模型标识（`model=…`），可用
  `python -c "import onnx;print(onnx.load('…').metadata_props)"` 查看。

**为什么文件名不带版本号**：版本号进文件名就必然要在代码里同步改名，"换模型"就又变成
"改代码 + 重新发版"。现在的约定是文件内的 metadata 记标识、本表记部署现状，替换只动文件。
