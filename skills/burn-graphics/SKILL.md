---
name: burn-graphics
description: Overlay graphics onto a video — lower thirds, watermarks/logos, image overlays, or text titles — at specified timestamps. Use when the user says "add a watermark", "put my logo in the corner", "burn in a lower third", "add a title card", "overlay this PNG at 0:30". Multiple overlays in one pass; each can have its own time range and position.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(jq *), Bash(realpath *), Bash(basename *), Bash(dirname *), Bash(ffmpeg *), Bash(ffprobe *), Read, Write
---

# Burn Graphics

Compose one or more graphic overlays into a video in a single ffmpeg pass. Inputs can be images (PNG with alpha, JPEG), text titles (rendered via `drawtext`), or pre-built lower-third image cards.

This skill superseds `commands/add-watermark.md` (legacy single-image watermark only).

## Procedure

### 1. Input model

The skill takes a list of overlay specs. The user can supply them inline, or via a JSON file:

```json
{
  "overlays": [
    {
      "type": "image",
      "path": "/abs/path/logo.png",
      "position": "top-right",
      "margin": 24,
      "scale": 0.15,
      "start": "0",
      "end": "end",
      "opacity": 0.8
    },
    {
      "type": "lower-third",
      "image": "/abs/path/lower-third-card.png",
      "position": "bottom-left",
      "margin_x": 60,
      "margin_y": 80,
      "start": "00:00:05",
      "end": "00:00:12",
      "fade": 0.5
    },
    {
      "type": "text",
      "text": "Daniel Rosehill",
      "subtext": "Technology Communications",
      "position": "bottom-left",
      "font": "Inter",
      "size": 48,
      "color": "white",
      "box": true,
      "box_color": "black@0.5",
      "start": "00:00:05",
      "end": "00:00:12"
    }
  ]
}
```

Position keywords (top-left, top-right, bottom-left, bottom-right, top-center, bottom-center, center) resolve to ffmpeg expressions during graph build.

### 2. Resolve positions

```
top-left      : x=margin                          : y=margin
top-right     : x=W-w-margin                      : y=margin
bottom-left   : x=margin                          : y=H-h-margin
bottom-right  : x=W-w-margin                      : y=H-h-margin
top-center    : x=(W-w)/2                         : y=margin
bottom-center : x=(W-w)/2                         : y=H-h-margin
center        : x=(W-w)/2                         : y=(H-h)/2
```

(`W,H` = main video dims; `w,h` = overlay dims.)

### 3. Build the filtergraph

Each overlay becomes one `overlay` segment, optionally preceded by `scale`, `format=rgba`, and `colorchannelmixer=aa=<opacity>`. Time-gating uses `enable='between(t,$start,$end)'`.

Example: logo top-right, 80% opacity, full duration + lower-third bottom-left from 5–12s with a 0.5s fade in/out:

```bash
FILTER="
[0:v]format=yuva420p[base];
[1:v]scale=iw*0.15:-1,format=rgba,colorchannelmixer=aa=0.8[ov1];
[base][ov1]overlay=x=W-w-24:y=24:enable='between(t,0,1e9)'[v1];
[2:v]format=rgba,fade=in:st=5:d=0.5:alpha=1,fade=out:st=11.5:d=0.5:alpha=1[ov2];
[v1][ov2]overlay=x=60:y=H-h-80:enable='between(t,5,12)'[vout]
"

ffmpeg -hide_banner -y \
  -i "$SRC" \
  -i "$LOGO" \
  -i "$LT_IMG" \
  -filter_complex "$FILTER" \
  -map "[vout]" -map 0:a? \
  -c:v libx264 -crf 18 -preset medium \
  -c:a copy \
  -movflags +faststart \
  "$OUT"
```

For text overlays, replace the image input + `[N:v]format=...` chain with a `drawtext` filter inserted between `[base]` and `[v1]`:

```
[base]drawtext=fontfile='/abs/path/Inter.ttf':text='Daniel Rosehill':fontsize=48:fontcolor=white:box=1:boxcolor=black@0.5:boxborderw=12:x=60:y=H-th-80:enable='between(t,5,12)'[v1];
```

(For multi-line text overlays, stack multiple `drawtext` filters with offset y values, or render the text to a transparent PNG offline and overlay as an image.)

### 4. Watermark shortcut

For the common case ("logo top-right, full duration"), the skill accepts a single-overlay shortcut:

```
src=in.mp4 watermark=logo.png position=top-right opacity=0.7 scale=0.12
```

→ builds the equivalent single-overlay spec and runs the same pipeline.

### 5. Lower-third helper (optional)

If the user wants a lower third but doesn't have a card image, offer to generate one with `drawtext` over a translucent rectangle:

```
[base]drawbox=x=40:y=H-160:w=600:h=120:color=black@0.6:t=fill:enable='between(t,5,12)',
       drawtext=fontfile=...:text='Daniel Rosehill':fontsize=44:fontcolor=white:x=64:y=H-130:enable='between(t,5,12)',
       drawtext=fontfile=...:text='Technology Communications':fontsize=28:fontcolor=white@0.85:x=64:y=H-80:enable='between(t,5,12)'[v1];
```

This avoids an external image asset and keeps the skill self-contained.

### 6. Encoder choice

Burning graphics requires re-encoding video (pixels change). Default to libx264 CRF 18 — predictable and matches `burn-subtitles`. If the user has a saved render profile, offer to use it, but warn that VAAPI hwupload chains complicate filtergraphs and may need a CPU fallback.

### 7. Report

```
Source : path/to/in.mp4
Output : path/to/in.graphics.mp4
Overlays applied:
  1. image  logo.png            top-right    full duration  opacity=0.8
  2. lower  lower-third-card.png bottom-left  00:05–00:12    fade=0.5s
  3. text   "Daniel Rosehill"    bottom-left  00:05–00:12
```

## Notes

- Time ranges accept `HH:MM:SS`, `S` (seconds), or `end` for "to end of source".
- For animated lower thirds (slide-in, dissolve), pre-render the lower third as a short transparent-background video (e.g. via After Effects or `editly`) and overlay as type `video` (filter chain identical, plus `[N:v]setpts=PTS-STARTPTS+5/TB` to time-offset the asset).
- This skill replaces `commands/add-watermark.md`. Once verified, delete that command.
