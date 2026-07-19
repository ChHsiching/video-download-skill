---
name: video-download
description: Download a video at the best available quality as a single mp4, from YouTube or any of the thousand-plus sites yt-dlp supports (Bilibili, X (Twitter), Vimeo, etc.). Use when the user wants to download a video, mentions 下载视频 / 生肉下载 / yt-dlp, or gives a URL to fetch.
---

Download a video at the **best available quality** as a single merged mp4, plus its source metadata and cover thumbnail. Uses [yt-dlp](https://github.com/yt-dlp/yt-dlp), which supports a thousand-plus sites — YouTube, Bilibili, X (Twitter), Vimeo, and most others. This skill handles the defenses you'll hit on the hard ones — YouTube's **n-challenge** (JavaScript signature) and **login wall**, site-specific geo-blocks — by pairing yt-dlp with a Node runtime and, where needed, browser cookies.

## What you produce

Three files, all sharing one `<name>` stem, all under a per-video output directory:

1. `<name>.raw.mp4` — best available quality, video + audio merged into one file
2. `<name>.source.json` — the source's own metadata (title, uploader, description, duration, upload date, URL, thumbnail). Captured so downstream work doesn't have to re-fetch it.
3. `<name>.jpg` — the source video's cover thumbnail (best-scored by yt-dlp, normalized to jpg)

## Where the outputs land

Default layout — a per-video directory, one level under a folder named after the source author, with all three files inside `raw/`:

```
<output-root>/
└── raw/
    ├── <name>.raw.mp4
    ├── <name>.source.json
    └── <name>.jpg
```

- **`<output-root>`** defaults to `<cwd>/<author>/<video-name>/`. `<author>` and `<video-name>` are derived from the source metadata (uploader / title, lowercased with separators normalized). The user can override either.
- **`<name>`** defaults to `<video-name>` (same stem as the directory). The user can override it.
- **`raw/`** holds the original download.

This layout is a generic "organize downloaded videos" convention. A user who just wants the file gets a tidy folder; a user who later wants subtitles can point a downstream skill at the same `<output-root>` and `<name>`, and find all three files at that path.

## Environment reuse — never reinstall blindly

Before downloading, check what's already on disk. Three things are needed and each may already be present.

1. **yt-dlp binary**. Check the **current project directory first** — `./yt-dlp.exe` on Windows, `./yt-dlp` on macOS/Linux. A project-local copy is preferred over the one on PATH, because yt-dlp's challenge-solving EJS scripts drift out of date and you want a known-fresh binary you control. Then:
   - **Found locally** → use it. Verify capability (not version number) by running Step 1's `-F` later: if it returns only storyboard images on a JS-challenge site (YouTube most commonly), the binary is too stale to solve the current challenge.
   - **Not found locally** → download the latest release directly into the current directory:
     ```powershell
     # Windows (PowerShell)
     curl.exe -L -o yt-dlp.exe https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp.exe
     # macOS/Linux
     curl -L -o yt-dlp https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp && chmod +x yt-dlp
     ```
     The `/latest/download/` path always serves the newest release — no need to look up the tag first. Take the full `yt-dlp.exe`, never `yt-dlp_min.exe` (the min build ships without the EJS scripts this skill relies on).
   - **Found locally but stale** (Step 1's `-F` returns only storyboard on a site that should serve video) → re-download over the existing file using the same command above, then retry `-F`. The `/latest/download/` URL already serves the newest release; there is no newer channel to switch to.
   - **PATH has yt-dlp but the project doesn't** → still download a local copy. A project-local binary keeps the run reproducible and immune to whatever yt-dlp version happens to be globally installed.

   The PyInstaller-bundled `.exe` ships the EJS scripts already; no separate install for those. The completion criterion is "a project-local yt-dlp binary exists and Step 1's `-F` later returns real video/audio formats" — version numbers are not the signal, capability is.
2. **Node 22+**. yt-dlp solves the n-challenge in a JavaScript runtime. The default is **Deno**, which most Windows machines don't have — so you must pass `--js-runtimes node` explicitly. You also need `--remote-components ejs:github`, which tells yt-dlp to fetch the current EJS challenge-solving scripts from GitHub (the bundled ones drift out of date as sites change their challenges). Check `node --version`; require >= 22.0.0. If Node is missing, tell the user to install it (don't try to install it yourself). Required only when the site needs JS challenge solving; harmless otherwise.
3. **ffmpeg**. yt-dlp merges the separate video and audio streams into one mp4 via ffmpeg. Run `ffmpeg -version`; if missing, tell the user to install it.

The completion criterion for this step: a project-local yt-dlp binary exists (downloaded if needed), Node is on PATH (>= 22), and ffmpeg is on PATH. Don't proceed to Step 1 until all three are confirmed. The project-local binary's actual capability gets verified in Step 1's `-F` run.

## The pipeline

### Step 1 — Try the URL with no cookies first

Many sites serve public videos without login — most unauthenticated Bilibili / X / Vimeo downloads work with no cookies at all. Don't reach for the cookie machinery until you've confirmed you need it.

```bash
yt-dlp --js-runtimes node --remote-components ejs:github -F "<URL>"
```

`-F` lists available formats without downloading. `--js-runtimes node --remote-components ejs:github` solves the n-challenge on sites that use one (YouTube especially); harmless on sites that don't.

Evaluate the result:

- **Format table lists real video and audio formats** (not just `sb0`–`sb3` storyboard images) → cookies not needed. Skip to Step 3 with no `--cookies-from-browser` flag.
- **"Sign in to confirm you're not a bot" / login wall / age gate / members-only error** → the site requires cookies. Continue to Step 2.

Done when `-F` returns real formats, whether cookieless (skip Step 2) or after cookies (Step 2 success).

### Step 2 — Find a working cookie source when login is required

You're here because Step 1 hit an auth wall. The goal is a `--cookies-from-browser <source>` value that makes Step 1's `-F` return real formats. Try sources in order from least to most invasive; stop at the first that works.

**2a. Detect installed browsers, then try each one directly.** yt-dlp reads cookies straight from a browser's own profile — no copy needed — for the mainstream browsers it supports. Check which of these profile directories exist on the machine:

| Browser | yt-dlp name | Windows profile |
|---|---|---|
| Firefox | `firefox` | `%APPDATA%\Mozilla\Firefox\Profiles` |
| Chrome | `chrome` | `%LOCALAPPDATA%\Google\Chrome\User Data` |
| Edge | `edge` | `%LOCALAPPDATA%\Microsoft\Edge\User Data` |
| Brave | `brave` | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\User Data` |
| Doubao (豆包) | `chromium:<profile-path>` | `%LOCALAPPDATA%\Doubao\User Data` |

For each detected browser, retry Step 1's `-F` with `--cookies-from-browser <name>` added. Use the exact `name` from the table (Firefox/Chrome/Edge/Brave read directly; Doubao and other Chromium forks need the `chromium:<profile-path>` form because yt-dlp doesn't recognize their brand). Stop at the first one whose `-F` returns real formats; remember it for Step 3 and Step 4.

**2b. If direct reads all fail, copy the cookies to a temp directory.** This handles two failure modes: a Chromium browser holding an exclusive lock on its Cookies database, and a Chromium fork yt-dlp can't read in-place. **Ask the user which browser they're logged into the target site with** — don't guess; trying a browser where they aren't logged in wastes a cycle and the failure mode looks identical to "cookies broken".

Kill that browser's processes first (closing the window is not enough — Chromium browsers leave background processes alive that hold the lock):

```bash
# Windows — replace the process name (Doubao.exe, chrome.exe, msedge.exe, brave.exe)
taskkill /F /IM <browser>.exe
# verify nothing remains before copying
tasklist | findstr /I "<browser>"
```

Then copy both the Cookies database and its encryption key. Chromium encrypts cookie values with a key stored in `Local State`; you need both. yt-dlp expects the standard profile subdirectory layout:

```bash
# <UserData> = e.g. C:\Users\<user>\AppData\Local\Doubao\User Data
# <cookies-dir> = a Windows-native absolute path (e.g. C:\Users\<user>\AppData\Local\Temp\yt-cookies)
mkdir -p "<cookies-dir>\Default\Network"
copy "<UserData>\Default\Network\Cookies" "<cookies-dir>\Default\Network\Cookies"
copy "<UserData>\Local State" "<cookies-dir>\Local State"
```

Use a **Windows-native absolute path** for `<cookies-dir>`. Do not use a Git-Bash `/tmp/...` path — yt-dlp's cookie loader rejects it with "could not find chromium cookies database". Then retry `-F` with `--cookies-from-browser "chromium:<cookies-dir>"`.

**2c. If everything fails, stop and tell the user.** Report which browsers you tried and that none worked. Ask the user to log into the target site in some browser and re-run. Stop here — the failure mode of "tried a browser where they aren't logged in" looks identical to "cookies broken", so more guessing won't help.

Done when `-F` (with whatever cookie source you found) returns real video/audio formats. Carry that cookie source forward to Step 3 and Step 4.

### Step 3 — Download the video, the metadata, and the thumbnail

Set up the output paths first. Derive `<author>` and `<video-name>` from the metadata you saw in Step 1's `-F` run (or a quick `--dump-json --skip-download` if you skipped it). Defaults: `<author>` from the uploader/channel, lowercased with separators normalized to `-`; `<video-name>` from the title the same way; `<name>` equals `<video-name>`. Confirm `<output-root>` (default `<cwd>/<author>/<video-name>/`) and `<name>` with the user before downloading — these set the filename stem for every downstream artifact.

Then download all three artifacts in one `yt-dlp` call:

```bash
yt-dlp --js-runtimes node --remote-components ejs:github \
  [--cookies-from-browser "<source-from-step-2>"] \
  -f "bestvideo+bestaudio" --merge-output-format mp4 \
  --write-thumbnail --convert-thumbnails jpg \
  --dump-json \
  -o "<output-root>/raw/<name>.raw.%(ext)s" \
  "<URL>"
```

What each flag does:

- `-f "bestvideo+bestaudio"` picks the highest-resolution video and highest-bitrate audio streams separately; `--merge-output-format mp4` muxes them with ffmpeg. Sites that serve video and audio as separate adaptive streams above 720p (YouTube, others) require this two-stream + merge to actually get 1080p+ with sound. If the site only offers muxed streams, `bestvideo+bestaudio` falls back to the best muxed format automatically.
- `--write-thumbnail --convert-thumbnails jpg` writes the best-scored thumbnail (the source's own cover, not a video frame) normalized to jpg. Some sites serve webp; this normalizes it.
- `--dump-json` emits one JSON line with the source's full metadata to stdout. Redirect it to `<name>.source.json`. The full yt-dlp info-dict is preserved (notably `title`, `uploader`, `channel`, `duration`, `upload_date`, `webpage_url`, `description`, `thumbnail`).
- `-o "<output-root>/raw/<name>.raw.%(ext)s"` — note the literal `.raw` before `%(ext)s`. This makes the video land as `<name>.raw.mp4` (matching the documented stem), while the thumbnail (different `%(ext)s` resolution) lands as `<name>.jpg` because `--convert-thumbnails jpg` overrides the extension.

**Quality override.** The default takes the best quality the source offers. If the user asked for less (e.g. "1080p is fine", "skip 4K"), replace `-f "bestvideo+bestaudio"` with `-f "bv*[height<=1080]+ba/b[height<=1080]"` (caps height at 1080p, falls back to the best muxed stream at or under 1080p). Make this swap only when the user asks; the default is best-quality.

To capture the JSON to its file, redirect `--dump-json`'s stdout — for example, in PowerShell:

```powershell
yt-dlp ... --dump-json -o "<output-root>/raw/<name>.raw.%(ext)s" "<URL>" > "<output-root>/raw/<name>.source.json"
```

Done when `<output-root>/raw/<name>.raw.mp4` exists and `ffprobe` reports `duration > 0`:

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1 "<output-root>/raw/<name>.raw.mp4"
```

`duration > 0` means the file is a well-formed, playable mp4. The source JSON and the thumbnail are degraded-completion artifacts — some sources have no thumbnail, and a transient `--dump-json` failure shouldn't block the download. If either is missing, tell the user explicitly (they'll need a frame-grab cover later, and downstream metadata will have to be re-fetched) but the video itself counts as done. To preview which thumbnails a source offers before downloading, swap Step 1's `-F` for `--list-thumbnails`.

## Gotchas we hit and you will too

- **A fresh binary still couldn't solve the challenge.** If `-F` returns only storyboard images on a JS-challenge site even after re-downloading the latest GitHub release and confirming `--js-runtimes node --remote-components ejs:github` is set, the site has changed its challenge faster than yt-dlp has caught up. Check [yt-dlp issues](https://github.com/yt-dlp/yt-dlp/issues) for the current status. Per-site quirks are noted in yt-dlp's [support list](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md).

## Handoff

When `<name>.raw.mp4` is downloaded and verified, tell the user the files are ready, in `<output-root>/raw/`, sharing the stem `<name>`. If they want subtitles next, point the subtitle skill at `<output-root>` and `<name>` — all three files (mp4, source.json, jpg) are findable at that path under that stem. Don't run the subtitle skill from inside this one; the user may want to review the raw video first.
