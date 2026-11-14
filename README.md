# Social Media Post with Image — Cloudflare FLUX

One form input → AI writes the post, title and image prompt → Cloudflare generates a high-quality image → the post goes live on Facebook with the image, and the image is saved to Google Drive.

No manual writing. No manual image design. Just topic + category.

## What it does

1. You fill a form with a **topic** and a **category** — Technology, Marketing, Sports or Education.
2. An AI Agent writes a ready-to-publish social media post — headline, 120–180 word description, call to action, 5–8 hashtags.
3. A second AI Agent writes a detailed image prompt for that topic — 100–180 words, 4:5 portrait, 1080 × 1350.
4. A third AI Agent generates 5 title options and picks the best recommended title.
5. Cloudflare Workers AI generates the image with **FLUX.1 Schnell** (`@cf/black-forest-labs/flux-1-schnell`).
6. The image is converted from Base64 to a file and posted in parallel to:
   - **Facebook** — photo + caption (Title 1 + full description)
   - **Google Drive** — the generated image is saved for reuse

## How it works

```
Form Trigger (Topic + Category)
  → Description Generator (AI Agent)
  → Image Generator Prompt (AI Agent)
  → Title Gen (AI Agent)
  → HTTP Request (Cloudflare Workers AI — FLUX.1 Schnell)
  → Convert to File (Base64 → generated-image.jpg)
  → Facebook Graph API (me/photos) ─┐
  → Google Drive Upload (file) ──────┘
```

All three AI Agents share the same model and search tool:

- **Model:** Groq `openai/gpt-oss-20b`
- **Search tool:** Tavily — used by the Description Generator and Image Prompt Agent so the content is based on real, current information, not invented facts

## Nodes — 10 working nodes

| Node | Type | What it does |
| --- | --- | --- |
| Posting for social media | Form Trigger | Input: main topic (required), category dropdown |
| Description Generator | AI Agent | Writes the full post: headline, description, CTA, hashtags. Uses Tavily search. |
| Image generator prompt | AI Agent | Writes a 100–180 word image prompt, 4:5 portrait style, no text inside the image |
| title gen | AI Agent | Generates 5 titles + Best Recommended Title |
| HTTP Request | HTTP Request | POST to Cloudflare Workers AI, model `@cf/black-forest-labs/flux-1-schnell`, `steps: 4`, prompt sliced to 2048 characters |
| Convert to File | Convert to File | Converts `result.image` (Base64) to `generated-image.jpg` |
| Facebook Graph API | Facebook Graph API | POST `v25.0`, `me/photos`, Send Binary File `file:data`, caption = Title 1 + description |
| Upload file | Google Drive | Uploads the image to Google Drive |
| Groq Chat Model | LM Chat Groq | Shared model for all 3 AI Agents |
| Tavily tool | Tavily Tool | Shared search tool for the Agents |

## The Facebook caption — how it is built

The caption is **not** the whole title list. It is built in the Facebook Graph API node, Query Parameter `caption`:

```
{{ $('title gen').item.json.output.split('\n')[0].replace('Title 1: ', '') }}

{{ $('Descirption Generator').item.json.output }}
```

- First line = Title 1 only, without the `Title 1:` prefix
- Then the full description from the Description Generator

That was the fix for the earlier error: putting the whole title output in the **Edge** field made Facebook return `Bad request — Unknown path components`. Edge must be only `photos`, and the text must go in `caption`.

## Setup

You need your own credentials — none are included in this repo.

1. **Import the workflow JSON** into n8n.
2. **Groq** — create a Groq API credential and connect it to the Groq Chat Model node.
3. **Tavily** — create a Tavily API credential and connect it to the Tavily tool node.
4. **Cloudflare** — create a Cloudflare API Token with Workers AI access, add it as an HTTP Bearer Auth credential named e.g. `Cloudflare image generate`, and set the HTTP Request URL to your own account:
   ```
   https://api.cloudflare.com/client/v4/accounts/<YOUR_CLOUDFLARE_ACCOUNT_ID>/ai/run/@cf/black-forest-labs/flux-1-schnell
   ```
5. **Facebook** — connect a Facebook Graph API credential with `pages_manage_posts` permission. The node posts to `me/photos` (v25.0).
6. **Google Drive** — connect Google Drive OAuth and set your own destination folder in the Upload file node.
7. Open the Form Trigger URL, enter a topic + category, submit, and check Facebook + Drive.

### Before you upload the JSON to GitHub

Replace your real IDs in the exported JSON first:

- Cloudflare Account ID in the HTTP Request URL → `<YOUR_CLOUDFLARE_ACCOUNT_ID>`
- Google Drive folder ID in the Upload file node → your own folder, or a placeholder
- Do not paste any API token, bearer token or secret anywhere in the repo. n8n exports credential IDs, not secrets — but check the file before uploading.

## Files

- `README.md` — this file
- `social-media-post-with-cloudflare-image-workflow.json` — the n8n workflow (rename the exported file to this name before uploading)

## Limitations

- **Facebook only for auto-posting.** This workflow does not post to LinkedIn or Instagram automatically. The LinkedIn post for this project is a manual share post.
- The title agent's system prompt asks for 5 titles and then for just 1 strong title — the output used in the caption is Title 1, which is also the Best Recommended Title in the tested run.
- Image generation depends on Cloudflare Workers AI availability and your account limits.
- The Description Generator is told not to invent statistics or factual claims — it uses Tavily search for that reason.

## Status

Working — tested end to end. Facebook returned a `post_id` for the photo + caption post, and the image was saved to Google Drive.

Built with n8n, Groq, Tavily, Cloudflare Workers AI (FLUX.1 Schnell), Facebook Graph API and Google Drive.
