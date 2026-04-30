---
name: render-with-profile
description: Render one or more video files using a previously saved render profile. Use when the user says "render these clips at YouTube 1080p", "apply my <profile> preset", "batch render this folder with <profile>", or similar. Reads render-profiles.json + system-profile.json; writes outputs to a renders/ folder.
disable-model-invocation: false
allowed-tools: Bash(ffmpeg *), Bash(ffprobe *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(find *), Bash(jq *), Bash(date *), Bash(realpath *), Bash(stat *), Bash(basename *), Bash(dirname *), Read, Write
---

# Render With Profile

Apply a saved render profile to a file, a folder of files, or a list of files. The profile carries codec/encoder/resolution/rate-control/audio/extra args — this skill just builds the ffmpeg command and loops.

## Procedure

### 1. Resolve config

```bash
DATA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/claude-media-plugins/video-editing"
PROFILES_FILE="$DATA_DIR/render-profiles.json"
SYS_FILE="$DATA_DIR/system-profile.json"
test -f "$PROFILES_FILE" || { echo "No saved profiles — run create-render-profile first."; exit 1; }
test -f "$SYS_FILE"      || { echo "No system profile — run profile-system first."; exit 1; }
```

### 2. Ask the user

- **Profile name** — if not given, list available profiles via `jq '.profiles | keys' "$PROFILES_FILE"` and ask.
- **Inputs** — file, folder (recurse?), or list of paths.
- **Output dir** — default:
  - if invoked inside an index project, `<project>/renders/`
  - else, sibling `rendered/` folder next to inputs

### 3. Build the ffmpeg command from the profile

Read fields with `jq`:

```bash
P=$(jq -r --arg n "$NAME" '.profiles[$n]' "$PROFILES_FILE")
ENCODER=$(echo "$P" | jq -r .encoder)
RES=$(echo "$P"     | jq -r .resolution)
RC_MODE=$(echo "$P" | jq -r .rate_control.mode)
RC_VAL=$(echo "$P"  | jq -r .rate_control.value)
AUDIO_CODEC=$(echo "$P"   | jq -r .audio.codec)
AUDIO_BR=$(echo "$P"      | jq -r .audio.bitrate)
EXTRA=$(echo "$P"   | jq -r '.extra_args | join(" ")')
RENDER_NODE=$(jq -r .gpu.render_node "$SYS_FILE")
```

Construct per encoder family:

- **VAAPI** (`*_vaapi`): `-vaapi_device "$RENDER_NODE" -vf 'format=nv12,hwupload,scale_vaapi=...' -c:v $ENCODER -qp $RC_VAL ...`
- **NVENC** (`*_nvenc`): `-c:v $ENCODER -rc constqp -qp $RC_VAL ...` (or `-b:v` if mode=bitrate)
- **libx264/libx265**: `-c:v $ENCODER -crf $RC_VAL -preset medium ...`

Always pass `-c:a $AUDIO_CODEC -b:a $AUDIO_BR` and the profile's `extra_args`.

### 4. Show the plan

Print the resolved ffmpeg template (with placeholders for IN/OUT) and the list of files about to be rendered. Ask for confirmation if more than 1 file.

### 5. Loop

For each input:

```bash
out="$OUT_DIR/$(basename "${in%.*}").${CONTAINER}"
ffmpeg -hide_banner -y -i "$in" <args> "$out"
```

After each render, report file size and the ratio vs. source. After the batch, summarize: N files, total input size → total output size, elapsed time.

### 6. Errors

If the chosen encoder fails (common with VAAPI on misconfigured systems), fall back to the encoder marked `working` for the same codec in `system-profile.json` and warn the user. If even that fails, abort and tell them to re-run `profile-system`.
