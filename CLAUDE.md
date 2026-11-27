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

### Rolland's Instagram (@rollandelendu)

- Apify plan: **Starter** (paid), so `includeSharesCount: true` works on
  `apify/instagram-reel-scraper`. Shares are a key metric: his breakout reels
  are the ones with high shares.
- He posts each video as a **trial reel first** (shown to non-followers), then
  as a regular reel. Near-duplicates with the same duration posted minutes apart
  are trial/regular pairs, not mistakes. To tell them apart, run the scraper a
  second time with `skipTrialReels: true`; posts missing from that run are
  trials. Compare trials with trials and regulars with regulars, and compare
  each video's trial vs regular result.

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
  **What actually worked (Instagram, Nov 2026):** run `apify/instagram-reel-scraper`
  *without* `includeDownloadedVideo` (its download sub-run sat queued and timed
  out; aborted). Its dataset gives `videoUrl` and `displayUrl` (thumbnail):
  signed Instagram CDN links that download directly with curl. The CDN host
  (e.g. `scontent-ssn1-1.cdninstagram.com`) must be in the environment's allowed
  hosts; it can vary, so if a new `scontent-*` host shows up, add that exact host.
  This route costs only the scrape (~$0.003 per reel).
  **Facebook (worked Nov 2026):** `facebook.com/share/r/...` short links aren't
  accepted by scrapers; resolve them first with the Apify `apify--web-fetch`
  tool (its `fetch.loadedUrl` is the real `/reel/<id>/` URL). Then run
  `apivault_labs/facebook-reels-video-scraper` with `workflow: "videoUrls"` and
  that URL (~$0.004). Output has `videoMp4HdUrl`, caption, hashtags, reactions,
  comments, and Facebook's own captions (which can be the wrong language;
  transcribe with WhisperX instead). Views and shares came back null. The MP4
  host (e.g. `video-dfw6-2.xx.fbcdn.net`) must be allowed, same as Instagram.
  **TikTok (worked Nov 2026):** `clockworks/tiktok-video-scraper` with
  `postURLs` (short `tiktok.com/t/...` links work as-is),
  `shouldDownloadVideos: true` (~$0.003). The video lands in the run's
  key-value store; `get-key-value-store-record` on it returns a **signed
  api.apify.com link** (route b above) that curl downloads directly, so no new
  host approval is needed. Stats include plays, likes, comments, shares, saves.
  For the account comparison use `clockworks/tiktok-scraper` with `profiles`
  (this Actor requires `maxTotalChargeUsd` of at least $0.50).
  **YouTube (worked Nov 2026):** stats from `streamers/youtube-scraper`
  (`startUrls: [{url}]`, `maxResults: 1`; views, likes, comments, subscribers,
  description, duration). **Video: use `hipersoft/youtube-downloader`**
  (`videoUrls: [url]`, `format: "360"`). Race on a 9-min video (Nov 2026):
  finished in 15 s, ~$0.003 flat per video, file in Apify storage with a signed
  `downloadUrl` (curl, ~2 s, has audio). Fallback if it fails:
  `marielise.dev/youtube-video-downloader` (`urls: [{url}]`, `quality: "360"`,
  `residentialProxyMode: "disabled"`; 16 s, ~$0.01/min). Also worked:
  `thenetaji/youtube-video-downloader` (26 s, `saveMedia: true`, needs
  `region`), `scraper_one/yt-downloader` (23 s, ignores quality, serves from
  `d3ggl2n0g1f8w9.cloudfront.net`, which is allowed). **Avoid
  `streamers/youtube-video-downloader`: 14 min for the same video.**
  Pass `https://www.youtube.com/watch?v=ID` (strip `youtu.be` share params).
  Speed matters as much as price: when a scraper is slow, race alternatives.
  When Rolland only says "watch this", watch the video; don't add channel
  research unless he asks.
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
