---
name: onboard
description: First-run setup for the video-editing plugin — collect per-user preferences (talking-head EQ preset, default loudness target, default render target) and write them to preferences.json. Use when the user says "onboard the video plugin", "set up my preferences", "register my voice EQ", "configure video editing", or when another skill (e.g. talking-head-eq) reports a missing preset.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(cp *), Bash(jq *), Bash(date *), Bash(realpath *), Bash(ls *), Bash(basename *), Bash(file *), Bash(head *), Read, Write
---

# Onboard

Walks the user through registering per-user preferences for the video-editing plugin. Idempotent — shows current values first, lets the user update or skip per field.

The plugin is publicly distributed — nothing user-specific is bundled. Everything collected here lives only on the user's machine, in:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/preferences.json
```

## Procedure

### 1. Resolve config

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS_FILE="$DATA_DIR/preferences.json"
PRESETS_DIR="$DATA_DIR/presets"
mkdir -p "$DATA_DIR" "$PRESETS_DIR"
test -f "$PREFS_FILE" && CURRENT=$(cat "$PREFS_FILE") || CURRENT='{}'
```

Show the current `$CURRENT` summary.

### 2. Walk the fields

For each, show the current value (if any), and ask: keep / update / skip.

#### a) Talking-head EQ preset

Ask the user for a path to their preset. Auto-detect format:

```bash
case "$EXT" in
  json)
    if jq -e '.. | objects | select(has("bands"))' "$PATH_" >/dev/null 2>&1; then
      # has a "bands" array somewhere — could be either format
      if jq -e '.bands[0] | has("frequency") and has("q") and has("gain")' "$PATH_" >/dev/null 2>&1; then
        FORMAT="band-list-json"
      else
        FORMAT="easyeffects-json"
      fi
    else
      FORMAT="easyeffects-json"  # assume EE; user can correct
    fi
    ;;
  txt|conf|ini) FORMAT="ffmpeg-string" ;;
  *)            FORMAT="ffmpeg-string" ;;
esac
```

Confirm the detected format with the user. Then copy into the plugin's presets dir:

```bash
DEST="$PRESETS_DIR/talking-head-eq.$EXT"
cp "$PATH_" "$DEST"
```

Record in preferences:

```json
{
  "talking_head_eq": {
    "path": "/abs/path/to/presets/talking-head-eq.json",
    "format": "easyeffects-json|ffmpeg-string|band-list-json",
    "registered": "<ISO-8601>"
  }
}
```

#### b) Default loudness target

Offer a quick pick: `youtube` (-14 / -1) / `spotify` (-14 / -1) / `apple-music` (-16 / -1) / `podcast` (-16 / -1) / `broadcast` (-23 / -2) / `custom`.

```json
{
  "loudness_target": { "preset": "youtube", "integrated_lufs": -14, "true_peak_dbtp": -1 }
}
```

#### c) Default render profile

If `render-profiles.json` already exists, list saved profiles and ask which is the default. Otherwise mark as unset and suggest `create-render-profile` later.

```json
{ "default_render_profile": "youtube-1080p" }
```

#### d) Python venv (managed by deps-setup)

Pre-record the path even before the venv exists, so `deps-setup` can find it:

```json
{ "python_venv": "/abs/.../claude-plugins/video-editing/venv" }
```

#### e) Subtitle backend

Used by `burn-subtitles`, `generate-deliverables`, and `karaoke-video` (fallback path). Pick one:

- `faster-whisper` — Python, fast on CPU, GPU optional. Installed via `deps-setup`.
- `whisper.cpp` — pure C++, no Python deps. User installs separately; record the binary path and a GGML model file path.

```json
{
  "subtitle_backend": "faster-whisper",
  "subtitle_model": "small",
  "subtitle_model_path": "/abs/path/to/ggml-small.en.bin",
  "subtitle_style": {
    "font": "Inter",
    "size": 24,
    "primary_colour": "&HFFFFFF",
    "outline_colour": "&H000000",
    "alignment": 2,
    "margin_v": 40
  }
}
```

`subtitle_model_path` is only needed for whisper.cpp (which loads model files by path, not by name).

#### f) Audio-to-video templates

Used by `audio-to-music-video`. Seed with a built-in default if the user has none registered:

```json
{
  "audio_to_video_templates": [
    {
      "name": "waveform-bottom",
      "resolution": "1920x1080",
      "fps": 30,
      "bg": { "type": "cover-blur", "blur": 40, "darken": 0.3 },
      "cover": { "size_pct": 60, "position": "center" },
      "viz": { "filter": "showwaves", "mode": "cline", "color": "white", "height_pct": 15 }
    }
  ],
  "default_audio_to_video_template": "waveform-bottom"
}
```

Offer to register additional templates inline; otherwise point the user at `audio-to-music-video`'s prose for the full list of presets.

### 3. Write atomically

```bash
echo "$NEW_PREFS" | jq . > "$PREFS_FILE.tmp" && mv "$PREFS_FILE.tmp" "$PREFS_FILE"
```

### 4. Report

Print the new file path and a one-line summary per field. Suggest `deps-setup` next if the user wants moviepy/auto-editor/etc.
