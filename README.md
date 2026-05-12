# Hermes Skills Collection

Custom skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent).

## Available Skills

### douyin-video-summarizer

Summarize Douyin (抖音) video content from share links.

**Pipeline:** parse link → extract CDN URL → extract audio → speech-to-text (whisper + OpenCC 繁转简) → AI summary

**Dependencies:** ffmpeg, faster-whisper, opencc-python-reimplemented, Browserbase browser

## Installation

```bash
git clone https://github.com/1625890439-create/hermes-skills.git
cp -r hermes-skills/douyin-video-summarizer ~/.hermes/skills/media/
```
