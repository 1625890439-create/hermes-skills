---
name: douyin-video-summarizer
description: "Summarize Douyin (抖音) videos from share links: extract audio → speech-to-text → AI summary"
version: 1.1.0
author: Hermes
license: MIT
metadata:
  hermes:
    tags: [douyin, video, summary, stt, whisper, media]
---

# Douyin Video Summarizer

Summarize Douyin (抖音) video content from share links. Pipeline: parse link → extract CDN URL → extract audio → speech-to-text → AI summary.

## Prerequisites

- `ffmpeg` — audio extraction (usually pre-installed)
- `faster-whisper` — local STT engine: `pip install faster-whisper`
- `opencc-python-reimplemented` — 繁转简: `pip install opencc-python-reimplemented -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com`
- Whisper base model — local at `/mnt/c/Users/admin/Desktop/Hermes/model/faster-whisper-base`
- Browser tool (Browserbase) — for accessing Douyin web pages

## Working Pipeline (Do These, Skip the Rest)

### Step 1: Parse Short Link via Windows CMD

Douyin share messages contain short links like `https://v.douyin.com/xxxxx/`.

**⚠️ Must use Windows CMD curl, NOT WSL curl** (WSL curl times out on douyin short links):

```bash
cmd.exe /c "curl -sI -L --max-time 15 https://v.douyin.com/xxxxx/" 2>&1 | grep -i "location"
```

Extract the video ID (numeric) from the redirect URL. Format: `https://www.douyin.com/video/XXXXXXXXXXX`

### Step 2: Browser Open Video Page & Extract CDN URL

Open the full video page URL in browser tool:
```
https://www.douyin.com/video/<VIDEO_ID>
```

**Pitfall:** `browser_navigate` to Douyin may timeout. If so, retry once.

Then extract CDN URL from DOM via `browser_console`:

```javascript
(() => {
    const videos = document.querySelectorAll('video');
    for (const v of videos) {
        const sources = v.querySelectorAll('source');
        if (sources.length > 0) return sources[0].src;
        if (v.currentSrc) return v.currentSrc;
    }
    return 'NOT_FOUND';
})()
```

### Step 3: Extract Audio with ffmpeg

Use ffmpeg to extract audio directly from CDN URL (no need to download full video):

```bash
ffmpeg -referer "https://www.douyin.com/" \
  -user_agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -i "<CDN_URL>" \
  -vn -acodec libmp3lame -q:a 2 audio.mp3 -y
```

**⚠️ CRITICAL: `-referer "https://www.douyin.com/"` is mandatory.** Without it, CDN returns 403 Forbidden.

**⚠️ CRITICAL: Must unset proxy env vars first.** WSL may have leftover proxy settings that block CDN access:
```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY all_proxy ALL_PROXY
```

**Pitfall:** Use the full CDN URL including all query parameters. Truncated URLs may fail.

### Step 4: Transcribe with faster-whisper + opencc (繁转简)

Use the locally downloaded Whisper model + OpenCC for simplified Chinese output:

```python
from faster_whisper import WhisperModel
from opencc import OpenCC

MODEL_PATH = "/mnt/c/Users/admin/Desktop/Hermes/model/faster-whisper-base"
cc = OpenCC("t2s")

model = WhisperModel(MODEL_PATH, device="cpu", compute_type="int8")
segments, info = model.transcribe("audio.mp3", language="zh", beam_size=5)

for seg in segments:
    text = cc.convert(seg.text)
    print(text)
```

**Pitfall:** Whisper outputs traditional Chinese (繁体). Always use `OpenCC("t2s")` to convert.

### Step 5: AI Summary

Summarize the transcript with this prompt:

```
请总结以下视频转录内容，提取关键要点，用清晰的中文列出。要求：
1. 视频标题/主题
2. 关键信息点（编号列表）
3. 一句话总结
```

## ❌ Steps That Don't Work (Do NOT Try)

| Failed Approach | Why |
|---|---|
| `yt-dlp` download | Douyin requires cookies, Edge cookies are v20 encrypted and locked |
| WSL `curl` to resolve short links | Times out (no proxy access from WSL) |
| `browser_navigate` → search page | Triggers CAPTCHA verification |
| Copy Edge cookies via PowerShell/robocopy | Edge holds kernel-level exclusive file lock |
| `faster-whisper` with HF model download | HuggingFace blocked from WSL without proxy |
| Xiaomi MiMo API for STT | No STT model available, only TTS |

## Complete Working Script

```python
#!/usr/bin/env python3
"""Douyin video transcript pipeline (all steps verified working)"""
import subprocess, sys

MODEL_PATH = "/mnt/c/Users/admin/Desktop/Hermes/model/faster-whisper-base"
AUDIO_PATH = "/tmp/video-summary/audio.mp3"
TRANSCRIPT_PATH = "/tmp/video-summary/transcript.txt"

def extract_audio(cdn_url, referer="https://www.douyin.com/"):
    """Step 3: ffmpeg extract audio from CDN URL"""
    cmd = [
        "ffmpeg", "-referer", referer,
        "-user_agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
        "-i", cdn_url,
        "-vn", "-acodec", "libmp3lame", "-q:a", "2",
        AUDIO_PATH, "-y"
    ]
    subprocess.run(cmd, capture_output=True)
    return AUDIO_PATH

def transcribe():
    """Step 4: faster-whisper + opencc 繁转简"""
    from faster_whisper import WhisperModel
    from opencc import OpenCC

    cc = OpenCC("t2s")
    model = WhisperModel(MODEL_PATH, device="cpu", compute_type="int8")
    segments, info = model.transcribe(AUDIO_PATH, language="zh", beam_size=5)

    full_text = ""
    for seg in segments:
        text = cc.convert(seg.text)
        full_text += f"[{seg.start:.1f}-{seg.end:.1f}] {text}\n"

    with open(TRANSCRIPT_PATH, "w") as f:
        f.write(full_text)
    return full_text
```

## Pitfalls & Troubleshooting

| Issue | Solution |
|---|---|
| `browser_navigate` times out on Douyin | Retry once; Douyin pages are heavy |
| ffmpeg HTTP 403 Forbidden | Missing `-referer` flag; add `-referer "https://www.douyin.com/"` |
| ffmpeg connection timeout | Proxy env vars left over; run `unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY` first |
| Whisper outputs traditional Chinese | Always use `OpenCC("t2s")` for simplified Chinese |
| Whisper can't find model | Check `MODEL_PATH` points to the local `faster-whisper-base` directory |
| Video is silent (audio.mp3 < 10KB) | Some Douyin videos have no sound track |

## Notes

- **No cookies needed**: CDN URL from browser `<video>` element is publicly accessible with Referer header
- **Fully offline STT**: Whisper model runs locally, no API calls needed
- **Model location**: `/mnt/c/Users/admin/Desktop/Hermes/model/faster-whisper-base` (git clone from `Systran/faster-whisper-base`)
- **Browser approach**: Browserbase loads Douyin page, extract CDN URL from DOM
- **Simplified Chinese**: OpenCC converts Whisper's traditional output to simplified automatically
