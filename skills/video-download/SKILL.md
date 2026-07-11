---
name: video-download
description: Download a video at the best available quality as a single mp4, from YouTube or any of the thousand-plus sites yt-dlp supports (Bilibili, Twitter/X, Vimeo, etc.). Use when the user wants to download a video, mentions 下载视频 / 生肉下载 / yt-dlp, or gives a URL to fetch. The downloaded mp4 is the raw input to the video-subtitle skill.
---

Download a video at the **best available quality** as a single merged mp4. Uses [yt-dlp](https://github.com/yt-dlp/yt-dlp), which supports a thousand-plus sites — YouTube, Bilibili, Twitter/X, Vimeo, and most others. This skill handles the defenses you'll hit on the hard ones — YouTube's **n-challenge** (JavaScript signature) and **login wall**, site-specific geo-blocks — by pairing yt-dlp with a Node runtime and, where needed, Chromium cookies. No packaging script: yt-dlp is a mature CLI, the skill calls it directly.

The output mp4 is the **raw input to the `video-subtitle` skill**. Run this skill first when the user gives a URL and wants subtitles; the two skills chain by hand (download → then subtitle).

## What you produce

1. `<title>.mp4` — best available quality, video + audio merged into one file

That's the only output. Single responsibility: fetch the raw video. Everything downstream (transcription, translation, hard-burn) is the `video-subtitle` skill's job.

## Environment reuse — never reinstall blindly

Before downloading, check what's already on disk. Three things are needed and each may already be present.

1. **yt-dlp binary**. Find an existing `yt-dlp.exe` (Windows) or `yt-dlp` (macOS/Linux). Check, in order:
   - The current project directory
   - `PATH` (run `yt-dlp --version`)
   Use the first one found. It **must** support EJS (the external JavaScript challenge solver) — the `stable` channel sometimes lags; if `-F` only returns storyboard images (no video/audio formats) on a site that requires JS challenge solving (notably YouTube), the version can't solve the n-challenge and you need the **nightly** channel: `yt-dlp --update-to nightly`, or download from `https://github.com/yt-dlp/yt-dlp/releases` if there's no binary at all. The PyInstaller-bundled `.exe` ships EJS scripts already; no extra install for those.
2. **Node 22+**. yt-dlp solves the n-challenge in a JavaScript runtime. The default is **Deno**, which most Windows machines don't have — so you must pass `--js-runtimes node` explicitly. Check `node --version`; require >= 22.0.0. If Node is missing, tell the user to install it (don't try to install it yourself). Required only when the site needs JS challenge solving; harmless otherwise.
3. **ffmpeg**. yt-dlp merges the separate video and audio streams into one mp4 via ffmpeg. Run `ffmpeg -version`; if missing, tell the user to install it.

The completion criterion for this step: you can name the exact yt-dlp binary path, Node is on PATH (>= 22), and ffmpeg is on PATH. Don't proceed to Step 1 until all three are confirmed.

## The pipeline

### Step 1 — Acquire cookies if the site requires login

Many sites — YouTube especially — block unauthenticated yt-dlp requests with a "Sign in to confirm you're not a bot" wall or an age/membership gate. Cookies from a logged-in browser session get past it. **Skip this step entirely for sites that don't require login** (many Bilibili / Twitter / public videos download fine with no cookies — try Step 2 first; only come back here if Step 2 fails with an auth error).

The user must have already logged into the target site in some Chromium-based browser (Chrome, Edge, Doubao/豆包, Brave, etc.). Ask which one if it isn't obvious.

**The file-lock trap.** Chromium browsers keep an exclusive lock on the Cookies database while running. Closing the window is not enough — many of these browsers (Doubao especially) leave dozens of background processes alive. You must kill them all first:

```bash
# Windows — replace the process name as needed (Doubao.exe, chrome.exe, msedge.exe, brave.exe)
taskkill /F /IM Doubao.exe
```

Verify with `tasklist | grep -i doubao` that nothing remains before copying.

**Copy the cookies + the encryption key.** Chromium encrypts cookie values with a key in `Local State`. You need both files, and yt-dlp expects the standard profile subdirectory layout (`Default/Network/Cookies`):

```bash
# <UserData> = e.g. C:\Users\<user>\AppData\Local\Doubao\User Data
mkdir -p "<cookies-dir>\Default\Network"
copy "<UserData>\Default\Network\Cookies" "<cookies-dir>\Default\Network\Cookies"
copy "<UserData>\Local State" "<cookies-dir>\Local State"
```

Use a **Windows-native absolute path** for `<cookies-dir>` (e.g. `C:\Users\<user>\AppData\Local\Temp\yt-cookies`). Do **not** use a Git-Bash `/tmp/...` path — the nightly build's cookie loader won't resolve it and you'll get "could not find chromium cookies database".

Done when both `<cookies-dir>\Default\Network\Cookies` and `<cookies-dir>\Local State` exist (or when you've confirmed Step 2 works without cookies and skipped this).

### Step 2 — List formats and confirm the challenge is solved

```bash
yt-dlp --js-runtimes node [--cookies-from-browser "chromium:<cookies-dir>"] -F "<URL>"
```

Include `--cookies-from-browser` only if Step 1 produced a cookies directory. `--js-runtimes node` is needed for sites with JS challenges (YouTube); harmless elsewhere. `-F` lists what's available without downloading.

Done when the format table lists real video and audio formats (not just `sb0`–`sb3` storyboard images). If you only see storyboard images on YouTube, the n-challenge wasn't solved — re-check `--js-runtimes node` and Node >= 22, and that yt-dlp is on nightly. If you get a "Sign in" / bot error, you need cookies — go back to Step 1.

### Step 3 — Download and merge at best quality

```bash
yt-dlp --js-runtimes node [--cookies-from-browser "chromium:<cookies-dir>"] -f "bestvideo+bestaudio" --merge-output-format mp4 -o "%(title)s.%(ext)s" "<URL>"
```

`-f "bestvideo+bestaudio"` picks the highest-resolution video stream and the highest-bitrate audio stream separately, then `--merge-output-format mp4` muxes them with ffmpeg into one file. Sites that serve video and audio as separate adaptive streams above 720p (YouTube, others) require this two-stream + merge approach to actually get 1080p+ with sound. If the site only offers muxed streams (video+audio together), `bestvideo+bestaudio` falls back to the best muxed format automatically.

Done when `<title>.mp4` exists and `ffprobe` reports a positive duration. Verify:

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1 "<title>.mp4"
```

A duration > 0 means the file is playable end to end.

## Gotchas we hit and you will too

- **`--js-runtimes node` is not optional (for JS-challenge sites).** yt-dlp defaults to Deno. On a machine without Deno, omitting this flag means YouTube's n-challenge fails silently and you only get storyboard images — no video. Always pass `--js-runtimes node` (or `--js-runtimes node:/path/to/node` if Node isn't on PATH) when the site needs JS solving.
- **Stable may lag.** Sites change their JS challenges frequently. If nightly doesn't work today, the challenge may have changed again — check yt-dlp issues. The bundled EJS scripts update with the binary.
- **Closing the browser window ≠ exiting the browser.** Chromium-based browsers keep background processes running. `taskkill /F /IM <browser>.exe` and verify with `tasklist` before copying cookies. A locked Cookies database throws "Could not copy Chrome cookie database" or "Device or resource busy".
- **Cookies-copy directory layout matters.** yt-dlp looks for `<dir>\Default\Network\Cookies` and `<dir>\Local State`. Recreate that subtree when copying.
- **Use Windows paths, not MSYS/Git-Bash paths.** `C:\Users\...\Temp\yt-cookies` works; `/tmp/yt-cookies` does not — the nightly cookie loader rejects it with "could not find chromium cookies database".
- **The `--cookies-from-browser` value is `chromium:<path>`, not `chrome:<path>`.** The `chromium:` prefix covers Chrome, Edge, Brave, Doubao, Vivaldi — anything Chromium-based. `chrome:` only matches Google Chrome's install path.
- **Not every site needs cookies or JS solving.** Try the plain `-F` first on a new site; only add the cookie machinery if you hit an auth wall. yt-dlp's [support list](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md) notes per-site quirks.

## Handoff to video-subtitle

Once `<title>.mp4` is downloaded and verified, tell the user the file is ready and that the next step is the `video-subtitle` skill — give it the mp4 path and ask for bilingual or single-language subtitles. Don't run that skill from inside this one; the user may want to review the raw video first.
