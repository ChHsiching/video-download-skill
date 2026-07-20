# video-download

A skill that downloads a video at the best available quality as a single mp4, using [yt-dlp](https://github.com/yt-dlp/yt-dlp). Handles YouTube's anti-bot defenses (n-challenge + login wall) out of the box, and works on any of the thousand-plus sites yt-dlp supports. Designed as the **first step** in a two-skill chain: download the raw video here, then feed the mp4 to the [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) skill for transcription and bilingual subtitles.

Built and tested on a CPU-only Windows machine against a real YouTube video.

## What it produces

Three files in a per-video directory, all sharing one `<name>` stem:

| File | What it is |
|---|---|
| `<name>.raw.mp4` | Best available quality, video + audio merged into one file |
| `<name>.source.json` | The source's own metadata (title, uploader, description, duration, upload date, URL, thumbnail) — captured so downstream work doesn't re-fetch |
| `<name>.jpg` | The source's cover thumbnail (best-scored, normalized to jpg) |

Default layout: `<cwd>/<author>/<video-name>/raw/<name>.{raw.mp4,source.json,jpg}`. The user can override the directory or the stem.

Single responsibility: fetch the raw video and its metadata. Transcription, translation, and hard-burning subtitles are the video-subtitle skill's job.

## How it works

```
<URL>
  │
  └─ cook download <url>
       │
       ├─ yt-dlp -F ──► format list (no cookies first; escalate only on auth wall)
       │     ├─ --js-runtimes node --remote-components ejs:github ──► solves YouTube's n-challenge
       │     └─ --cookies-from-browser <negotiated> ──► passes the login wall (firefox→chrome→edge→brave)
       │
       └─ yt-dlp -f bestvideo+bestaudio --merge-output-format mp4 \
            --write-thumbnail --convert-thumbnails jpg --print-to-file
            ──► <name>.raw.mp4 + <name>.source.json + <name>.jpg
            (cook renames thumbnail .raw.jpg → .jpg; ffmpeg muxes the separate streams)
```

[`cook`](https://github.com/ChHsiching/video-cook) assembles every yt-dlp command correctly every time — no shell escaping traps, no stdout-redirect swallowing downloads, no forgotten thumbnail rename. The skill docs shrink to a one-step pipeline; the completion criterion is "cook exit 0".

YouTube (and other adaptive-streaming sites) serve video and audio as **separate streams** above 720p. Fetching `bestvideo+bestaudio` and merging is the only way to get 1080p+ with sound.

## Requirements

- **[`cook`](https://github.com/ChHsiching/video-cook)** CLI: `pip install video-cook[download]`
- **yt-dlp** (pulled in by cook's `download` extra). A project-local `yt-dlp.exe` is also supported if the pip version is too stale to solve the current challenge — see [REFERENCE.md](skills/video-download/REFERENCE.md).
- **Node 22+** on PATH (yt-dlp solves JS challenges via Node).
- **ffmpeg** on PATH (merges the video + audio streams).
- **A logged-in browser** (only if the source requires login). cook tries the URL with no cookies first; on an auth wall it detects installed browsers (Firefox, Chrome, Edge, Brave) and tries each in turn. Falls back to asking the user + temp-directory copy only if direct reads all fail.

## Install

```bash
npx skills add ChHsiching/video-download-skill
pip install video-cook[download]
```

Then make sure Node and ffmpeg are on PATH system-wide.

## Usage

Inside your agent, ask in plain language:

> 下载这个视频:https://www.youtube.com/watch?v=...

The skill fires, runs `cook doctor` to confirm the environment, then `cook download`. For public videos it skips cookies entirely; for login-walled sources cook walks detected browsers until one works. Confirm or override the `<author>` / `<video-name>` / `<name>` stem before the download starts.

To run the full download → subtitle chain in one command, use the [`video-cooking`](https://github.com/ChHsiching/video-cooking-skill) router: `/video-cooking <URL>`.

## Handoff

This skill and [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) share a directory convention, so they chain by pointing at the same path:

```
video-download  ──►  <output-root>/raw/<name>.{raw.mp4, source.json, jpg}  ──►  video-subtitle  ──►  cooked mp4 + srt + upload.md + shipment
```

Both default `<output-root>` to `<cwd>/<author>/<video-name>/`, so running them back-to-back produces a single tidy folder with no moving or renaming between them. Run them by hand, one after the other — the user may want to review the raw video before committing to the (slower) subtitle pipeline.

## License

MIT
