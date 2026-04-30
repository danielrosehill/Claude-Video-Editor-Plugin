---
name: clean-transcription
description: Clean up a raw whisper transcript (SRT or TXT) — strip filler words, fix recurring mistranscriptions via a per-project glossary, and (optionally) re-flow line breaks. Use when the user says "clean this transcript", "remove the ums", "fix the transcription", "tidy up the SRT before burning". Pairs with burn-subtitles (run before burning) and generate-deliverables.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(jq *), Bash(realpath *), Bash(basename *), Bash(dirname *), Bash(sed *), Bash(awk *), Bash(grep *), Bash(wc *), Bash(cp *), Read, Write, Edit
---

# Clean Transcription

Two-stage cleanup:

1. **Heuristic** — regex pass over filler words, repeated stutters, all-caps shouting, and a per-project glossary of known mistranscriptions.
2. **LLM polish** (optional) — Claude rewrites cues for grammar/punctuation while preserving timing.

Output is a parallel file: `<basename>.clean.srt` (or `.clean.txt`). Original is never modified.

## Procedure

### 1. Inputs

| Field | Default |
|-------|---------|
| Source | required (`.srt` or `.txt`) |
| Mode | `heuristic` (default) / `llm` / `both` |
| Glossary | `<project>/subtitles/glossary.json` if present; or path supplied inline |
| Output | sibling `<basename>.clean.<ext>` |

### 2. Heuristic pass

#### Filler words

Default list (case-insensitive, word-boundary):

```
um, uh, erm, ah, like, you know, sort of, kind of, basically, literally, I mean, right
```

Don't blanket-strip — `like` and `right` have legitimate uses. Strip only when surrounded by sentence-internal commas / mid-cue ("..., um, ..."). For SRT input, operate per-cue text without touching the index or timing lines.

```bash
# Per-cue text only — preserve "N\n00:00:01,000 --> 00:00:03,000\n..." structure
awk 'BEGIN{RS="\n\n"; ORS="\n\n"} {
  n=split($0, lines, "\n");
  for(i=3;i<=n;i++) {
    gsub(/\<(um|uh|erm)\>[ ,]*/, "", lines[i])
    gsub(/  +/, " ", lines[i])
  }
  for(i=1;i<=n;i++) printf "%s%s", lines[i], (i==n?"":"\n")
}' "$SRC" > "$OUT"
```

#### Stutters

Collapse adjacent identical short tokens: `the the cat` → `the cat`. Only collapse 1–4 character tokens (don't merge `really really good`).

#### Glossary

`glossary.json` is a list of `{wrong, right}` pairs. Apply as literal substitutions (not regex) unless explicitly marked.

```json
{
  "substitutions": [
    { "wrong": "claud", "right": "Claude" },
    { "wrong": "ml flow", "right": "MLflow" },
    { "wrong": "kdenlive", "right": "Kdenlive", "case_insensitive": true }
  ]
}
```

Walk the array and apply with `sed`. For SRT, again confine to text lines.

### 3. LLM polish (optional)

When mode is `llm` or `both`, ask Claude to rewrite cue text in place. Constraints:

- Preserve cue index and timestamps verbatim. Do not merge or split cues.
- Stay within the original cue's character budget × 1.1 (small leeway). If it must shrink, that's fine.
- Fix punctuation, capitalisation, obvious grammar. Don't paraphrase or summarise.
- Keep proper nouns and technical terms exactly as they appear in the glossary.

Process cues in batches of ~30 to keep context tight. After each batch, validate that cue count and timestamp lines are unchanged before writing.

### 4. Diff + report

Show before/after stats and a small sampled diff (5 random cues):

```
Source : in.srt — 142 cues, 8.2 KB
Output : in.clean.srt — 142 cues, 7.4 KB
Heuristic: removed 38 fillers, 6 stutters, applied 12 glossary subs
LLM polish: 142 cues rewritten
Sample diff (cue 27):
- Um, so the the thing is, you know, we want to..
+ The thing is, we want to..
```

Always validate cue count is unchanged. If it isn't, abort and keep `.tmp` for inspection.

## Notes

- For TXT input (plain transcript, no timing), skip the SRT-aware structure-preserving logic and run the same passes line-by-line.
- LLM polish on a long transcript is the slow step. For big files, run heuristic first, ask the user if they want to also LLM-polish.
- Glossary lives at the project level, not the data store — the right vocabulary is project-specific (one project's "Daniel" is another's "Daniela").
