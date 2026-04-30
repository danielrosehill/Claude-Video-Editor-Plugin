---
name: delete-render-profile
description: Delete a saved render profile by name. Use when the user says "delete the <name> render profile", "remove my <name> preset", or "clean up old render profiles".
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(jq *), Bash(mv *), Bash(cat *), Read, Write
---

# Delete Render Profile

Remove a profile from `render-profiles.json`. Confirms before deleting.

## Procedure

```bash
DATA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/claude-media-plugins/video-editing"
PROFILES_FILE="$DATA_DIR/render-profiles.json"
test -f "$PROFILES_FILE" || { echo "No profiles file."; exit 0; }
```

1. If a name wasn't given, list profiles (use the same `jq` from `list-render-profiles`) and ask which to delete.
2. Show the full profile JSON for the chosen name and ask for confirmation.
3. Delete atomically:

```bash
jq --arg n "$NAME" 'del(.profiles[$n])' "$PROFILES_FILE" > "$PROFILES_FILE.tmp" && mv "$PROFILES_FILE.tmp" "$PROFILES_FILE"
```

4. Confirm deletion. If `.profiles` is now empty, leave the file in place (don't delete the JSON itself).
