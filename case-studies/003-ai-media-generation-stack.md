# 003 — AI Image and Video Generation Stack: DALL-E → Runway → Gemini

> **Status:** Complete  
> **Stack:** n8n · DALL-E · Runway · Gemini 3 Pro Image · Gemini Veo 3.1 · WordPress Media Library  
> **Context:** DMS Layer 4 (Media Generation) at Dot.ID

---

## The Goal

Fully automated AI-generated images and videos for social media posts — one per
scheduled post, generated from the content brief (caption, trend topic, campaign
theme), uploaded to WordPress, and published to social platforms without any manual
step.

Requirements:
- Accepts a text prompt from n8n
- Returns an image or video binary (not a URL to a third-party viewer)
- Can be called from an n8n HTTP Request node
- Output is usable directly for social media publishing

---

## What We Tried

### Attempt 1 — DALL-E (OpenAI Images API)

**Why we tried it:** Most familiar, well-documented, integrates cleanly with
n8n via HTTP Request. Generates images from a text prompt in one API call.

**How we set it up:**

```
POST https://api.openai.com/v1/images/generations
Authorization: Bearer {OPENAI_KEY}
{
  "model": "dall-e-3",
  "prompt": "...",
  "n": 1,
  "size": "1024x1024",
  "response_format": "b64_json"
}
```

**What happened:** Generated acceptable images for some content categories but
struggled with Arabic-language prompts and brand-consistent outputs. The bigger
issue: cost and rate limits at production volume (one image per post × multiple
clients × daily cadence) made it expensive. Removed 2026-04-30.

**Verdict:** ⚠️ Partial — worked technically, dropped for cost and quality reasons

---

### Attempt 2 — Runway (video generation)

**Why we tried it:** Runway Gen-3 Alpha was a leading video generation API at
the time. Wanted short video clips alongside static images for Reels/Stories.

**How we set it up:** HTTP Request node hitting the Runway API with a text prompt
and image input. Async job pattern (submit → poll → download).

**What happened:** Integration worked in isolation, but the async polling pattern
added workflow complexity, and the output quality for branded social content was
inconsistent. Removed 2026-04-30 when Gemini Veo became available.

**Verdict:** ⚠️ Partial — worked, replaced by better alternative

---

### Attempt 3 — Gemini 3 Pro Image (images) + Gemini Veo 3.1 (video)

**Why we tried it:** Google's Gemini image generation was available via the
`generativelanguage.googleapis.com` endpoint with the same API key as text
generation — no new account, no new billing. Veo for video followed the same
pattern.

**Image generation — how we set it up:**

```
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent?key={GEMINI_KEY}
Content-Type: application/json

{
  "contents": [{
    "parts": [{ "text": "Your prompt here" }]
  }],
  "generationConfig": {
    "responseModalities": ["IMAGE", "TEXT"]
  }
}
```

Response contains `inlineData.data` (base64) and `inlineData.mimeType`.
A Code node ("Extract Gemini Image") converts the inline base64 to an n8n
binary item:

```js
const response = $input.first().json;
const part = response.candidates[0].content.parts.find(p => p.inlineData);
const b64 = part.inlineData.data;
const mimeType = part.inlineData.mimeType;
const buffer = Buffer.from(b64, 'base64');
return [{
  json: { mimeType },
  binary: {
    data: await this.helpers.prepareBinaryData(buffer, 'image.jpg', mimeType)
  }
}];
```

Fallback model if `gemini-3-pro-image-preview` is unavailable: `gemini-2.5-flash-image`.

**Video generation — how we set it up:**

Veo uses a long-running operation pattern:

```
POST https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-generate-preview:predictLongRunning?key={GEMINI_KEY}
{
  "instances": [{ "prompt": "Short video of..." }],
  "parameters": { "aspectRatio": "9:16", "durationSeconds": 5 }
}
```

Returns an operation name. A separate "checker" workflow polls until done:

```
GET https://generativelanguage.googleapis.com/v1beta/{operationName}?key={GEMINI_KEY}
```

When `done: true`, download from:
`response.generateVideoResponse.generatedSamples[0].video.uri`
(append `?key={GEMINI_KEY}` to authenticate the download).

The Detailed Sheet stores `GEMINI_OP:{operationName}` in the video link column
until the checker replaces it with the real WordPress URL.

Fallback model: `veo-3.0-generate-001`.

**What happened:** Both worked reliably. Image quality was better than DALL-E
for our use cases, multilingual prompts handled well, and the single API key
for both text + image + video simplified the workflow significantly.

**Verdict:** ✅ Worked

---

## What We Shipped

```
Layer 4 — Media Generator (per post):
  [Read post brief from Google Sheets]
      → [Build image prompt (Code node)]
      → [Gemini 3 Pro Image API]
      → [Extract Gemini Image (Code node)]
      → [Upload to WordPress /wp/v2/media]
      → [Write image URL + WP media ID to Sheets]

Layer 4 — Video Generator (async):
  [Read post brief]
      → [Build video prompt]
      → [Veo 3.1 predictLongRunning]
      → [Write GEMINI_OP:{name} to Sheets]

Layer 4 — Video Checker (cron):
  [Read rows with GEMINI_OP: prefix]
      → [Poll GET /{operationName}]
      → [If done: download URI → upload to WordPress → update Sheets]
```

---

## Key Takeaway

Gemini's unified API key for text + image + video eliminates credential sprawl.
The inline base64 response for images avoids the async polling complexity of
other providers. For video, the long-running operation pattern adds a checker
workflow but is manageable. Start with Gemini before reaching for DALL-E or
Runway — fewer accounts, lower cost, comparable or better quality.

---

## Related

- [`tools/wordpress-media.md`](../tools/wordpress-media.md) — where images land after generation
- [`case-studies/002-n8n-binary-data-mode.md`](002-n8n-binary-data-mode.md) — binary handling gotchas when processing Gemini output
- [`case-studies/001-social-media-image-publishing.md`](001-social-media-image-publishing.md)
