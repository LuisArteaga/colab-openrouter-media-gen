# colab-openrouter-media-gen

Google Colab notebooks for AI media generation — images and videos created through the
[OpenRouter API](https://openrouter.ai/docs), so a single provider-agnostic endpoint can serve
every image and video model.

The repository ships the `Image_Generator.ipynb` notebook built on **Black Forest Labs FLUX.2 Pro**,
the high-end image generation and editing model from BFL, plus **FLUX.2 Klein 4B** as a budget
alternative for drafts and high-volume runs.

## Repository contents

| File | Description |
| --- | --- |
| `Image_Generator.ipynb` | Colab notebook. Text-to-image with FLUX.2 Pro (`black-forest-labs/flux.2-pro`) — aspect ratio, resolution, seed, output format; image editing via `input_references`; and a FLUX.2 Klein 4B (`black-forest-labs/flux.2-klein-4b`) budget cell. Optional Google Drive mount for reference images. |

## Open in Google Colab

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/LuisArteaga/colab-openrouter-media-gen/blob/main/Image_Generator.ipynb)

> Note: the badge only works once `Image_Generator.ipynb` is committed to this repository.

## Setup: OpenRouter API key

1. Create an API key at <https://openrouter.ai/settings/keys>.
2. Add credits to your account if needed (<https://openrouter.ai/credits>).
3. In Colab, store the key as a **secret** (key icon in the left sidebar) named
   `OPENROUTER_API_KEY`, then load it in the notebook:

```python
from google.colab import userdata

OPENROUTER_API_KEY = userdata.get("OPENROUTER_API_KEY")
```

Fallback (interactive prompt, as in the notebook):

```python
import getpass

OPENROUTER_API_KEY = getpass.getpass("Enter your OpenRouter API key")
```

Never hard-code the key into the notebook and never commit it.

## Image generation — `POST /api/v1/images`

OpenRouter's dedicated image endpoint takes a prompt (plus optional parameters) and returns
base64-encoded images:

```python
import base64
import requests
from io import BytesIO
from PIL import Image

response = requests.post(
    "https://openrouter.ai/api/v1/images",
    headers={
        "Authorization": f"Bearer {OPENROUTER_API_KEY}",
        "Content-Type": "application/json",
    },
    json={
        "model": "black-forest-labs/flux.2-pro",
        "prompt": "A retro travel poster of the Black Forest at dusk, with the bold legible headline 'FLUX.2 PRO' in art-deco lettering",
        "aspect_ratio": "1:1",
        "resolution": "1K",
    },
)
response.raise_for_status()
data = response.json()

image = Image.open(BytesIO(base64.b64decode(data["data"][0]["b64_json"])))
display(image)
image.save("output.png", format=data["data"][0].get("media_type", "image/png").split("/")[1])
```

Optional parameters: `n` (1–10), `resolution` (`512`, `1K`, `2K`, `4K`), `aspect_ratio`
(`1:1`, `16:9`, `9:16`, `4:3`, …), `size`, `quality` (`low`/`medium`/`high`), `output_format`
(`png`/`jpeg`/`webp`/`svg`), `background: "transparent"`, `seed`, and `input_references` for
image-to-image generation (HTTP URLs or base64 data URLs) — FLUX.2 Pro supports editing
existing images this way.

Budget alternative: `black-forest-labs/flux.2-klein-4b` — the fastest and most cost-effective
model in the FLUX.2 family, ideal for drafts.

Model discovery:

```bash
curl "https://openrouter.ai/api/v1/images/models"
```

or browse <https://openrouter.ai/models?output_modalities=image>.

## Video generation — `POST /api/v1/videos` (asynchronous)

Video generation is a background-job workflow: **submit → poll → download**.

```python
import time
import requests

headers = {
    "Authorization": f"Bearer {OPENROUTER_API_KEY}",
    "Content-Type": "application/json",
}

# 1) Submit the job (returns HTTP 202 with a job ID and polling URL)
job = requests.post(
    "https://openrouter.ai/api/v1/videos",
    headers=headers,
    json={
        "model": "google/veo-3.1",
        "prompt": "A slow cinematic push-in on a glowing neon sign in a rainy street at night",
        "duration": 5,           # seconds — check supported_durations per model
        "resolution": "1080p",   # 480p … 4K
        "aspect_ratio": "16:9",
    },
).json()

# 2) Poll until the job finishes (generation takes 30 s to several minutes)
while True:
    status = requests.get(job["polling_url"], headers=headers).json()
    print(f"Status: {status['status']}")
    if status["status"] in ("completed", "failed"):
        break
    time.sleep(30)

assert status["status"] == "completed", status.get("error", "generation failed")

# 3) Download the MP4 (authorization required — the URL is not presigned)
video = requests.get(
    f"https://openrouter.ai/api/v1/videos/{job['id']}/content?index=0",
    headers=headers,
)
with open("output.mp4", "wb") as f:
    f.write(video.content)
```

Image-to-video uses `frame_images` (with `frame_type: "first_frame"` / `"last_frame"`); style or
content guidance uses `input_references`. If both are provided, `frame_images` wins. Models also
support `generate_audio` and per-request webhooks via `callback_url`.

Model discovery:

```bash
curl "https://openrouter.ai/api/v1/videos/models"
```

or browse <https://openrouter.ai/models?output_modalities=video> (e.g. Veo 3.1, Seedance, Wan,
Hailuo).

## Notes

- Image billing is **all-or-nothing**: a failed or cancelled generation is not billed.
- Video generation is asynchronous; poll every ~30 s. Jobs can be delivered via webhook
  (`callback_url`) instead of polling.
- The OpenRouter account needs credits; per-model pricing is visible on the
  [models page](https://openrouter.ai/models).
- Reference docs:
  [Image generation](https://openrouter.ai/docs/guides/overview/multimodal/image-generation) ·
  [Video generation](https://openrouter.ai/docs/guides/overview/multimodal/video-generation)

## License

MIT — see [LICENSE](LICENSE).
