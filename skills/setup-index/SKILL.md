---
name: setup-index
description: Register or create the user's video index — the base directory that holds all their video projects. Use on first run, or when the user says "set up my video index", "register my video workspace", "change my video index path", or similar. Persists the path so other skills can find it.
disable-model-invocation: false
allowed-tools: Bash(mkdir *), Bash(ls *), Bash(test *), Bash(realpath *), Bash(date *), Bash(jq *), Read, Write
---

# Set Up Video Index

The **video index** is the top-level directory under which every video project lives as a subfolder. This skill registers (or creates) that directory.

## Procedure

### 1. Check for existing config

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
INDEX_FILE="$DATA_DIR/index.json"
test -f "$INDEX_FILE" && cat "$INDEX_FILE"
```

If one exists, show the user the current path and ask if they want to keep or change it.

### 2. Ask the user

1. **Existing or new?** — do they already have an index, or should one be created?
2. **Path?**
   - Existing: ask for absolute path; verify with `test -d`.
   - New: propose `~/media-workspaces/video` as the default. Don't auto-create until the user confirms the path.

### 3. Create + persist

```bash
mkdir -p "$DATA_DIR"
mkdir -p "<chosen-path>"   # only if new
```

Write `index.json`:

```json
{
  "path": "/absolute/path",
  "created": "<ISO-8601>",
  "type": "video"
}
```

### 4. Offer follow-up

Suggest the user run `profile-system` next (so render commands know what GPU/encoders to use). Mention they can now say "open my video index" or "create a new video project".
