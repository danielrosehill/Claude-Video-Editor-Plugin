---
name: setup-nas
description: Register a NAS (or any remote/mounted) location for video raw ingest and finished-render archive. Use when the user says "set up my NAS", "register my video archive", "configure NAS for video", or similar. Persists path(s) to nas.json so pull-from-nas and push-to-nas can find them.
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(ls *), Bash(mkdir *), Bash(realpath *), Bash(date *), Bash(jq *), Bash(stat *), Bash(df *), Bash(mount *), Read, Write
---

# Set Up NAS

Register the location(s) where raw footage lives and finished renders should be archived. Works for any rsync-reachable path: a locally-mounted NAS share, an SSHFS mount, or an `ssh` host alias (`user@host:/path`).

## Procedure

### 1. Resolve data dir

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
NAS_FILE="$DATA_DIR/nas.json"
mkdir -p "$DATA_DIR"
test -f "$NAS_FILE" && cat "$NAS_FILE"
```

If one exists, show the user the current config and ask whether to update or replace.

### 2. Ask the user

1. **Transport**
   - `mount` — locally-mounted path (NFS / SMB / SSHFS already mounted).
   - `ssh` — remote host via rsync-over-ssh (e.g. `daniel@nas.local:/volume1/video`).
2. **Raw source path** — where unsorted/incoming footage lives. Required if they want `pull-from-nas`.
3. **Render archive path** — where finished renders should be pushed. Required if they want `push-to-nas`. Can be the same root as raw or different.
4. **Optional ssh alias / port / identity file** — only if transport is `ssh` and non-default.

### 3. Verify reachability

For `mount` transport:

```bash
test -d "<path>" && ls "<path>" >/dev/null 2>&1
```

If the directory doesn't exist or isn't readable, warn the user (they may need to mount it first) but allow saving anyway — they may be configuring ahead of time.

For `ssh` transport, do a non-interactive probe:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 <host> "test -d '<path>' && echo OK"
```

Report the result. Don't block on failure.

### 4. Persist

Write `nas.json`:

```json
{
  "transport": "mount" | "ssh",
  "raw": {
    "path": "/absolute/or/remote:/path",
    "host": "<only for ssh>",
    "ssh_opts": "<optional, e.g. '-p 2222 -i ~/.ssh/nas_id'>"
  },
  "archive": {
    "path": "/absolute/or/remote:/path",
    "host": "<only for ssh>",
    "ssh_opts": "<optional>"
  },
  "updated": "<ISO-8601>"
}
```

Either `raw` or `archive` may be omitted if the user only wants one side.

### 5. Confirm

Echo the saved config. Mention next-step skills:
- `pull-from-nas` — sync raw clips into the active project's `raw/`.
- `push-to-nas` — sync finished renders up to the archive.
