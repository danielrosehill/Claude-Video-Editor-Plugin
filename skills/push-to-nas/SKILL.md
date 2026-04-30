---
name: push-to-nas
description: Rsync finished renders from a project's renders/ or exports/ folder up to the registered NAS archive. Use when the user says "push to NAS", "archive these renders", "send the final cut to the archive", or similar. Reads nas.json + index.json. Dry-run preview by default, optional per-project subfolder on the archive side.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(realpath *), Bash(jq *), Bash(rsync *), Bash(find *), Bash(date *), Bash(du *), Bash(ssh *), Read, Write
---

# Push To NAS

Sync a project's deliverables up to the registered NAS archive path. Pushes from `exports/` by default (final outputs); `renders/` is also an option for intermediates.

## Procedure

### 1. Resolve config

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
NAS_FILE="$DATA_DIR/nas.json"
INDEX_FILE="$DATA_DIR/index.json"

test -f "$NAS_FILE"   || { echo "No NAS config — run setup-nas first."; exit 1; }
test -f "$INDEX_FILE" || { echo "No video index — run setup-index first."; exit 1; }

TRANSPORT=$(jq -r .transport "$NAS_FILE")
DST=$(jq -r '.archive.path // empty' "$NAS_FILE")
DST_HOST=$(jq -r '.archive.host // empty' "$NAS_FILE")
SSH_OPTS=$(jq -r '.archive.ssh_opts // empty' "$NAS_FILE")
INDEX_PATH=$(jq -r .path "$INDEX_FILE")

test -n "$DST" || { echo "No archive path configured in nas.json — re-run setup-nas."; exit 1; }
```

### 2. Pick project + source folder

- **Project** — named or chosen from `$INDEX_PATH/*/`.
- **Source** — `exports/` (default) or `renders/`. Default to `exports/` because that's "shipped" output.

```bash
SRC="$INDEX_PATH/<project-slug>/exports"
test -d "$SRC" || { echo "Source folder missing: $SRC"; exit 1; }
ls -A "$SRC" >/dev/null 2>&1 || { echo "Nothing in $SRC to push."; exit 0; }
```

### 3. Decide archive layout

Default: push under a per-project subfolder on the archive side, so multiple projects don't collide.

```
<archive_path>/<project-slug>/
```

Confirm the resolved destination with the user before transferring.

For SSH transport, ensure the parent dir exists:

```bash
ssh ${SSH_OPTS} "$DST_HOST" "mkdir -p '$DST/<project-slug>'"
```

For mount transport, just `mkdir -p` locally.

### 4. Build rsync command

Common flags:

```
-av --partial --progress --info=progress2 --human-readable
```

Always do a `--dry-run` first and report file count + bytes. Ask before the real run.

#### Mount transport
```bash
rsync -av --partial --progress --info=progress2 \
  "$SRC/" "$DST/<project-slug>/"
```

#### SSH transport
```bash
rsync -av --partial --progress --info=progress2 \
  -e "ssh ${SSH_OPTS:-}" \
  "$SRC/" "$DST_HOST:$DST/<project-slug>/"
```

### 5. Verify + report

After transfer, confirm count and total bytes. For SSH, optionally re-list the remote directory:

```bash
ssh ${SSH_OPTS} "$DST_HOST" "ls -la '$DST/<project-slug>/'"
```

Suggest the user mark the project as archived in their own notes.

## Safety rules

- **No `--delete`.** Push is additive; we never remove files from the archive based on local state.
- **Confirm before overwriting.** rsync's default updates by mtime/size — fine. But surface large overwrites in the dry-run summary so the user can sanity-check.
- **Don't push `working/`.** That's NLE project state, not deliverables. If the user asks, push it but warn: NLE files often reference absolute paths that won't resolve from the archive.
