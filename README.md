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
  ├─ yt-dlp -F ──► format list (no cookies first; escalate only on auth wall)
  │     │
  │     ├─ --js-runtimes node --remote-components ejs:github ──► solves YouTube's n-challenge
  │     └─ --cookies-from-browser <detected> ──► passes the login wall (only if needed)
  │
  └─ yt-dlp -f bestvideo+bestaudio --merge-output-format mp4 \
       --write-thumbnail --convert-thumbnails jpg --dump-json
       ──► <name>.raw.mp4 + <name>.source.json + <name>.jpg
       (ffmpeg muxes the separate streams into one file)
```

YouTube (and other adaptive-streaming sites) serve video and audio as **separate streams** above 720p. Fetching `bestvideo+bestaudio` and merging is the only way to get 1080p+ with sound.

## Requirements

- **yt-dlp** with EJS support. The skill checks the project directory for a local `yt-dlp.exe` (Windows) or `yt-dlp` (macOS/Linux) first; if missing or stale (can't solve the current challenge), it downloads the latest release straight from [GitHub](https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp.exe) into the project directory. The PyInstaller-bundled `.exe` ships the EJS scripts already.
- **Node 22+** on PATH. The skill passes `--js-runtimes node --remote-components ejs:github` so yt-dlp can solve JS challenges using Node and fetch the current EJS scripts from GitHub.
- **ffmpeg** on PATH (merges the video + audio streams).
- **A logged-in browser** (only if the source requires login). The skill tries the URL with no cookies first; on an auth wall it detects installed browsers (Firefox, Chrome, Edge, Brave, Doubao) and tries each one's cookies in turn. Falls back to copying cookies to a temp directory only if direct reads all fail.

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

The skill fires, checks the environment (reuses existing binaries, doesn't reinstall), and downloads the best-quality mp4 plus its metadata and cover into `<cwd>/<author>/<video-name>/raw/`. For public videos it skips cookies entirely; for login-walled sources it walks detected browsers until one works. Confirm or override the `<author>` / `<video-name>` / `<name>` stem before the download starts.

## Handoff

This skill and [`video-subtitle`](https://github.com/ChHsiching/video-subtitle-skill) share a directory convention, so they chain by pointing at the same path:

```
video-download  ──►  <output-root>/raw/<name>.{raw.mp4, source.json, jpg}  ──►  video-subtitle  ──►  cooked mp4 + srt + upload.md
```

Both default `<output-root>` to `<cwd>/<author>/<video-name>/`, so running them back-to-back produces a single tidy folder with no moving or renaming between them. Run them by hand, one after the other — the user may want to review the raw video before committing to the (slower) subtitle pipeline.

## License

MIT
