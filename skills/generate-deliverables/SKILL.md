---
name: generate-deliverables
description: For a finished video, generate one or more publishing deliverables — thumbnail (poster frame), description (LLM summary), transcription (SRT/TXT), or all three. Use when the user says "generate deliverables", "make a thumbnail and description for this", "package this for upload", "give me the YouTube assets". Each deliverable is individually requestable.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(jq *), Bash(realpath *), Bash(basename *), Bash(dirname *), Bash(ffmpeg *), Bash(ffprobe *), Bash(*/whisper-cli *), Bash(whisper-cli *), Bash(uv *), Bash(awk *), Read, Write
---

# Generate Deliverables

Bundle of three small generators. Each can be requested alone or as part of "all three".

| Deliverable | Tool | Output |
|-------------|------|--------|
| Thumbnail | ffmpeg poster frame, optional smart-frame pick | `<basename>.thumb.jpg` |
| Description | Claude reads transcript → writes summary + chapters | `<basename>.description.md` |
| Transcription | whisper backend (delegates to `burn-subtitles` generator) | `<basename>.srt` + `<basename>.txt` |

All deliverables land in a sibling `<basename>.deliverables/` folder unless the project layout dictates otherwise (see step 1).

## Procedure

### 1. Resolve output dir

If the source lives inside a project's `renders/`, write deliverables to the project's `exports/<basename>/`. Otherwise, write to `<dirname>/<basename>.deliverables/`.

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
INDEX_FILE="$DATA_DIR/index.json"
INDEX_PATH=$(jq -r '.path // empty' "$INDEX_FILE" 2>/dev/null)

REL=$(realpath --relative-to="$INDEX_PATH" "$SRC" 2>/dev/null)
if [[ "$REL" == */renders/* ]]; then
  PROJECT_DIR="${SRC%/renders/*}"
  OUT_DIR="$PROJECT_DIR/exports/$(basename "${SRC%.*}")"
else
  OUT_DIR="$(dirname "$SRC")/$(basename "${SRC%.*}").deliverables"
fi
mkdir -p "$OUT_DIR"
```

### 2. Inputs

| Field | Default |
|-------|---------|
| Source | required (rendered video) |
| Deliverables | multi-select: `thumbnail`, `description`, `transcription`, `all` |
| Thumbnail timestamp | `auto` (smart pick — see below); else `HH:MM:SS` |
| Description style | `youtube` (default) / `linkedin` / `plain` |
| Language | auto-detect for transcription |

### 3. Thumbnail

**Manual timestamp**:

```bash
ffmpeg -hide_banner -y -ss "$TS" -i "$SRC" -vframes 1 -q:v 2 "$OUT_DIR/$(basename "${SRC%.*}").thumb.jpg"
```

**Auto pick** — sample N evenly-spaced frames (default 12) and pick the one with the highest "interestingness" score. ffmpeg's `thumbnail` filter does this internally:

```bash
ffmpeg -hide_banner -y -i "$SRC" \
  -vf "thumbnail=300,scale=1280:720:force_original_aspect_ratio=decrease" \
  -frames:v 1 -q:v 2 \
  "$OUT_DIR/$(basename "${SRC%.*}").thumb.jpg"
```

`thumbnail=300` averages over 300-frame batches and picks the most distinctive frame. Reasonable default — fast and avoids dark/transition frames.

After generating, show the thumbnail path and ask if the user wants to redo with a manual timestamp (paired with `video-timeline` makes this easy: list timeline frames, pick one).

### 4. Description

Requires a transcription. If `<basename>.txt` doesn't exist alongside, run the transcription deliverable first (or ask the user to skip if they don't want one).

Read the transcript, then have Claude generate:

- **Title** — single line, ≤70 chars.
- **Summary** — 2–3 sentences, no marketing fluff.
- **Chapters** — extracted from the transcript at natural topic breaks. Format as YouTube chapters: `00:00 Intro`, `02:14 Section name`, ...
- **Tags** — 5–10 keywords.

Write to `$OUT_DIR/$(basename "${SRC%.*}").description.md`:

```markdown
# Title

Summary paragraph.

## Chapters
00:00 Intro
02:14 ...

## Tags
tag1, tag2, ...
```

For the `linkedin` style, drop chapters/tags and produce a 3-paragraph post with a hook in line 1. For `plain`, just summary + tags.

### 5. Transcription

Delegate to the same backend selection logic as `burn-subtitles` (faster-whisper or whisper.cpp from `preferences.subtitle_backend`). Write both `.srt` and `.txt` (plain text from concatenated cue bodies):

```bash
awk 'BEGIN{RS="\n\n"} { n=split($0,a,"\n"); for(i=3;i<=n;i++) printf "%s ", a[i]; print "" }' \
  "$OUT_DIR/$(basename "${SRC%.*}").srt" \
  | sed 's/  */ /g' \
  > "$OUT_DIR/$(basename "${SRC%.*}").txt"
```

If `clean-transcription` is wanted, suggest running it on the SRT before re-using for the description.

### 6. Report

```
Source       : path/to/in.mp4
Out dir      : path/to/in.deliverables/
Thumbnail    : in.thumb.jpg (1280×720, 142 KB)
Transcription: in.srt + in.txt (142 cues, 9.4 min)
Description  : in.description.md (212 words, 6 chapters, 8 tags)
```

## Notes

- This skill is a coordinator — it doesn't reimplement transcription. It calls into the same backend logic as `burn-subtitles`.
- Description quality depends on transcript quality. For a polished YouTube description, run `clean-transcription` first.
- Thumbnail "interestingness" via the `thumbnail` filter is heuristic — for a hero-shot thumbnail, the user should pick manually from `video-timeline` output.
