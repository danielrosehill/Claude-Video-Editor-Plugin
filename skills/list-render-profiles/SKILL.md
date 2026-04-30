---
name: list-render-profiles
description: List the user's saved render profiles with their codec, resolution, and rate-control summary. Use when the user says "list my render profiles", "what presets do I have", "show me my render profiles", or similar.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(jq *), Bash(cat *), Read
---

# List Render Profiles

Print the names and key settings of every saved render profile.

## Procedure

```bash
PROFILES_FILE="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/render-profiles.json"
test -f "$PROFILES_FILE" || { echo "No render profiles saved yet. Run create-render-profile."; exit 0; }

jq -r '
  .profiles
  | to_entries[]
  | "\(.key)\t\(.value.codec)/\(.value.encoder)\t\(.value.resolution)\t\(.value.rate_control.mode)=\(.value.rate_control.value)\t\(.value.container)"
' "$PROFILES_FILE" | column -t -s $'\t' -N "NAME,CODEC,RES,RATE,CONT"
```

If the user wants more detail on one profile, `jq '.profiles["<name>"]' "$PROFILES_FILE"`.
