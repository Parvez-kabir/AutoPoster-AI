
# PosenNigmaFinal — AI Social Media Auto-Poster

An **n8n workflow** that fully automates daily social media content creation and posting for a product brand ([TheBitz420](https://thebitz420.io)). Every day it picks a product, generates a fresh AI product photo, an AI video, and platform-specific captions — then posts automatically to **Facebook** and **Instagram**, alternating image/video between the two platforms each day.

<img width="1280" height="272" alt="image" src="https://github.com/user-attachments/assets/5e1d93da-f898-499e-bf1f-4ebda89b25c5" />


> ⚠️ This repository contains the workflow JSON with all credentials removed. You must connect your own credentials in n8n before running it. See [Setup](#setup).

---

## What it does

Triggered daily by a **Cron trigger**, the workflow:

1. **Picks the next product** — Lists product images from a Google Drive folder and rotates through them in order (tracked via a Google Sheet), so the same product isn't repeated back-to-back.
2. **Matches product info** — Looks for a matching Google Doc (by filename) containing that product's description, and fetches its text if found.
3. **Generates content with AI**:
   - **GPT-4o** writes a product description from the matched doc text (or falls back to a generic description).
   - **GPT-4o** generates a separate **Facebook caption** and **Instagram caption** in each platform's own style/tone.
   - **Gemini** analyzes the real product photo and writes a detailed AI image-generation prompt describing the product, then generates a brand-new **AI product photo** from that prompt.
   - **Veo 3** turns that AI-generated image into a short **AI video**.
4. **Alternates post format** — Each day flips the mode:
   - Day A: Facebook gets the **image**, Instagram gets the **video**
   - Day B: Facebook gets the **video**, Instagram gets the **image**
   (This mode is tracked in the tracking sheet so it keeps alternating correctly.)
5. **Uploads media** — Uploads the generated image/video to Cloudinary (for public hosting) and to Google Drive (for archiving).
6. **Publishes the posts** — Uses the `upload-post` node to publish the correct image/video + caption combo to Facebook and Instagram.
7. **Tracks results** — Checks whether both posts succeeded, logs the outcome (product used, mode, success/failure) to a Google Sheet post-log, and updates the rotation tracking sheet.
8. **Sends notifications** — Emails a success summary, or a failure alert if any step (image generation, video generation, or posting) breaks.

---

## Workflow architecture

```
Daily Cron Trigger
        │
        ▼
List Product Images (Drive) + Tracking Sheet (rotation state)
        │
        ▼
Compute Next Product & Post Mode (image/video rotation logic)
        │
        ▼
Match Product Doc → Fetch Description Text
        │
        ▼
GPT-4o: Product Description → FB Caption + IG Caption (in parallel)
        │
        ▼
Gemini: Analyze Real Photo → Generate Prompt → Generate AI Image
        │
        ▼
Veo 3: Generate AI Video from AI Image
        │
        ▼
Upload AI Image + Video to Google Drive
        │
        ▼
Build Final Post Data (decides which platform gets image vs video today)
        │
        ├── FB Image / IG Video path ──► Cloudinary Upload ──► Post to FB + IG
        └── FB Video / IG Image path ──► Cloudinary Upload ──► Post to FB + IG
        │
        ▼
Collect Results → All Succeeded? 
        ├── Yes ──► Update Tracking Sheet + Log Success + Success Email
        └── No  ──► Log Failure + Failure Email
```

---

## Requirements

| Service | Used for |
|---|---|
| [n8n](https://n8n.io) (self-hosted or cloud) | Running the workflow |
| Google Drive | Storing/reading product images & docs, archiving AI-generated media |
| Google Sheets | Rotation tracking + post success/failure logging |
| OpenAI API (GPT-4o) | Product descriptions and captions |
| Google Gemini API | Image analysis, prompt generation, AI image generation |
| Google Veo 3 (via Gemini) | AI video generation |
| [Cloudinary](https://cloudinary.com) | Public media hosting for posting |
| `upload-post` n8n community node | Publishing to Facebook & Instagram |
| Gmail | Success/failure email notifications |

---

## Setup

1. **Import the workflow**
   In n8n: *Workflows → Import from File* and select the workflow JSON.

2. **Connect credentials**
   Re-select credentials on every node that needs one (removed from this export):
   - `List Product Images`, `Upload AI Image to GDrive`, `GDrive – List Product Docs1`, `Download Product Image`, `Download AI Image`, `Upload AI Video to GDrive` → Google Drive OAuth2
   - `Tracking_sheet`, `Update Tracking Sheet2`, `Log Post to post_log Sheet2`, `Log Failure to post_log Sheet2` → Google Sheets OAuth2
   - `Gemini – Generate AI Image`, `Veo 3 – Generate AI Video` → Google Gemini API credential
   - `GPT-4o – Generate Description`, `Generate FB Caption`, `Generate IG Caption5` → OpenAI API credential
   - `Cloudinary – Upload FB Image/Video`, `Cloudinary – Upload IG Image/Video` → Cloudinary credential
   - `FB – Post Image/Video`, `IG – Post Image/Video` → `upload-post` credential
   - `Email – Success/Failure Notification` → Gmail OAuth2

3. **Set your Gemini API key**
   The `Gemini – Analyze Image → Prompt` node calls the Gemini API directly via HTTP — replace the `X-goog-api-key` header value (currently `YOUR_GEMINI_API_KEY`) with your real key.

4. **Set up the Google Sheet**
   Needs at least two tabs: one for rotation tracking (last used image, last post mode) and one for the post log (success/failure history).

5. **Prepare your Google Drive folders**
   One folder with product images (named so they can be sorted, e.g. `1_productname.png`), and matching Google Docs with the same base filename for product descriptions (optional — falls back to a generic description if missing).

6. **Test with a manual run** before enabling the schedule trigger, to confirm captions, generated media, and posts look correct.

---

## Notes

- No credentials or API keys are included in this repository — connect your own via n8n's credential store.
- The image/video mode alternates automatically each run based on the last recorded mode, so no manual toggling is needed.
- If AI video generation fails, the workflow can still post using the AI-generated image, depending on how the failure branches are configured.


