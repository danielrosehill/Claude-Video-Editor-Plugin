---
name: new-project
description: Create a new video project workspace inside the user's registered video index. Use when the user says "new video project", "create a project called X", "scaffold a new video", or similar. Creates a project subfolder with raw, proxies, working, renders, exports, assets layout and an optional git init.
disable-model-invocation: false
allowed-tools: Bash(cat *), Bash(test *), Bash(mkdir *), Bash(ls *), Bash(git *), Bash(date *), Bash(realpath *), Bash(jq *), Read, Write
---

# New Video Project

Scaffold a project workspace inside the registered video index.

## Procedure

### 1. Resolve the index

```bash
INDEX_FILE="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/index.json"
test -f "$INDEX_FILE" || { echo "No index — run setup-index first."; exit 1; }
INDEX_PATH=$(jq -r .path "$INDEX_FILE" 2>/dev/null || grep -oP '"path":\s*"\K[^"]+' "$INDEX_FILE")
```

### 2. Ask for project metadata

- **Name** — required. Slugify (lowercase, hyphens for spaces).
- **Version control** — default **no** for video. Most video projects are too large for git.

### 3. Create the layout

```bash
PROJ="$INDEX_PATH/<slug>"
test -e "$PROJ" && { echo "Already exists: $PROJ"; exit 1; }
mkdir -p "$PROJ"/{raw,proxies,working,renders,exports,assets,subtitles,graphics}
```

| Folder       | Purpose                                              |
|--------------|------------------------------------------------------|
| `raw/`       | Original source clips — never edited in place        |
| `proxies/`   | Low-res transcodes for fast scrubbing                |
| `working/`   | NLE project files (Kdenlive, MLT XML)                |
| `renders/`   | Intermediate / preview renders                       |
| `exports/`   | Final deliverables                                   |
| `assets/`    | Music, lower thirds, graphics, SFX                   |
| `subtitles/` | SRT / VTT files                                      |
| `graphics/`  | Overlays, lower thirds, watermarks                   |

Write a `README.md` with the slug, ISO date, and a short notes section.

### 4. (Optional) git init

If the user wants version control:

```bash
cd "$PROJ" && git init
```

Add `.gitignore` excluding `raw/`, `proxies/`, `renders/`, `exports/` — only project file + notes get versioned.

### 5. Confirm

Print the project path. Offer to `cd` there or open in a terminal.
