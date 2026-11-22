# Working notes for Claude

## How to work with Rolland

Be proactive. When asked to do something and you don't know how, or whether it
will work, research it yourself first (web search, Apify docs via
`search-apify-docs`, tool docs) and come back with the answer. Don't ask Rolland
to find things out or hand the question back to him.

## Apify is the research tool, not just a downloader

The Apify MCP is connected. Use it for any social media or web research: search
the Apify Store for the scraper that best fits the goal, check its input schema,
and run it with a `maxTotalChargeUsd` cap (tell the user the cost).

- **"Research this page/profile/channel"** (Facebook, Instagram, TikTok, YouTube,
  etc.): use a profile/page/channel scraper for that platform and pull its posts
  and stats.
- **Studying a video** ("why is this reel doing so well", "why is this channel
  getting so many views", "was this the outlier"): pull *everything* about it,
  not only the file: caption copy, hashtags, thumbnail, title, views, likes,
  comments, shares, post date, duration, audio/music, and the account's other
  recent posts so the video can be compared against the account's normal
  performance (outlier = far above the account's typical views/engagement).
  Then watch the video itself (below) and explain why it worked: the hook,
  visuals, script, caption and packaging.

## Watching videos

The `watch` skill (`.claude/skills/watch/`) turns a video file into timestamped
frames and a word-level transcript. It runs locally: ffmpeg for frames, WhisperX
(small model, CPU) for speech. The environment's setup script installs WhisperX
at the start of each fresh session.

- **Local files** (uploaded in chat or fetched from Google Drive via the Google
  Drive connector): run the watch skill on the file directly.
- **Links (Instagram, Facebook, TikTok, YouTube, etc.)**: this cloud machine is
  blocked from downloading them directly (YouTube asks for bot sign-in; social
  CDNs need wildcard hosts). Use the connected **Apify MCP** to retrieve the
  video (alongside its metadata, as above):
  1. `search-actors` for a scraper/downloader for that platform (e.g.
     `apify/instagram-reel-scraper` with `includeDownloadedVideo`,
     `streamers/youtube-video-downloader` with `storeInKVStore: true`).
  2. Run it with a `maxTotalChargeUsd` cap; tell the user the cost.
  3. Download the saved file from `api.apify.com` (allowed in the environment),
     then run the watch skill on it locally. The account's storages are likely
     "Restricted" (Apify's default), so a bare record URL needs a token. In order:
     a. Use the pre-signed record URL the Actor puts in its dataset output, if any.
     b. Read the record through the Apify MCP resource (`resources/read` on
        `https://api.apify.com/v2/key-value-stores/{storeId}/records/{key}`);
        large records come back as a download URL.
     c. Get the store via the MCP (its response includes `urlSigningSecretKey`
        for writers) and sign the record URL per Apify's reference
        implementation (apify-client-js `key_value_store.ts` `getRecordPublicUrl`).
  Not yet verified end to end: the first attempt was blocked by unpaid Apify
  invoices. Record which route works on the first real run.
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
