---
name: pull-from-nas
description: Rsync raw video clips from the registered NAS into the active project's raw/ folder. Use when the user says "pull from NAS", "import my footage", "grab clips from the archive into project X", or similar. Reads nas.json + index.json. Supports dry-run preview, optional date/extension filters, and resumable transfers.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(realpath *), Bash(jq *), Bash(rsync *), Bash(find *), Bash(date *), Bash(du *), Read, Write
---

# Pull From NAS

Sync raw footage from the registered NAS source into the active project's `raw/` directory. Idempotent and resumable — re-running only transfers what's new or changed.

## Procedure

### 1. Resolve config

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
NAS_FILE="$DATA_DIR/nas.json"
INDEX_FILE="$DATA_DIR/index.json"

test -f "$NAS_FILE"   || { echo "No NAS config — run setup-nas first."; exit 1; }
test -f "$INDEX_FILE" || { echo "No video index — run setup-index first."; exit 1; }

TRANSPORT=$(jq -r .transport "$NAS_FILE")
SRC=$(jq -r '.raw.path // empty' "$NAS_FILE")
SRC_HOST=$(jq -r '.raw.host // empty' "$NAS_FILE")
SSH_OPTS=$(jq -r '.raw.ssh_opts // empty' "$NAS_FILE")
INDEX_PATH=$(jq -r .path "$INDEX_FILE")

test -n "$SRC" || { echo "No raw source configured in nas.json — re-run setup-nas."; exit 1; }
```

### 2. Pick the project

If the user named one, use it. Otherwise list `$INDEX_PATH/*/` and ask. The destination is `<project>/raw/`.

```bash
DEST="$INDEX_PATH/<project-slug>/raw"
test -d "$DEST" || { echo "Project raw/ dir missing — is the project scaffolded?"; exit 1; }
```

### 3. Optional filters

Ask (or accept from the user):
- **Extension filter** — e.g. `mp4,mov,mxf`. Default: pull everything.
- **Newer-than** — e.g. files modified in the last N days. Translates to `--files-from` via `find -mtime -N` on remote, or `find` on a local mount.
- **Subfolder** — pull only a named subfolder under the configured `raw.path`.

### 4. Build the rsync command

Common flags:

```
-av --partial --progress --info=progress2 --human-readable
```

Add `--dry-run` first if the user hasn't explicitly skipped preview. Print the count and total size, then ask before running for real.

#### Mount transport
```bash
rsync -av --partial --progress --info=progress2 \
  ${INCLUDE_FILTERS:-} \
  "$SRC/<subfolder>/" "$DEST/"
```

#### SSH transport
```bash
rsync -av --partial --progress --info=progress2 \
  -e "ssh ${SSH_OPTS:-}" \
  ${INCLUDE_FILTERS:-} \
  "$SRC_HOST:$SRC/<subfolder>/" "$DEST/"
```

For extension filtering, use `--include='*.mp4' --include='*.mov' --include='*/' --exclude='*'`.

### 5. Run + report

After the real run, summarize:
- Files transferred / skipped
- Total bytes moved
- Destination path
- Suggest next skills: `video-timeline` to scan what was pulled, or `sort-clips-by` (when built) to organize.

## Notes

- **Never delete** on the source. Pull-from-nas is read-only against the NAS — no `--delete`.
- **Resumable** — `--partial` means an interrupted transfer continues on re-run.
- **Permissions** — preserve mtimes (`-a` does this); don't try to preserve owner/group across hosts.
