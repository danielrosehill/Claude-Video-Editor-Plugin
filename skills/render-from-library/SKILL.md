---
name: render-from-library
description: Assemble a single video from a folder of clips — concat in a chosen order, optionally with crossfade transitions, normalized to a target resolution/fps. Use when the user says "stitch this folder into one video", "render-from-library", "concat the clips in folder X", "make a montage from these clips". Lossless concat path when all clips share codec/params; transcoding fallback otherwise.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(find *), Bash(sort *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(printf *), Bash(awk *), Bash(ffmpeg *), Bash(ffprobe *), Read, Write
---

# Render From Library

Assemble a single deliverable from a folder of clips.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source folder | required |
| Output | default: `<source>/_assembled_<timestamp>.mp4` |
| Order | `name` (alphabetical) — `mtime` (modification time) — `manual` (user supplies an ordered list) |
| Recursive | `no` |
| Transitions | `none` (hard cuts) — `crossfade:<seconds>` (default 0.5s when on) |
| Normalize | `auto` — re-encode if clips differ; pick most-common resolution/fps as target |
| Render profile | optional — apply a named profile from `render-profiles.json` for the final encode |

### 2. Enumerate

```bash
SRC="$(realpath "$INPUT")"
if [ "$RECURSIVE" = "yes" ]; then
  mapfile -t CLIPS < <(find "$SRC" -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.mkv' -o -iname '*.m4v' \))
else
  mapfile -t CLIPS < <(find "$SRC" -maxdepth 1 -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.mkv' -o -iname '*.m4v' \))
fi
case "$ORDER" in
  name)  IFS=$'\n' CLIPS=($(printf '%s\n' "${CLIPS[@]}" | sort)); unset IFS ;;
  mtime) IFS=$'\n' CLIPS=($(printf '%s\n' "${CLIPS[@]}" | xargs -d '\n' stat -c '%Y %n' | sort -n | cut -d' ' -f2-)); unset IFS ;;
  manual) ;; # caller supplies the array directly
esac
[ ${#CLIPS[@]} -lt 2 ] && { echo "Need at least 2 clips."; exit 1; }
```

### 3. Probe each clip

```bash
probe() {
  ffprobe -v error -select_streams v:0 \
    -show_entries stream=width,height,r_frame_rate,codec_name -of json "$1"
}
```

Build a table of (codec, w, h, fps) per clip. Record the most-common signature.

### 4. Choose strategy

- **Lossless concat** if all clips share codec + container + width/height/fps and there are **no transitions** — use ffmpeg concat demuxer:
  ```bash
  printf "file '%s'\n" "${CLIPS[@]}" > /tmp/concat-list.txt
  ffmpeg -f concat -safe 0 -i /tmp/concat-list.txt -c copy "$OUT"
  ```
- **Filter-graph re-encode** otherwise. Build with `xfade` for crossfades or `concat` filter for hard cuts. All clips first re-scaled / fps-converted to the target signature with `scale` + `fps`.

### 5. Build the filter graph (re-encode path)

For hard cuts (no transition):

```
[0:v]scale=W:H,fps=F,setsar=1[v0];[1:v]scale=W:H,fps=F,setsar=1[v1];...
[v0][0:a][v1][1:a]...concat=n=N:v=1:a=1[v][a]
```

For crossfades:

Chain `xfade` between consecutive video streams, offsetting at `(cumulative_duration - fade_duration)`. Audio uses `acrossfade`. The cumulative offset must be computed from each clip's duration via ffprobe — list them in the preview before running.

### 6. Pick the encoder

If a render profile was supplied, build the output args from it (same shape `render-with-profile` uses). Otherwise default to `-c:v libx264 -crf 20 -preset medium -c:a aac -b:a 192k`.

### 7. Preview

Print:
- ordered clip list with durations + total duration
- chosen strategy (lossless concat / re-encode + transition)
- target resolution/fps and encoder
- estimated output path

Ask for **confirm** before running.

### 8. Run + verify

```bash
mkdir -p "$(dirname "$OUT")"
ffmpeg ... "$OUT"
ffprobe -v error -show_entries format=duration,size -of default=nw=1 "$OUT"
```

### 9. Confirm

Print output path, size, duration. Offer follow-ups (`audio-analysis`, `burn-graphics`, `generate-deliverables`).

## Notes

- Crossfades require all clips to be re-encoded — don't promise a "lossless crossfade".
- Audio sample rates that differ across clips are resampled to 48 kHz on the re-encode path.
- For a folder of stills, prefer the `audio-to-music-video` skill (cover-art + audio composite) — this skill is for video clips.
- Manual ordering: caller passes an explicit ordered list. The skill should accept it as a JSON array argument or read from a `library.txt` in the source folder, one filename per line.
