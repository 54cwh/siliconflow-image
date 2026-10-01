# siliconflow-image

基于 **SiliconFlow（硅基流动）** 的图像生成 Skill，支持**文生图**与**图生图**。

> ⭐ **重点：本 Skill 调用的是 SiliconFlow 上的免费生图模型 [`Kwai-Kolors/Kolors`](https://siliconflow.cn)，`model` 参数固定为 `Kwai-Kolors/Kolors`，无需付费即可生成图片。**

## 功能

- 文生图（text-to-image）
- 图生图（image-to-image，参考图可用 URL 或 base64）
- 单次批量生成 1–4 张、指定尺寸 / 随机种子 / 采样步数 / 引导系数

## 前置条件

- 一个 SiliconFlow 账号与 API Key，设置环境变量：

  ```bash
  export SILICONFLOW_API_KEY="你的Key"
  ```

- `curl`，或 Python + `requests`

## 使用的模型

| 项 | 值 |
|---|---|
| 模型 ID | `Kwai-Kolors/Kolors` |
| 厂商 | 快手 Kwai（Kolors / 可图） |
| 价格 | **免费** |
| 接口 | `POST https://api.siliconflow.cn/v1/images/generations` |

## 快速开始

```bash
curl https://api.siliconflow.cn/v1/images/generations \
  -H "Authorization: Bearer $SILICONFLOW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Kwai-Kolors/Kolors",
    "prompt": "一只在月光下打坐的猫，水墨风格",
    "image_size": "1024x1024",
    "batch_size": 1
  }'
```

返回里的图片 URL **1 小时过期**，请立即下载到本地（默认输出目录 `~/siliconflow_outputs/`），并同时保存 `result.json` 元数据。

## 参数

| 参数 | 说明 |
|---|---|
| `model` | 固定 `Kwai-Kolors/Kolors`（免费） |
| `prompt` | 正向提示词 |
| `negative_prompt` | 负向提示词 |
| `image_size` | 默认 `1024x1024` |
| `batch_size` | 1–4，默认 1 |
| `seed` | 随机种子 |
| `num_inference_steps` | 采样步数，默认 20 |
| `guidance_scale` | 引导系数，默认 7.5 |
| `image` | 图生图参考图（URL 或 base64） |

## 说明

完整工作流、Kolors 支持的尺寸表、选参指引与错误码（400/401/403/404/429/503/504）见 [`SKILL.md`](./SKILL.md)。
