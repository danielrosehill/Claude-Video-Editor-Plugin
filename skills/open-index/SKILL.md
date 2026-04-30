---
name: open-index
description: Open the user's registered video index — the base directory holding all video projects. Use when the user says "open my video index", "open my video workspace", "show me my video projects", or similar. Reads the path from index.json and opens a terminal there.
disable-model-invocation: false
allowed-tools: Bash(cat *), Bash(test *), Bash(ls *), Bash(jq *), Bash(konsole *), Bash(gnome-terminal *), Bash(xdg-open *), Bash(command *), Read
---

# Open Video Index

Open the registered video index in a terminal (or file manager fallback) and list the existing projects.

## Procedure

```bash
INDEX_FILE="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/index.json"
test -f "$INDEX_FILE" || { echo "No video index registered. Run setup-index first."; exit 1; }
PATH_=$(jq -r .path "$INDEX_FILE" 2>/dev/null || grep -oP '"path":\s*"\K[^"]+' "$INDEX_FILE")
test -d "$PATH_" || { echo "Index path missing: $PATH_"; exit 1; }

if command -v konsole >/dev/null; then
  konsole --workdir "$PATH_" &
elif command -v gnome-terminal >/dev/null; then
  gnome-terminal --working-directory="$PATH_" &
elif command -v xdg-open >/dev/null; then
  xdg-open "$PATH_" &
fi

echo "Index: $PATH_"
ls -la "$PATH_"
```

If the config is missing or stale, redirect the user to the `setup-index` skill.
