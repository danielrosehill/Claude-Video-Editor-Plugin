---
name: mlt-render
description: Render an MLT XML timeline (Kdenlive `.kdenlive` project or raw `.mlt`) to a deliverable using the `melt` CLI. Use when the user says "render the timeline", "render this kdenlive project to mp4", "melt render", "headless render". Supports an optional named render profile from the data store, otherwise asks for codec/bitrate/resolution/fps inline.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(command *), Bash(melt *), Bash(ffmpeg *), Bash(ffprobe *), Read, Write
---

# MLT Render

Headless render of an MLT-format timeline (Kdenlive's native format) using the `melt` CLI.

## Procedure

### 1. Verify melt

```bash
command -v melt >/dev/null || { echo "melt not installed. Try: sudo apt install melt  (or sudo apt install mlt-tools on some distros)"; exit 1; }
```

### 2. Resolve inputs

| Field | Default |
|-------|---------|
| Source | required: `.kdenlive` or `.mlt` file |
| Output | default: `<project>/renders/<basename>_<timestamp>.mp4` |
| Profile name | optional — load from `render-profiles.json` |
| Inline params | codec, bitrate (or qp), resolution, fps — only when no profile chosen |

```bash
SRC="$(realpath "$INPUT")"
test -f "$SRC" || { echo "Not a file: $SRC"; exit 1; }
case "$SRC" in
  *.kdenlive|*.mlt) ;;
  *) echo "Not an MLT-format file: $SRC"; exit 1 ;;
esac
```

### 3. Build the melt command

`melt` renders via a `consumer` chain. Two modes:

**3a. With a render profile** — load codec/QP/resolution/fps from `render-profiles.json` (same shape used by `render-with-profile`). Build a melt avformat consumer:

```bash
PROFILES="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/render-profiles.json"
PROF=$(jq -r --arg n "$PROFILE_NAME" '.[$n] // empty' "$PROFILES")
[ -z "$PROF" ] && { echo "Profile not found: $PROFILE_NAME"; exit 1; }

VCODEC=$(jq -r '.vcodec' <<<"$PROF")
ACODEC=$(jq -r '.acodec // "aac"' <<<"$PROF")
WIDTH=$(jq -r '.width' <<<"$PROF")
HEIGHT=$(jq -r '.height' <<<"$PROF")
FPS=$(jq -r '.fps' <<<"$PROF")
QP=$(jq -r '.qp // empty' <<<"$PROF")
VB=$(jq -r '.vbitrate // empty' <<<"$PROF")
AB=$(jq -r '.abitrate // "192k"' <<<"$PROF")
```

Compose:

```bash
ARGS=( "$SRC" -consumer avformat:"$OUT" )
ARGS+=( vcodec="$VCODEC" acodec="$ACODEC" )
ARGS+=( width="$WIDTH" height="$HEIGHT" frame_rate_num="$FPS" frame_rate_den=1 )
[ -n "$QP" ] && ARGS+=( qp="$QP" )
[ -n "$VB" ] && ARGS+=( vb="$VB" )
ARGS+=( ab="$AB" )
```

**3b. Inline params** — prompt the user for the missing fields and build the same consumer chain.

> **Hardware encoders.** `melt` uses libavformat under the hood, so VAAPI/NVENC encoder names match ffmpeg (`h264_vaapi`, `hevc_nvenc`, etc). However, melt does not set up the VAAPI hwupload chain automatically — for hardware-encoded renders the safer path is **render to an intermediate (e.g. ProRes or h264 sw) via melt, then transcode to the GPU-encoded deliverable via `render-with-profile`**. The skill should warn the user and offer this two-step path when a VAAPI/NVENC profile is selected.

### 4. Render

```bash
mkdir -p "$(dirname "$OUT")"
echo "Running: melt ${ARGS[*]}"
melt "${ARGS[@]}"
```

Show melt's progress output live.

### 5. Verify

```bash
ffprobe -v error -show_entries format=duration,size -of default=nw=1 "$OUT"
```

Print: output path, file size, duration, codec.

### 6. Confirm

Print the deliverable path. Offer to follow up with `transcode` (e.g. to make a YouTube-friendly version) or `audio-analysis` (to check loudness).

## Notes

- For Kdenlive projects with proxy clips: melt resolves them via the project file's proxy paths. If the proxies have moved, the render will fail — fix the project paths in Kdenlive first.
- Render time is dominated by the most complex effect on the timeline; melt does not parallelize across CPU cores within a single track.
- Output container is inferred from the file extension (`.mp4`, `.mkv`, `.mov`).
