---
name: create-render-profile
description: Build a named, reusable render profile (codec + resolution + bitrate/QP + container) by iterating on a 30-second test clip until file size and quality are right. Use when the user says "create a render profile", "tune my YouTube preset", "I want a render preset for archival", or is frustrated with bloated GPU-encoded files. Saves to render-profiles.json for later use by render-with-profile and transcode.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(stat *), Bash(jq *), Bash(date *), Bash(realpath *), Bash(du *), Bash(awk *), Bash(numfmt *), Read, Write
---

# Create Render Profile

Walks the user through building a render profile interactively, with a tight feedback loop: render a short test clip, inspect file size + bitrate, tune, repeat.

This is the antidote to "GPU-encoded but bloated". The goal is a profile the user trusts — saved by name, reused across jobs.

## Procedure

### 1. Resolve the system profile

```bash
DATA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/claude-media-plugins/video-editing"
PROFILE_FILE="$DATA_DIR/system-profile.json"
PROFILES_FILE="$DATA_DIR/render-profiles.json"
test -f "$PROFILE_FILE" || { echo "No system profile — run profile-system first."; exit 1; }
```

Read preferred encoders and the VAAPI render node from `system-profile.json`.

### 2. Ask the user

| Field | Example | Default |
|-------|---------|---------|
| Profile name | `youtube-1080p` | required, slugify |
| Container | mp4 / mkv / mov | mp4 |
| Codec | h264 / hevc / av1 | hevc (best size/quality) |
| Resolution | 1920x1080 / 3840x2160 / source | source |
| Framerate | 30 / 60 / source | source |
| Quality target | "small file", "YouTube", "archival" | "YouTube" |
| Sample clip | path to a representative source file | required |

Map quality target → starting QP / bitrate (per encoder). Reasonable starting points:

- VAAPI HEVC: `-qp 23` (YouTube), `-qp 20` (archival), `-qp 28` (small file)
- VAAPI H.264: `-qp 22` (YouTube), `-qp 18` (archival), `-qp 27` (small file)
- libx264/libx265: `-crf 20` / `-crf 23` / `-crf 28`

### 3. Build the ffmpeg command

For AMD VAAPI HEVC at 1080p:

```bash
ffmpeg -hide_banner -ss 0 -t 30 -i "<sample>" \
  -vaapi_device "<render_node>" \
  -vf 'format=nv12,hwupload,scale_vaapi=w=1920:h=1080' \
  -c:v hevc_vaapi -qp <QP> \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  "<test_out>"
```

For software fallback (x265):

```bash
ffmpeg -hide_banner -ss 0 -t 30 -i "<sample>" \
  -vf scale=1920:1080 \
  -c:v libx265 -crf <CRF> -preset medium \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  "<test_out>"
```

Show the user the exact command before running.

### 4. Render the test clip + report

Write the test to `$DATA_DIR/test-renders/<profile-name>-<attempt>.<ext>`. After it completes:

```bash
size_bytes=$(stat -c%s "$test_out")
size_human=$(numfmt --to=iec --suffix=B "$size_bytes")
duration=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$test_out")
bitrate_kbps=$(awk -v s="$size_bytes" -v d="$duration" 'BEGIN { printf "%.0f", (s*8/1000)/d }')
```

Report:

```
Test render (attempt N):
  Profile: youtube-1080p (hevc_vaapi, 1080p, qp=23)
  File:    1080p-test-1.mp4 (28.4 MB, 30.0s, ~7700 kbps)
```

### 5. Iterate

Ask the user: **keep / tune / abandon**.

- **Tune**: ask which way (smaller file / better quality / different codec / different resolution). Adjust QP by ±2 or bitrate by ±20% and re-render. Loop. Show all attempts in a small table so the user sees the trade-off.
- **Abandon**: clean up the test renders, exit without saving.
- **Keep**: proceed to step 6.

### 6. Save the profile

Append to `render-profiles.json` (create if missing):

```json
{
  "profiles": {
    "youtube-1080p": {
      "created": "<ISO-8601>",
      "container": "mp4",
      "codec": "hevc",
      "encoder": "hevc_vaapi",
      "resolution": "1920x1080",
      "framerate": "source",
      "rate_control": { "mode": "qp", "value": 23 },
      "audio": { "codec": "aac", "bitrate": "192k" },
      "extra_args": ["-movflags", "+faststart"],
      "notes": "Tuned on <sample basename>: 28.4 MB / 30s / ~7700 kbps"
    }
  }
}
```

Write atomically (`.tmp` + `mv`).

### 7. Offer next step

Suggest `render-with-profile` to apply the new profile to a real folder. Mention `list-render-profiles` to review saved presets.
