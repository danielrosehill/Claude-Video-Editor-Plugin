---
name: audio-to-music-video
description: Turn an audio file into a music video by compositing a waveform / spectrum / circular-CQT animation over cover art or a background. Templates (waveform-bottom, spectrum-bars, circular-cqt-cover, etc.) are saved in preferences.json so the user can pick by name. Use when the user says "make a music video from this audio", "audio to video with waveform", "render this song with a visualizer", "spectrum animation over cover art".
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(jq *), Bash(realpath *), Bash(basename *), Bash(dirname *), Bash(ffmpeg *), Bash(ffprobe *), Bash(awk *), Read, Write
---

# Audio to Music Video

Single-pass ffmpeg synthesis of a music video from an audio file plus a chosen template. The audio is pristine (stream-copied where the container allows) and the visual is built entirely with ffmpeg's `-filter_complex` graph — no external editor needed.

## Templates

Templates are stored in `preferences.json` under `audio_to_video_templates` and selected by name. Each template defines: visualizer type, layout, colour palette, optional cover-art treatment, and resolution. Built-in defaults ship in this skill's prose; users add their own via `onboard` or by editing `preferences.json` directly.

### Built-in templates

| Name | Layout | Visualizer |
|---|---|---|
| `waveform-bottom` | full-frame blurred cover art bg + crisp cover art centered + linear waveform strip across bottom 15% | `showwaves` (mode=cline) |
| `spectrum-bars` | solid colour bg + cover art top-center + spectrum bars bottom 40% | `showfreqs` (mode=bar, ascale=log) |
| `circular-cqt-cover` | blurred cover bg + circular spectrum (showcqt) ring around centered cover | `showcqt` |
| `vector-scope` | dark bg + cover art top-left + Lissajous vector scope center | `avectorscope` |
| `volume-meter` | minimal — solid bg + cover art + horizontal volume meter | `showvolume` |

A template entry:

```json
{
  "name": "waveform-bottom",
  "resolution": "1920x1080",
  "fps": 30,
  "bg": { "type": "cover-blur", "blur": 40, "darken": 0.3 },
  "cover": { "size": "60%", "position": "center", "shadow": true },
  "viz": {
    "filter": "showwaves",
    "mode": "cline",
    "size": "1920x162",
    "color": "white",
    "position": "bottom",
    "opacity": 0.9
  }
}
```

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Audio | required (`.mp3`, `.wav`, `.flac`, `.m4a`) |
| Cover art | optional path; if omitted and the audio has embedded cover art, extract it; else use `bg.solid` colour |
| Template | name from `preferences.audio_to_video_templates`; default `waveform-bottom` |
| Title / Artist | optional `drawtext` overlay |
| Output | `<basename>.mv.mp4` |

### 2. Resolve cover art

If not supplied:

```bash
ffmpeg -hide_banner -y -i "$AUDIO" -an -vcodec copy "$WORK/cover.jpg" 2>/dev/null \
  || COVER_MISSING=1
```

If still missing, use a plain colour from `bg.solid` (or fall back to `#111111`).

### 3. Build the filtergraph (template = `waveform-bottom`)

```bash
WIDTH=1920; HEIGHT=1080; FPS=30
COVER_W=$((WIDTH * 60 / 100))   # 60% width centered

ffmpeg -hide_banner -y \
  -loop 1 -i "$COVER" \
  -i "$AUDIO" \
  -filter_complex "
    [0:v]scale=${WIDTH}:${HEIGHT}:force_original_aspect_ratio=increase,
         crop=${WIDTH}:${HEIGHT},
         boxblur=40:1,
         eq=brightness=-0.3[bg];
    [0:v]scale=${COVER_W}:-1[cov];
    [bg][cov]overlay=x=(W-w)/2:y=(H-h)/2[bgcov];
    [1:a]showwaves=mode=cline:s=${WIDTH}x162:colors=white:rate=${FPS}[wf];
    [bgcov][wf]overlay=x=0:y=H-h-40:format=auto[v]
  " \
  -map "[v]" -map 1:a \
  -c:v libx264 -crf 20 -preset medium -pix_fmt yuv420p \
  -c:a aac -b:a 192k \
  -shortest \
  -movflags +faststart \
  "$OUT"
```

### 4. Spectrum-bars template

Replace the visualizer chain:

```
[1:a]showfreqs=s=${WIDTH}x432:mode=bar:ascale=log:fscale=log:colors=cyan|magenta:rate=${FPS}[viz];
[bgcov][viz]overlay=x=0:y=H-h-60:format=auto[v]
```

### 5. Circular CQT template

```
[1:a]showcqt=s=${WIDTH}x${HEIGHT}:fps=${FPS}:bar_h=300:axis_h=0:cscheme=1|0.5|0|0|0.5|1[viz];
[bgcov][viz]overlay=0:0:format=auto:enable='gte(t,0)'[v]
```

`showcqt` is the most musical-looking visualizer — it maps frequency on a constant-Q logarithmic scale, so each octave gets equal screen real estate. Heavier on CPU than `showwaves`.

### 6. Title / artist overlay (optional)

Add a `drawtext` after the visualizer overlay:

```
[v]drawtext=fontfile='/abs/path/Inter-Bold.ttf':text='${TITLE}':fontsize=64:fontcolor=white:x=80:y=80:shadowcolor=black@0.6:shadowx=2:shadowy=2,
    drawtext=fontfile='/abs/path/Inter.ttf':text='${ARTIST}':fontsize=36:fontcolor=white@0.85:x=80:y=160:shadowcolor=black@0.6:shadowx=2:shadowy=2[v]
```

### 7. Encoder choice

For stable colour and broad compatibility, default to libx264 + `pix_fmt yuv420p`. The visualizer filters generate full-range RGB — without the `pix_fmt` clamp some players show washed-out colours.

If the user has a render profile, offer to use it, but warn that the filtergraph already runs on CPU and a GPU encoder won't speed up rendering by much.

### 8. Adding a custom template

If the user wants to register their own (different colours, different cover crop, different visualizer):

```bash
jq '.audio_to_video_templates += [$tpl]' --argjson tpl "$NEW_TPL" "$PREFS" \
  > "$PREFS.tmp" && mv "$PREFS.tmp" "$PREFS"
```

`onboard` should be extended to walk new users through registering at least one template.

### 9. Report

```
Audio    : path/to/song.mp3 (4:32, 320 kbps)
Cover    : embedded → extracted (1400x1400)
Template : waveform-bottom (1920x1080 @ 30fps)
Output   : path/to/song.mv.mp4 (28 MB, 4:32)
```

## Notes

- Render time is dominated by the visualizer filter, not the encoder. `showcqt` is ~5–10× slower than `showwaves`.
- For very long audio (album-length), encode at CRF 22 to keep file size reasonable.
- This skill is the audio-only complement to `karaoke-video` — combine them for a "make a karaoke video from this MP3" flow: run `audio-to-music-video` to synthesise a base video, then `karaoke-video` to add the lyric overlay (skip the demucs step in that case, since the audio is already stem-friendly or already instrumental).
- For animated cover-art scenes (Ken Burns zoom, parallax), pre-render the background as a short looped video and substitute it for the `loop 1 -i $COVER` input.
