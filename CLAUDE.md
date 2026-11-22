# Working notes for Claude

## Watching videos

The `watch` skill (`.claude/skills/watch/`) turns a video file into timestamped
frames and a word-level transcript. It runs locally: ffmpeg for frames, WhisperX
(small model, CPU) for speech. The environment's setup script installs WhisperX
at the start of each fresh session.

- **Local files** (uploaded in chat or fetched from Google Drive via the Google
  Drive connector): run the watch skill on the file directly.
- **Links (Instagram, Facebook, TikTok, YouTube, etc.)**: this cloud machine is
  blocked from downloading them directly (YouTube asks for bot sign-in; social
  CDNs need wildcard hosts). Use the connected **Apify MCP** instead:
  1. `search-actors` for a downloader for that platform (e.g.
     `apify/instagram-reel-scraper` with `includeDownloadedVideo`,
     `streamers/youtube-video-downloader` with `storeInKVStore: true`).
  2. Run it with a `maxTotalChargeUsd` cap; tell the user the cost.
  3. Download the saved file from `api.apify.com` (allowed in the environment),
     then run the watch skill on it locally.
  Not yet verified end to end: the first attempt was blocked by unpaid Apify
  invoices. Confirm the file download works on the first real run.
- No Gemini key is configured; the Gemini engine is not in use.

For word-level timestamps (precise trims, text on the exact word), run
WhisperX directly with alignment instead of the skill's default `--no_align`:

```bash
HF_HUB_DISABLE_XET=1 ~/.cache/watch/whisperx-venv/bin/whisperx audio.wav \
  --model small --device cpu --compute_type int8 --language en \
  --vad_method silero --output_format json --output_dir out
```

The user's editing style lives in the `rolland-edit-style` skill. Brand fonts are
in `brand-kit/fonts/`.
