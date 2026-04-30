---
name: transcode
description: Ad-hoc transcode of a video file or folder — pick codec, container, resolution, framerate, and rate-control on the spot. Uses the hardware encoder detected by profile-system when available, falls back to libx264/libx265. Use when the user says "transcode this", "convert these to h.265", "change the framerate to 30", "make these smaller", or "VBR to CBR".
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(find *), Bash(jq *), Bash(stat *), Bash(basename *), Bash(dirname *), Bash(realpath *), Read, Write
---

# Transcode

Generic, on-demand transcoding. Unlike `render-with-profile` (which uses a saved preset), this skill takes parameters per invocation. Use it for one-offs, format conversions, framerate changes, or fixes (VBR → CBR).

## Procedure

### 1. Resolve system profile

```bash
DATA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/claude-media-plugins/video-editing"
SYS_FILE="$DATA_DIR/system-profile.json"
test -f "$SYS_FILE" && HAVE_SYS=1 || HAVE_SYS=0
```

If missing, warn the user that the skill will fall back to libx264/libx265 (slower, CPU-only) and offer to run `profile-system`.

### 2. Inputs

Ask if not provided:

| Field | Example |
|-------|---------|
| Source | file or folder |
| Codec | h264 / hevc / av1 / copy (stream-copy) |
| Container | mp4 / mkv / mov |
| Resolution | source / 1920x1080 / 1280x720 |
| Framerate | source / 24 / 30 / 60 |
| Rate control | crf / qp / cbr / vbr — with value |
| Output dir | default: sibling `transcoded/` folder |

### 3. Pick encoder

If `HAVE_SYS=1`, read `.encoders.<codec>.preferred` from `system-profile.json`. Otherwise:

| Codec | Software fallback |
|-------|-------------------|
| h264 | libx264 |
| hevc | libx265 |
| av1  | libsvtav1 |

For `codec=copy`, skip encoder selection — just `-c:v copy -c:a copy` (useful for container changes).

### 4. Build the ffmpeg command

VAAPI hardware path:

```bash
ffmpeg -hide_banner -i "$IN" \
  -vaapi_device "$RENDER_NODE" \
  -vf 'format=nv12,hwupload[,scale_vaapi=...]' \
  -c:v "$ENCODER" -qp "$QP" \
  -r "$FPS" \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  "$OUT"
```

Software path:

```bash
ffmpeg -hide_banner -i "$IN" \
  [-vf scale=...] -r "$FPS" \
  -c:v "$ENCODER" -crf "$CRF" -preset medium \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  "$OUT"
```

For CBR: `-c:v libx264 -b:v <rate> -minrate <rate> -maxrate <rate> -bufsize <2*rate>`.

Show the resolved command before running. Ask for confirmation when transcoding more than 1 file.

### 5. Loop and report

```bash
mkdir -p "$OUT_DIR"
for in in <inputs>; do
  out="$OUT_DIR/$(basename "${in%.*}").$CONTAINER"
  ffmpeg -hide_banner -y -i "$in" <args> "$out"
  echo "$in -> $out ($(stat -c%s "$out") bytes)"
done
```

Originals are never modified. Summarize input vs. output total bytes at the end.
