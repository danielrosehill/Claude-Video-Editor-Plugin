---
name: talking-head-eq
description: Apply the user's saved voice/dialogue EQ preset to the audio of a video (or a standalone audio file) and re-mux. Use when the user says "EQ this talking head", "apply my voice preset", "clean up the audio for this video", or similar. Reads the preset from the per-user data store (registered via the onboard skill). Supports EasyEffects JSON, raw ffmpeg filter strings, and band-list JSON formats.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(cat *), Bash(awk *), Read, Write
---

# Talking-Head EQ

Apply a saved EQ chain to dialogue audio inside a video, then re-mux the corrected audio back into a copy of the video. Useful for podcasts, interviews, monologues — any "talking head" content where the same voice EQ is applied repeatedly.

The preset is **per-user**, registered via the `onboard` skill. This plugin ships no defaults — it works only after the user has provided a preset.

## Procedure

### 1. Resolve preset

```bash
DATA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/claude-media-plugins/video-editing"
PREFS_FILE="$DATA_DIR/preferences.json"
test -f "$PREFS_FILE" || { echo "No preferences. Run onboard first."; exit 1; }

PRESET_PATH=$(jq -r '.talking_head_eq.path // empty' "$PREFS_FILE")
PRESET_FORMAT=$(jq -r '.talking_head_eq.format // empty' "$PREFS_FILE")
test -n "$PRESET_PATH" || { echo "No talking-head EQ preset registered. Run onboard."; exit 1; }
test -f "$PRESET_PATH" || { echo "Registered preset path missing: $PRESET_PATH"; exit 1; }
```

### 2. Build the ffmpeg filter chain by format

#### a) `ffmpeg-string`

The preset file is a single-line ffmpeg `-af` string (no leading `-af`).

```bash
FILTER=$(cat "$PRESET_PATH")
```

Example file contents:

```
highpass=f=80,equalizer=f=200:t=q:w=1.0:g=-3,equalizer=f=3500:t=q:w=1.2:g=2,acompressor=threshold=-18dB:ratio=2.5:attack=10:release=120
```

#### b) `easyeffects-json`

EasyEffects exports a JSON tree. The `output` (or `input`) section carries an `equalizer` plugin with N bands, each having `frequency`, `q`, `gain`. Translate to ffmpeg `equalizer=f=<freq>:t=q:w=<q>:g=<gain>` and chain with commas.

```bash
FILTER=$(jq -r '
  (.output // .input)
  | (.. | objects | select(has("bands"))) as $eq
  | $eq.bands
  | to_entries
  | map(
      "equalizer=f=\(.value.frequency):t=q:w=\(.value.q):g=\(.value.gain)"
    )
  | join(",")
' "$PRESET_PATH")
```

If the preset also has a `highpass` / `lowpass` / `compressor` plugin upstream, prepend the corresponding ffmpeg filters (`highpass=f=<f>`, `lowpass=f=<f>`, `acompressor=...`).

#### c) `band-list-json`

A simple plugin-native format the `onboard` skill writes when the user types in bands manually:

```json
{
  "highpass": 80,
  "bands": [
    { "f": 200,  "q": 1.0, "g": -3 },
    { "f": 3500, "q": 1.2, "g":  2 }
  ],
  "compressor": { "threshold_db": -18, "ratio": 2.5, "attack_ms": 10, "release_ms": 120 }
}
```

Translate the same way: optional `highpass=`, then `equalizer=...` per band, then optional `acompressor=...`.

### 3. Inputs

- **Source** — video or audio path (required).
- **Output dir** — default: sibling `eq/` folder; or project's `renders/` inside an index project.

### 4. Apply

For a video, re-encode the audio (we're modifying it) and stream-copy the video:

```bash
mkdir -p "$OUT_DIR"
OUT="$OUT_DIR/$(basename "${SRC%.*}").eq.${SRC##*.}"
ffmpeg -hide_banner -y -i "$SRC" \
  -map 0 \
  -c:v copy \
  -af "$FILTER" \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  "$OUT"
```

For a standalone audio file:

```bash
ffmpeg -hide_banner -y -i "$SRC" -af "$FILTER" -c:a flac "$OUT_DIR/$(basename "${SRC%.*}").eq.flac"
```

Show the resolved filter chain to the user before running so the EQ is auditable.

### 5. Report

Print output path and size. Suggest:

- A/B test: open original and EQ'd files in a player, compare.
- Run `audio-analysis` on the output to verify loudness landed where expected (EQ shifts loudness slightly).
- Adjust the preset via `onboard` if the result is too bright/dark/honky.
