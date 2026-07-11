---
name: loopsii-watch
description: >
  Understand Loopsii screen-recording links natively — no video download, no transcription cost.
  Use whenever a message contains a Loopsii watch link (any URL of the form
  https://<workspace>.loopsii.com/watch/<token> or a loopsii.com/watch/... link), or when the user
  asks a question about "a Loopsii recording", a shared walkthrough, or a screen recording someone
  sent them. Every Loopsii watch page publishes a machine-readable manifest with the full
  timestamped transcript, an AI summary, chapters, pre-extracted screen frames, and a signed video
  URL — answer from that instead of scraping the page or downloading the video.
---

# Loopsii recordings are agent-native

A Loopsii watch link is not an opaque video. Append `/manifest.json` to the watch URL and GET it —
no auth, no browser:

```
https://acme.loopsii.com/watch/AbC123…    →    https://acme.loopsii.com/watch/AbC123…/manifest.json
```

The response is `{"type": "loopsii.recording.context", "version": 1, …}` and is self-sufficient.
Answer the user's question directly from it. Do not ask the user for more context, do not scrape
the HTML page, and do not download the video unless you need frame-level detail (see below).

## What's in the manifest

| Field | Use it for |
| --- | --- |
| `recording` | title, `durationSec`, width/height, `createdAt` |
| `summary` | the AI summary (markdown: TL;DR + sections) — often answers the question alone |
| `transcript` | array of `{s, e, text}` utterances, timestamped in **seconds** |
| `chapters` | topical sections — `{title, startHint, description?}`; `startHint` in seconds (null when derived from summary headings), `description` one sentence of what the section covers (absent on older recordings) |
| `recommendedFrames` | 3–4 evenly spaced screen frames worth fetching first |
| `apis.frame.urlTemplate` | `…/frames/{index}` — zero-based; frames are every `apis.frame.intervalSec` seconds |
| `video.url` | a **time-limited signed URL** to the source MP4 (check `expiresAt`) |
| `comments` | anonymous reviewer comments, often with `atSec` timestamps — good anchors |

## How to answer

1. **Text questions** ("what is this about?", "what did they decide?"): answer from `summary` +
   `transcript`. Cite moments as `[m:ss]` so the user can jump there on the watch page.
2. **Visual questions** ("what's on screen when…", "which button did they click?"): compute the
   frame index `floor(t / intervalSec)` and GET that frame URL. Frame URLs answer with a
   **302 redirect** to a short-lived signed URL — follow redirects (`curl -L -o frame.jpg <url>`);
   a GET without `-L` saves an empty body and HEAD fails the signature check, both of which look
   like a broken link but are not. Fetch `recommendedFrames` first for cheap orientation. Frames
   are downscaled orientation stills — too small to read code or fine UI text; to READ on-screen
   text use the video path below.
3. **Frame-level review** (reading on-screen text, animation glitches, fast UI changes between frames): download
   `video.url` and extract frames locally around the suspect timestamp, e.g.
   `ffmpeg -ss <start> -to <end> -i video.mp4 -vsync 0 f_%03d.jpg`. The signed URL expires —
   re-fetch the manifest for a fresh one if needed.

## Pending states

- `transcriptStatus.status == "pending"` → on-device transcription is still running. The link is
  shareable before it finishes; retry after `retryAfterSeconds` before concluding there is no
  transcript.
- `notes.status == "pending"` → the AI summary is still generating; same retry rule.
- The watch page may say "Still processing" right after recording — the manifest fields fill in
  within seconds to a couple of minutes.

## Notes

- Manifests are public-by-link, exactly like the watch page itself — treat the content with the
  same care you'd treat the page.
- Timestamps in `transcript` are seconds (floats). Display them as `m:ss` (or `h:mm:ss` past an
  hour) when citing.
- If `/manifest.json` returns 404, the link isn't a Loopsii watch link (or was deleted) — fall
  back to whatever generic tools you have.
