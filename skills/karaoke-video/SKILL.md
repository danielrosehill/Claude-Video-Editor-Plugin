---
name: karaoke-video
description: Build a karaoke-style music video from a source video or song — separate stems with Demucs, attenuate (or remove) the lead vocal, then burn animated lyric subtitles with music-note glyphs synced to the song. Accepts an existing LRC/SRT lyric file, or whisper-transcribes the original vocal as a fallback. Use when the user says "make this karaoke", "remove the vocals and add lyrics", "karaoke version", "instrumental with lyrics".
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(jq *), Bash(realpath *), Bash(basename *), Bash(dirname *), Bash(ffmpeg *), Bash(ffprobe *), Bash(*/demucs *), Bash(demucs *), Bash(*/whisper-cli *), Bash(whisper-cli *), Bash(uv *), Bash(awk *), Bash(sed *), Read, Write
---

# Karaoke Video

Composite pipeline:

1. Extract audio from the source.
2. Run [Demucs](https://github.com/facebookresearch/demucs) to separate stems (`vocals`, `drums`, `bass`, `other`).
3. Mix a karaoke audio bed: instrumental at full level + lead vocal at user-chosen attenuation (`-12 dB` default; `mute` removes entirely).
4. Build the lyric track:
   - If the user supplied an LRC or SRT, use it.
   - Otherwise, run whisper on the **isolated vocals** stem (much cleaner transcript than mixed audio) and convert to ASS with per-line timing.
5. Render the karaoke video — original picture + new audio bed + animated subtitles with music-note bullets.

## Procedure

### 1. Resolve tools

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS="$DATA_DIR/preferences.json"
VENV=$(jq -r '.python_venv // empty' "$PREFS" 2>/dev/null)

DEMUCS="$VENV/bin/demucs"
[ -x "$DEMUCS" ] || DEMUCS=$(command -v demucs)
[ -n "$DEMUCS" ] || { echo "Demucs not found. Run deps-setup and select demucs."; exit 1; }
```

Whisper backend resolution mirrors `burn-subtitles` (use `preferences.subtitle_backend`).

### 2. Inputs

| Field | Default |
|-------|---------|
| Source | required (video or audio) |
| Vocal level | `-12dB` (alt: `mute`, `-6dB`, `-18dB`) |
| Lyrics | optional path to `.lrc` / `.srt` |
| Lyric style | `bouncy` (default) / `block` / `word-highlight` |
| Output | `<basename>.karaoke.mp4` |

### 3. Demucs separation

```bash
WORK=$(mktemp -d)
"$DEMUCS" --two-stems=vocals -o "$WORK" "$SRC"
# Output layout: $WORK/htdemucs/<basename>/vocals.wav and no_vocals.wav
STEMS="$WORK/htdemucs/$(basename "${SRC%.*}")"
VOCALS="$STEMS/vocals.wav"
INSTR="$STEMS/no_vocals.wav"
```

For 4-stem separation (drums/bass/other), drop `--two-stems=vocals`. Two-stems is faster and sufficient for karaoke.

### 4. Mix the karaoke audio bed

```bash
case "$VOCAL_LEVEL" in
  mute)   FILTER="[1:a]anull[bed]" ;;                                # instrumental only
  *)      DB="${VOCAL_LEVEL/dB/}"; DB="${DB/-/-}"
          FILTER="[0:a]volume=${DB}dB[v];[v][1:a]amix=inputs=2:normalize=0:duration=longest[bed]" ;;
esac

ffmpeg -hide_banner -y -i "$VOCALS" -i "$INSTR" \
  -filter_complex "$FILTER" \
  -map "[bed]" -c:a aac -b:a 192k \
  "$WORK/karaoke-audio.m4a"
```

Set `normalize=0` so the instrumental retains its loudness and the vocal is genuinely attenuated rather than auto-leveled back up.

### 5. Lyrics

#### If LRC supplied

LRC is line-timed (`[mm:ss.xx]Lyric line`). Convert to ASS with karaoke `\k` tags for per-syllable highlight (when `lyric_style=word-highlight`), or to plain SRT for `block` / `bouncy`.

For `word-highlight`: split each line into words, distribute the line's duration evenly across words, emit `{\kNN}word` tokens. (More accurate karaoke needs forced-alignment — out of scope.)

#### If SRT supplied

Use as-is for `block` / `bouncy`. For `word-highlight`, same word-split approach as LRC.

#### If nothing supplied — whisper the isolated vocal

```bash
"$WHISPER" --output_format srt --output_dir "$WORK" --language "$LANG" "$VOCALS"
LYRICS="$WORK/$(basename "${VOCALS%.*}").srt"
```

Vocals-only input gives whisper a much cleaner signal than the mix — output is significantly more accurate. Suggest the user run `clean-transcription` on this SRT before rendering.

### 6. Subtitle styling — music-note glyphs

ASS supports inline styling and Unicode glyphs. Build the karaoke ASS with a music-note prefix on each cue:

```
Style: Karaoke,Inter,52,&H00FFFFFF,&H000000FF,&H00000000,&HCC000000,-1,0,0,0,100,100,0,0,1,3,2,2,30,30,80,1
Dialogue: 0,0:00:05.00,0:00:08.50,Karaoke,,0,0,0,,♪ {\fad(200,200)}This is a lyric line ♪
```

For `bouncy` style, add a vertical wobble via `\move` or `\t(0,500,\fscy105)\t(500,1000,\fscy100)` (subtle 5% scale pulse on the beat — looks animated without forced-alignment data).

For `word-highlight`, use ASS karaoke tags (`\k`) so each word lights up in sequence as its duration elapses:

```
Dialogue: 0,0:00:05.00,0:00:08.50,Karaoke,,0,0,0,,♪ {\k50}This {\k40}is {\k30}a {\k60}lyric {\k70}line ♪
```

(The `\k` value is centiseconds. Computed by `word_duration_ms / 10`.)

Music-note glyphs to consider: `♪` `♫` `♬` `🎵`. Stick to the BMP characters (`♪♫♬`) — emoji renders inconsistently with libass.

Save the styled file as `$WORK/lyrics.ass`.

### 7. Render

```bash
ESC_ASS=$(printf %s "$WORK/lyrics.ass" | sed -e 's/\\/\\\\/g' -e "s/'/\\\\'/g" -e 's/:/\\:/g')

ffmpeg -hide_banner -y \
  -i "$SRC" \
  -i "$WORK/karaoke-audio.m4a" \
  -filter_complex "[0:v]ass='${ESC_ASS}'[v]" \
  -map "[v]" -map 1:a \
  -c:v libx264 -crf 18 -preset medium \
  -c:a copy \
  -movflags +faststart \
  "$OUT"
```

The `ass` filter (vs `subtitles`) honours full ASS styling including animations.

For **audio-only sources** (no source video), pair this with `audio-to-music-video` to produce a waveform background, then run karaoke-video over that synthesised video.

### 8. Cleanup

```bash
rm -rf "$WORK"
```

…unless the user wants to keep the stems for further work — in which case copy to `<project>/working/stems/` first.

### 9. Report

```
Source     : path/to/song.mp4
Stems      : separated via Demucs (htdemucs)
Vocals     : attenuated -12 dB (alt: mute / -6dB / -18dB)
Lyrics     : 42 lines (whisper-transcribed from isolated vocal)
Style      : word-highlight, ♪ glyphs, bouncy fade
Output     : path/to/song.karaoke.mp4 (1080p, 4:32, 38 MB)
```

## Notes

- Demucs first run downloads the `htdemucs` model (~80 MB). Subsequent runs are fast.
- If the user wants a true sing-along (perfect per-syllable timing), forced alignment (e.g. `aeneas`, `whisperx`) is the right tool — out of scope here.
- Music-note glyph rendering depends on the system having a font that covers `U+266A`. Inter does. If a custom font is set in `preferences.subtitle_style`, verify it before rendering.
- Demucs is GPU-accelerated when CUDA is available. On AMD-only systems it runs CPU-only — separation of a 4-min track takes ~2–4 minutes.
