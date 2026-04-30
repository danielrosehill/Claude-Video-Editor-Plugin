---
name: burn-subtitles
description: Generate subtitles (whisper.cpp / faster-whisper) for a video and either save the SRT alongside or burn them into the picture. Use when the user says "subtitle this", "add captions", "burn subtitles", "transcribe and caption this video", "generate SRT for this". Accepts an existing SRT instead of regenerating. Picks the user's preferred whisper backend from preferences.json (set via onboard).
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(jq *), Bash(realpath *), Bash(basename *), Bash(dirname *), Bash(ffmpeg *), Bash(ffprobe *), Bash(*/whisper-cli *), Bash(whisper-cli *), Bash(whisper *), Bash(*/main *), Bash(uv *), Read, Write
---

# Burn Subtitles

Two related operations gated on a single flag:

1. **Generate** — transcribe a video to SRT (saved alongside as `<basename>.srt`).
2. **Burn** — composite the SRT into the picture as styled subtitles.

Default behaviour: generate the SRT and ask the user whether to also burn it. SRT-only is the safer default — burned subtitles are unrecoverable without the source.

## Procedure

### 1. Resolve backend

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS="$DATA_DIR/preferences.json"
VENV=$(jq -r '.python_venv // empty' "$PREFS" 2>/dev/null)
BACKEND=$(jq -r '.subtitle_backend // empty' "$PREFS" 2>/dev/null)
MODEL=$(jq -r '.subtitle_model // "base"' "$PREFS" 2>/dev/null)

case "$BACKEND" in
  faster-whisper)
    if [ -x "$VENV/bin/faster-whisper-cli" ]; then
      WHISPER="$VENV/bin/faster-whisper-cli"
    elif [ -x "$VENV/bin/python" ]; then
      WHISPER="$VENV/bin/python -m faster_whisper"
    fi
    ;;
  whisper.cpp)
    WHISPER=$(command -v whisper-cli || command -v main)
    ;;
  *)
    # Auto-detect
    if [ -x "$VENV/bin/faster-whisper-cli" ]; then WHISPER="$VENV/bin/faster-whisper-cli"; BACKEND=faster-whisper
    elif command -v whisper-cli >/dev/null; then WHISPER=whisper-cli; BACKEND=whisper.cpp
    else
      echo "No whisper backend found. Run deps-setup and choose faster-whisper or whisper.cpp, then onboard."
      exit 1
    fi
    ;;
esac
```

### 2. Inputs

| Field | Default |
|-------|---------|
| Source | required (video) |
| SRT mode | `generate` (default) / `existing <path>` |
| Burn? | `false` (default — save SRT alongside; ask before burning) |
| Model | from `preferences.subtitle_model` (default `base` for whisper.cpp, `small` for faster-whisper) |
| Language | auto-detect; user can force (`en`, `he`, etc.) |
| Style | from `preferences.subtitle_style` (font, size, primary colour, outline, position); else ffmpeg `subtitles` defaults |
| Output | `<basename>.subs.<ext>` if burning; SRT always saved as `<basename>.srt` |

### 3. Generate the SRT (if mode == generate)

#### faster-whisper

```bash
"$VENV/bin/python" -m faster_whisper \
  --model "$MODEL" \
  --output_format srt \
  --output_dir "$(dirname "$SRC")" \
  ${LANG:+--language "$LANG"} \
  "$SRC"
```

#### whisper.cpp

whisper.cpp wants 16 kHz mono WAV. Extract first:

```bash
WAV=$(mktemp --suffix=.wav)
ffmpeg -hide_banner -y -i "$SRC" -ac 1 -ar 16000 -vn "$WAV"
"$WHISPER" -m "$MODEL_PATH" -f "$WAV" --output-srt --output-file "${SRC%.*}"
rm "$WAV"
```

`$MODEL_PATH` is the GGML model file path recorded in `preferences.subtitle_model_path` by `onboard` (whisper.cpp ships models as files, not names).

### 4. Burn (only if requested)

ffmpeg's `subtitles` filter renders SRT styling via libass. Style is passed inline via `force_style`:

```bash
STYLE="${SUB_STYLE:-FontName=Inter,FontSize=24,PrimaryColour=&HFFFFFF,OutlineColour=&H000000,BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=40}"

# Escape the SRT path for filtergraph (single-quote, escape colons & backslashes)
ESC_SRT=$(printf %s "$SRT" | sed -e 's/\\/\\\\/g' -e "s/'/\\\\'/g" -e 's/:/\\:/g')

ffmpeg -hide_banner -y -i "$SRC" \
  -vf "subtitles='${ESC_SRT}':force_style='${STYLE}'" \
  -c:v libx264 -crf 18 -preset medium \
  -c:a copy \
  -movflags +faststart \
  "$OUT"
```

Re-encoding video is required (subtitles modify pixels). If the user has a render profile selected, prefer the GPU encoder from `system-profile.json` — but burned subtitles need libass-compatible YUV, and VAAPI hwupload chains complicate things. For a first pass, libx264 at CRF 18 is the safe choice. Mention this to the user.

### 5. Soft-subtitle alternative (no burn)

If the user wants embedded but selectable subtitles (recommended), mux the SRT as a track:

```bash
ffmpeg -hide_banner -y -i "$SRC" -i "$SRT" \
  -map 0 -map 1 \
  -c copy -c:s mov_text \
  -metadata:s:s:0 language="${LANG:-eng}" \
  "${SRC%.*}.subbed.mp4"
```

This is reversible, smaller, and works on most players. Prefer this unless the user explicitly wants burned-in.

### 6. Report

```
Source : path/to/in.mp4
SRT    : path/to/in.srt        (12.4 KB, 142 cues)
Mode   : burned | soft-muxed | srt-only
Output : path/to/in.subs.mp4
Backend: faster-whisper / small / en
```

## Notes

- For karaoke-style timing with per-word highlighting, this skill produces sentence-level SRT only. Per-word ASS timing belongs in `karaoke-video`.
- If the source already has a soft-subbed track, ask before regenerating — the existing track is probably better than a fresh whisper pass.
- `clean-transcription` pairs with this skill: run after generation to strip filler words and fix recurring mistranscriptions before burning.
