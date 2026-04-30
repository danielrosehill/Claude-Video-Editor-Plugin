---
name: open-in-kdenlive
description: Open a project's working directory (or a specific .kdenlive file) in Kdenlive. Use when the user says "open this in Kdenlive", "launch the editor for project X", "edit this in Kdenlive". If the project has no .kdenlive file yet, opens the working/ folder via the file dialog so the user can start fresh.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(find *), Bash(jq *), Bash(basename *), Bash(dirname *), Bash(realpath *), Bash(command *), Bash(kdenlive *), Bash(setsid *), Bash(nohup *), Read
---

# Open in Kdenlive

Launch Kdenlive on a project's working directory or specific project file.

## Procedure

### 1. Resolve target

Two input modes:

- **Project name** (slug) — look up under the registered video index.
- **Direct path** — to a `.kdenlive` file or a `working/` folder.

```bash
INDEX_FILE="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/index.json"
INDEX_PATH=$(jq -r .path "$INDEX_FILE" 2>/dev/null)
```

If a project slug was given: `TARGET="$INDEX_PATH/<slug>/working"`.

### 2. Verify Kdenlive is installed

```bash
command -v kdenlive >/dev/null || { echo "Kdenlive not installed. Try: sudo apt install kdenlive  (or flatpak install flathub org.kde.kdenlive)"; exit 1; }
```

### 3. Pick the file to open

```bash
if [ -f "$TARGET" ]; then
  OPEN="$TARGET"
elif [ -d "$TARGET" ]; then
  # Most recently modified .kdenlive in the folder, if any
  OPEN=$(find "$TARGET" -maxdepth 2 -type f -name '*.kdenlive' -printf '%T@ %p\n' 2>/dev/null | sort -nr | head -1 | cut -d' ' -f2-)
  [ -z "$OPEN" ] && OPEN="$TARGET"  # fall back to opening the folder via file dialog
else
  echo "Not found: $TARGET"; exit 1
fi
```

If multiple `.kdenlive` files exist, list them and ask which one (default to most recent).

### 4. Launch (detached)

```bash
setsid kdenlive "$OPEN" >/dev/null 2>&1 < /dev/null &
disown
```

`setsid` + `disown` so Kdenlive survives the shell session.

### 5. Confirm

Print the file/folder opened. Don't block waiting on Kdenlive to exit.
