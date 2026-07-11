# video-download

A skill that downloads a video at the best available quality as a single mp4, using [yt-dlp](https://github.com/yt-dlp/yt-dlp). Handles YouTube's anti-bot defenses (n-challenge + login wall) out of the box, and works on any of the thousand-plus sites yt-dlp supports. Designed as the **first step** in a two-skill chain: download the raw video here, then feed the mp4 to the [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) skill for transcription and bilingual subtitles.

Built and tested on a CPU-only Windows machine against a real YouTube video.

## What it produces

| File | What it is |
|---|---|
| `<title>.mp4` | Best available quality, video + audio merged into one file |

Single responsibility: fetch the raw video. Transcription, translation, and hard-burning subtitles are the video-subtitle skill's job.

## How it works

```
<URL>
  │
  ├─ yt-dlp -F ──► format list (pick bestvideo + bestaudio)
  │     │
  │     ├─ --js-runtimes node ──► solves YouTube's n-challenge
  │     └─ --cookies-from-browser chromium:<copy> ──► passes the login wall
  │
  └─ yt-dlp -f bestvideo+bestaudio --merge-output-format mp4 ──► <title>.mp4
        (ffmpeg muxes the separate streams into one file)
```

YouTube (and other adaptive-streaming sites) serve video and audio as **separate streams** above 720p. Fetching `bestvideo+bestaudio` and merging is the only way to get 1080p+ with sound.

## Requirements

- **yt-dlp** with EJS support (the `stable` channel sometimes lags YouTube's challenge changes; use `nightly` — `yt-dlp --update-to nightly`). The PyInstaller-bundled `.exe` ships the EJS scripts already.
- **Node 22+** on PATH (yt-dlp defaults to Deno; this skill passes `--js-runtimes node`).
- **ffmpeg** on PATH (merges the video + audio streams).

## Install

Install into any agent project via the [skills.sh](https://skills.sh) installer:

```bash
npx skills add ChHsiching/video-download-skill
```

This works because the repo follows the standard layout the installer walks: a `skills/video-download/SKILL.md` with valid `name` + `description` frontmatter, plus a `.claude-plugin/plugin.json` manifest. The installer copies the skill into your agent's skills directory (`.agents/skills/`, `.claude/skills/`, etc.).

Then make sure yt-dlp, Node, and ffmpeg are on PATH system-wide.

## Usage

Inside your agent, ask in plain language:

> 下载这个视频:https://www.youtube.com/watch?v=...

The skill fires, checks the environment (reuses existing binaries, doesn't reinstall), acquires cookies from a logged-in Chromium browser if the site needs login, and downloads the best-quality mp4. For sites that don't require login (many public Bilibili / Twitter videos), it skips the cookie step.

When the download finishes, hand the mp4 path to the `video-subtitle` skill for transcription and bilingual subtitles.

## Handoff

This skill and [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) are designed to chain:

```
video-download  ──►  <title>.mp4  ──►  video-subtitle  ──►  cooked bilingual mp4 + srt + upload.md
```

Run them by hand, one after the other — the user may want to review the raw video before committing to the (slower) subtitle pipeline.

## License

MIT
