---
name: pixa-generate
description: Generate AI images or videos using the Pixa MCP tools. Handles model discovery, generation, async job completion, and rich display.
when_to_use: |
  When the user asks to generate, create, or make images, videos, photos, AI art, or media.
  Also when they provide a reference image for image-to-image generation.
  Triggers: "generate image", "create video", "AI photo", "generate media", "create image", "make a photo".
user-invocable: false
---

# AI Media Generation with Pixa MCP

Generate images and videos using Pixa's MCP tools. This skill orchestrates model selection, generation, job polling, and display.

## Workflow

### Step 1: Understand the request

Parse from the user's message:
- **Media type**: image or video
- **Subject/prompt**: what to generate
- **Style cues**: photorealistic, illustration, artistic, etc.
- **Aspect ratio**: 16:9, 1:1, 4:3, 4:5, 9:16, etc.
- **Quantity**: how many variations (max 4 per call)
- **Output format**: png, jpg, webp (image only, not applicable to video)
- **Reference image**: if the user provides an image to use as input (image-to-image)

### Step 2: Find the best model

**User specified a model?** → Call `pixa:models(action: "search", query: "<model name>")`.
**No model specified?** → Call `pixa:models(action: "recommend", type: "image"|"video")`.
**Recommend returns nothing?** → Fall back to `pixa:models(action: "list", type: "image"|"video")` and let the user pick.

If the user specifies input type (text prompt vs reference image), add `input: "text"|"image"`.
If they need a specific aspect ratio, add `aspect_ratio: "<ratio>"`.

Present the top recommendation to the user with its name, description, and capabilities.

### Step 3: Generate

Call `pixa:generate_media`:

```
pixa:generate_media(
  prompt: "<the prompt>",
  model: "<model_id from step 2>",
  aspect_ratio: "<ratio>",          # optional
  num_variations: <1-4>,            # optional, default 1
  output_format: "png"|"jpg"|"webp", # optional, image only
  media_type: "image"|"video",      # optional, inferred from model
  attachments: ["<url_or_asset_id>"] # optional, for image-to-image
)
```

The response returns immediately with:
- `job_id` — for tracking the async operation
- `asset_ids` — pre-allocated, durable IDs in the user's library

### Step 4: Wait for completion

**If the response renders a widget** (rich UI client): STOP. The widget auto-polls and displays results. Do NOT call `pixa:get_job_status`.

**In text-only mode**: Call `pixa:get_job_status`:

```
pixa:get_job_status(job_id: "<job_id>", sync: true)
```

This blocks for up to ~25 seconds. If the job is still running (common for video), call again. Repeat until status is `completed` or `failed`.

If status is `failed`, report the error message to the user and ask if they want to retry with a different model or prompt.

### Step 5: Display results

Call `pixa:display` with the returned `asset_ids`:

```
pixa:display(asset_ids: ["ast_...", "ast_..."])
```

## Image-to-Image Generation

When the user provides a reference image:

1. **Asset ID**: Pass directly in `attachments`
2. **URL**: Pass directly in `attachments`
3. **Local file**: First call `pixa:upload(method: "upload_url", filename: "image.jpg")`, execute the returned curl command, then use the resulting `asset_id` in `attachments`

When using attachments, filter models with `input: "image"` in the recommend call.

## Edge Cases

- **Max 4 variations** per `pixa:generate_media` call. For more, make multiple calls.
- **Video generation** can take minutes. Multiple `pixa:get_job_status` calls may be needed (each polls ~25s).
- **Model ID is required**. Always call `pixa:models` first — never guess a model ID.
- **URLs are temporary**. Always use `asset_id` for downstream operations, not URLs.
- **Credits consumed** on generation. Check with `pixa:account` first if the user seems budget-conscious.
