---
name: audio-analysis
description: Extract audio from a video (or analyze a standalone audio file) and report loudness — integrated LUFS, true peak (dBTP), loudness range — against a target like YouTube (-14 LUFS) or broadcast (-23 LUFS). Use when the user says "check loudness", "is this normalized", "extract the audio", "audio analysis", or "what LUFS is this". Read-only on the source — does not normalize.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(basename *), Bash(dirname *), Bash(grep *), Bash(awk *), Bash(sed *), Read, Write
---

# Audio Analysis

Pull the audio out of a video (or take a standalone audio file) and run an EBU R128 loudness pass. Output is a short markdown report with LUFS / dBTP / LRA and a recommendation against a chosen target.

This skill does not normalize. Apply normalization separately (with `transcode` or a future `normalize-audio` skill) once you've decided on a target.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source | required (video or audio) |
| Mode | `extract` / `analyze` / `both` (default `both`) |
| Target | `youtube` (-14 LUFS / -1 dBTP) / `broadcast` (-23 / -2) / `podcast` (-16 / -1) / `streaming-spotify` (-14 / -1) |
| Output dir | sibling `audio/` (or project's `assets/` if inside an index project) |

### 2. Extract

If the source is a video, pull the audio. Prefer stream-copy when the codec is sane (aac/opus/flac/mp3); otherwise re-encode to wav for analysis.

```bash
CODEC=$(ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of csv=p=0 "$SRC")
case "$CODEC" in
  aac|mp3|opus|flac) EXT="$CODEC"; ARGS="-vn -acodec copy" ;;
  *)                 EXT="wav";    ARGS="-vn -ac 2 -ar 48000 -c:a pcm_s24le" ;;
esac
mkdir -p "$OUT_DIR"
EXTRACTED="$OUT_DIR/$(basename "${SRC%.*}").$EXT"
ffmpeg -hide_banner -y -i "$SRC" $ARGS "$EXTRACTED"
```

### 3. Analyze (ebur128)

```bash
ffmpeg -hide_banner -nostats -i "$EXTRACTED" -af 'ebur128=peak=true:framelog=quiet' -f null - 2>&1 | tail -25
```

Parse the summary block at the end:

```
[Parsed_ebur128_0 @ ...] Summary:
  Integrated loudness:
    I:         -18.4 LUFS
    Threshold: -28.5 LUFS
  Loudness range:
    LRA:         8.1 LU
    Threshold: -38.5 LUFS
    LRA low:   -22.2 LUFS
    LRA high:  -14.1 LUFS
  True peak:
    Peak:       -1.3 dBFS
```

Pull I (integrated), LRA, and Peak with `grep`/`awk`.

Also probe stream metadata:

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bits_per_sample -of default=nw=1 "$SRC"
```

### 4. Compare to target

| Target | Integrated | True Peak |
|--------|-----------:|----------:|
| YouTube | -14 LUFS | -1 dBTP |
| Spotify | -14 LUFS | -1 dBTP |
| Apple Music | -16 LUFS | -1 dBTP |
| Podcast | -16 LUFS | -1 dBTP |
| Broadcast (EBU R128) | -23 LUFS | -2 dBTP |

Compute deltas (`measured_I - target_I`).

### 5. Write the report

`<OUT_DIR>/<basename>.loudness.md`:

```markdown
# Loudness report — <basename>

- Source: `<absolute path>`
- Stream: aac, 48 kHz, 2 ch
- Target: YouTube (-14 LUFS / -1 dBTP)

| Metric | Measured | Target | Delta |
|--------|---------:|-------:|------:|
| Integrated | -18.4 LUFS | -14 LUFS | **-4.4 LU** (too quiet) |
| True peak | -1.3 dBTP | -1 dBTP | -0.3 dB (ok) |
| LRA | 8.1 LU | — | — |

**Recommendation:** raise integrated by ~4.4 LU. Use ffmpeg `loudnorm` (two-pass) or apply gain when re-encoding.
```

### 6. Print summary

Show the table inline so the user doesn't have to open the file. Offer:

- Run `transcode` with a `loudnorm` step.
- Try a different target (re-run analysis with a different reference — no re-extraction needed).
