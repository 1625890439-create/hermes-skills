# WSL Proxy Pitfalls — Video Pipeline

## Problem
WSL2 cannot reach Windows Clash proxy (17891 has no LAN mode). But residual env vars from bash sessions cause tools to try routing through it.

## Affected Tools

### curl — resolving Douyin short links
WSL curl can't resolve `v.douyin.com` when proxy env vars are set.

**Workaround:** Use Windows curl via cmd.exe:
```bash
cmd.exe /c "curl -sI -L https://v.douyin.com/xxxxx/" 2>&1 | grep -i "location"
```

### ffmpeg — extracting audio from CDN URLs
ffmpeg inherits env vars and tries to route through 172.30.160.1:17891 (Windows host IP), which times out.

**Workaround:** Unset all proxy vars before running:
```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY all_proxy ALL_PROXY
ffmpeg -referer "https://www.douyin.com/" ...
```

### yt-dlp — downloading videos
Same proxy issue. Also, Douyin requires cookies that are locked by Edge browser.

**Workaround:** Use the browser CDN URL extraction method instead of yt-dlp for Douyin.

## Key Learning
The user's Clash proxy is only accessible from Windows native processes, not from WSL. When the user says "proxy is on", it means Windows-side access works. For WSL-side network access, either:
1. Use `cmd.exe` to run Windows-native tools
2. Unset proxy env vars and rely on direct network access
3. Use browser tool (Browserbase) which has its own network path
