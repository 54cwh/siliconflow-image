---
name: siliconflow-image
description: >-
  Use SiliconFlow API to generate images using the Kwai-Kolors/Kolors model.
  Supports both text-to-image and image-to-image (with a reference image).
  Make sure to use this skill whenever the user says things like "画一幅...",
  "帮我生成一张图片", "做个封面图", "create an image", "generate a picture",
  "illustrate", "draw", "设计一个图片", "做一张配图", or any request involving
  visual content creation — even if they don't explicitly say "image generation"
  or "AI image." This skill handles the full pipeline: constructing the API
  request, downloading the generated image, and saving it locally to a file.
---

# SiliconFlow Image Generation Skill

Use this skill when the user wants to generate images via SiliconFlow's API
with the **Kwai-Kolors/Kolors** model. Both text-to-image and image-to-image
(with a reference image) are supported.

## Prerequisites

- **Environment variable**: `SILICONFLOW_API_KEY` must be set (the user has configured this)
- **API endpoint**: `https://api.siliconflow.cn/v1/images/generations`
- **Model**: `Kwai-Kolors/Kolors`

## Workflow Overview

1. Understand the user's request (text-to-image or image-to-image)
2. Build the API request with appropriate parameters
3. Call the API and handle the response
4. Download and save the generated image(s) locally
5. Save API metadata (result.json) alongside the image(s) for traceability
6. Report the result to the user

---

## Step 1: Understand the Task

### Text-to-Image
The user provides a text description — generate an image from it.

**Example**: "帮我画一只在月光下打坐的猫" → prompt: `"a cat meditating under moonlight, zen atmosphere, digital art style"`

### Image-to-Image (Img2Img)
The user provides a reference image + text description — generate a new image based on both.

**Example**: "把这个logo换个赛博朋克风格" → use the image as reference + prompt: `"transform this logo into cyberpunk style, neon lights, dark background"`

---

## Step 2: Build the API Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | `Kwai-Kolors/Kolors` | **Required**. The model name |
| `prompt` | string | — | **Required**. Image description |
| `negative_prompt` | string | None | What to exclude from the image |
| `image_size` | string | `1024x1024` | Format: `"widthxheight"` |
| `batch_size` | int | 1 | Number of images (1-4) |
| `seed` | int | None | Fixed seed for reproducible results (max 9999999999) |
| `num_inference_steps` | int | 20 | Quality vs speed (1-100) |
| `guidance_scale` | float | 7.5 | How strictly to follow the prompt (≤ 20) |
| `image` | string | None | Image URL or base64 data URL (for img2img) |

### Supported Image Sizes for Kolors

| Size | Aspect | Use Case |
|------|--------|----------|
| `1024x1024` | 1:1 | **Default**. Social media, universal |
| `960x1280` | 3:4 | Portrait, Xiaohongshu, Instagram feed |
| `768x1024` | 3:4 | Smaller portrait |
| `720x1440` | 1:2 | Vertical banner, Stories |
| `720x1280` | 9:16 | Phone wallpaper, TikTok/YouTube Shorts |

### Parameter Selection Guidance

- **image_size**: Default to `1024x1024` (most versatile). Adjust based on use case:
  - 社交媒体帖子 (social media posts) → `1024x1024`
  - 竖版/小红书风格 (vertical/portrait) → `960x1280` or `768x1024`
  - 横幅/封面图 (banner/cover) → `720x1440`
  - 手机壁纸 (phone wallpaper) → `720x1280`
- **guidance_scale**: 7.5 is a good balanced default.
  - Higher (10-15) → strict prompt adherence, good for specific subjects
  - Lower (3-7) → more creative/interpretive, good for artistic effects
- **num_inference_steps**: 20 is the default sweet spot.
  - More steps (30-50) → finer details but slower
  - Fewer steps (10-15) → faster but may lack detail
- **batch_size**: Use 4 if the user wants multiple variations without specifying each one individually
- **seed**: Only set if the user wants reproducible results or specifies a seed
- **negative_prompt**: Always suggest one if the user doesn't specify. Common defaults:
  - "blurry, low quality, distorted, ugly, deformed, bad anatomy, watermark, text"

---

## Step 3: Make the API Call

Use **bash/curl** for simple requests:

```bash
curl -s --request POST \
  --url "https://api.siliconflow.cn/v1/images/generations" \
  --header "Authorization: Bearer $SILICONFLOW_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{
    "model": "Kwai-Kolors/Kolors",
    "prompt": "a serene mountain lake at sunset, misty, photorealistic",
    "image_size": "1024x1024",
    "batch_size": 1,
    "num_inference_steps": 20,
    "guidance_scale": 7.5
  }'
```

Use **Python** for more complex workflows (img2img, base64 encoding, etc.):

```python
import requests, os, json

url = "https://api.siliconflow.cn/v1/images/generations"
api_key = os.environ["SILICONFLOW_API_KEY"]
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json",
}

payload = {
    "model": "Kwai-Kolors/Kolors",
    "prompt": "a serene mountain lake at sunset, misty, photorealistic",
    "image_size": "1024x1024",
    "batch_size": 1,
    "num_inference_steps": 20,
    "guidance_scale": 7.5,
}

resp = requests.post(url, json=payload, headers=headers)
resp.raise_for_status()
data = resp.json()
```

### Image-to-Image: Passing a Reference Image

For img2img, pass the image in the `image` field:

**If the image is already a URL:**
```python
payload["image"] = "https://example.com/reference.png"
```

**If the image is a local file, encode it as base64:**
```python
import base64

with open("/path/to/image.png", "rb") as f:
    b64 = base64.b64encode(f.read()).decode("utf-8")
payload["image"] = f"data:image/png;base64,{b64}"
```

---

## Step 4: Handle the Response

### Success Response (200)

```json
{
  "images": [{"url": "https://...png"}],
  "timings": {"inference": 0.1},
  "seed": 12345
}
```

**⚠️ IMPORTANT**: The image URL expires in **1 hour**. Always download it immediately.

### Save Metadata (result.json)

Alongside the image file, always save a `result.json` with the full API response and request parameters for traceability:

```python
# Save metadata alongside the image
metadata = {
    "model": payload["model"],
    "prompt": payload["prompt"],
    "image_size": payload.get("image_size"),
    "batch_size": payload.get("batch_size"),
    "num_inference_steps": payload.get("num_inference_steps"),
    "guidance_scale": payload.get("guidance_scale"),
    "seed": data.get("seed"),
    "timings": data.get("timings"),
    "image_url": image_url,
    "saved_image": filename,
}
with open("result.json", "w") as f:
    json.dump(metadata, f, indent=2)
```

### Download the Image

Save the image to a file with a descriptive name:

```bash
# Extract URL from the JSON response
IMAGE_URL=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['images'][0]['url'])")
FILENAME="mountain_lake_sunset_2026-05-30.png"
curl -sL "$IMAGE_URL" -o "$FILENAME"
echo "Saved to: $FILENAME"
```

Or in Python:

```python
import requests

image_url = data["images"][0]["url"]
filename = "mountain_lake_sunset.png"

img_resp = requests.get(image_url)
img_resp.raise_for_status()

with open(filename, "wb") as f:
    f.write(img_resp.content)

print(f"Image saved to: {filename}")
```

**Filename convention**: Use a descriptive name in English based on the image content, optionally with a date: `[content_description]_[YYYY-MM-DD].png`

### Error Responses

| Code | Meaning | Likely Cause | Action |
|------|---------|-------------|--------|
| 400 | Bad Request | Missing/invalid parameters | Check required fields (model, prompt, image_size) |
| 401 | Invalid Token | API key issue | Verify `SILICONFLOW_API_KEY` is set correctly |
| 403 | Forbidden | No permission | Check API key quotas |
| 404 | Not Found | Wrong endpoint | Verify the URL is correct |
| 429 | Rate Limited (IPM/TPM) | Too many requests — hitting per-minute image or token quota | Use batch_size to generate multiple images in one call. If hit, wait 30-60s and retry with fewer/batched requests |
| 503 | Service Overloaded | Model busy | Retry after a short wait |
| 504 | Gateway Timeout | Generation too slow | Try fewer inference steps or smaller image |

Always show the actual error message from the API response so the user knows what went wrong.

---

## Step 5: Report the Result

Tell the user:
1. ✅ Image generated successfully
2. **File saved at**: `/path/to/filename.png`
3. **Details**: resolution, seed, inference steps used
4. **Options**: suggest what they could change (style, prompt, size)

**Example output**:
```
✅ 图片已生成！

📁 保存位置: ~/siliconflow_outputs/cat_meditating_moonlight.png
📐 尺寸: 1024x1024
🔢 Seed: 12345
⚙️ 参数: 20 steps, guidance 7.5

想调整什么？我可以换风格、改 prompt、调尺寸，或者再生成几张变体。
```

---

## Best Practices

1. **Prompt engineering**: Write prompts in English for best results (the model was primarily trained on English prompts). If the user gives a Chinese description, translate it to English for the API call while telling the user what prompt was used.

2. **Quality over speed**: If the user doesn't specify, use defaults (20 steps, 7.5 guidance). Increase steps for higher quality if they're willing to wait.

3. **Multiple images**: When the user says "生成几张" or "几组方案", use `batch_size: 4` (max) to generate up to 4 variations in a single call. **Always prefer batch over multiple sequential calls** — this avoids the API's IPM (images-per-minute) rate limit and reduces total wait time from minutes to seconds.

4. **Seed preservation**: Always note the seed. If the user says "差不多但换个配色", you can regenerate with the same seed + modified prompt for consistency.

5. **Save directory**: Save images to the current working directory by default, or to a dedicated `siliconflow_outputs/` subdirectory if many images are being generated.

6. **Handle large base64 imgs**: For image-to-image with local files, base64 encoding can produce very long strings. Use Python for this (avoid inline bash with huge base64 data).

7. **Rate limit avoidance**: The API enforces an IPM (images-per-minute) quota. To stay under this limit:
   - Always use `batch_size` to batch multiple images into one API call
   - Never loop individual API calls for multiple images — this triggers 429 errors
   - If you get a 429, wait 30-60 seconds, then retry with `batch_size=1` or reduced batch count
