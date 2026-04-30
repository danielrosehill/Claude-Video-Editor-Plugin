---
name: profile-system
description: Detect the user's GPU and ffmpeg hardware encoders (NVENC, VAAPI, QSV, AMF) and persist the result to system-profile.json. Run this once per machine before using render-profile or transcode skills. Use when the user says "profile my system", "detect my GPU", "what encoders do I have", or before any render skill that complains about a missing profile.
disable-model-invocation: false
allowed-tools: Bash(lspci *), Bash(vainfo *), Bash(nvidia-smi *), Bash(ffmpeg *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(jq *), Bash(date *), Bash(grep *), Bash(awk *), Bash(command *), Read, Write
---

# Profile System

Detect the host's GPU and ffmpeg encoder capabilities so other skills (`create-render-profile`, `render-with-profile`, `transcode`) can pick the right hardware encoder instead of falling back to CPU x264.

## Procedure

### 1. Set up paths

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PROFILE_FILE="$DATA_DIR/system-profile.json"
mkdir -p "$DATA_DIR"
```

If `$PROFILE_FILE` already exists, show the user the current contents and ask whether to re-detect.

### 2. Detect GPU vendor

```bash
lspci | grep -iE 'vga|3d|2d'
```

Identify NVIDIA / AMD / Intel from the output. Also probe:

```bash
command -v nvidia-smi >/dev/null && nvidia-smi --query-gpu=name --format=csv,noheader
command -v vainfo     >/dev/null && vainfo 2>/dev/null | grep -E 'Driver version|VAProfile' | head -20
ls /dev/dri/ 2>/dev/null
```

Record the **VAAPI render node** (typically `/dev/dri/renderD128`) — needed for AMD/Intel hardware encode.

### 3. Probe ffmpeg encoders

```bash
ffmpeg -hide_banner -encoders 2>/dev/null | grep -iE 'nvenc|vaapi|qsv|amf|videotoolbox' || true
```

Bucket what's available per codec:

| Codec | Preferred order |
|-------|-----------------|
| H.264 | `h264_nvenc` (NVIDIA) → `h264_vaapi` (AMD/Intel on Linux) → `h264_qsv` (Intel) → `h264_amf` (AMD) → `libx264` |
| HEVC  | `hevc_nvenc` → `hevc_vaapi` → `hevc_qsv` → `hevc_amf` → `libx265` |
| AV1   | `av1_nvenc` → `av1_vaapi` → `av1_qsv` → `av1_amf` → `libsvtav1` → `libaom-av1` |

For AMD on Linux, **prefer VAAPI over AMF** — Mesa's VAAPI is more reliable and produces smaller files at the same quality than AMF in most ffmpeg builds.

### 4. Smoke-test the hardware encoder (optional but recommended)

For the chosen H.264 encoder, run a 1-second synthetic test:

```bash
ffmpeg -hide_banner -f lavfi -i testsrc=duration=1:size=1280x720:rate=30 \
  -vaapi_device /dev/dri/renderD128 -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi -f null - 2>&1 | tail -5
```

(Adjust the command per encoder family.) Record `working: true|false` per encoder. A listed-but-broken encoder is common on AMD setups missing the right Mesa packages.

### 5. Write `system-profile.json`

```json
{
  "detected": "<ISO-8601>",
  "gpu": {
    "vendor": "AMD|NVIDIA|Intel|unknown",
    "model": "<lspci string>",
    "render_node": "/dev/dri/renderD128"
  },
  "encoders": {
    "h264": { "preferred": "h264_vaapi", "available": ["h264_vaapi", "libx264"], "working": ["h264_vaapi", "libx264"] },
    "hevc": { "preferred": "hevc_vaapi", "available": ["hevc_vaapi", "libx265"], "working": ["hevc_vaapi", "libx265"] },
    "av1":  { "preferred": "libsvtav1",  "available": ["libsvtav1"],              "working": ["libsvtav1"] }
  },
  "ffmpeg_version": "<ffmpeg -version | head -1>"
}
```

Write atomically: `jq` (or `cat <<EOF`) to `$PROFILE_FILE.tmp`, then `mv`.

### 6. Report + offer next step

Print a short summary: vendor, render node, preferred H.264 / HEVC / AV1 encoders. Suggest the user run `create-render-profile` next.
