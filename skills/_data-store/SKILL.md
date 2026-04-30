---
name: data-store
description: Reference for the video-editing plugin's per-user data store. Other skills in this plugin (setup-index, profile-system, render-profiles, setup-nas, etc.) read and write to this location. Linux-only — Windows users will need to fork. Don't invoke directly; this is documentation other skills consult.
disable-model-invocation: true
---

# Plugin Data Store

All persistent state for this plugin lives in a single per-user directory. Skills read and write to this directory; nothing belongs under `~/.claude/`.

## Location

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/
```

## Files

| File                  | Purpose                                                       |
|-----------------------|---------------------------------------------------------------|
| `index.json`          | Path to the user's video index (base dir holding all projects) |
| `system-profile.json` | Detected GPU, ffmpeg encoders, codecs                          |
| `render-profiles.json`| Named, reusable render presets (codec, resolution, bitrate)    |
| `nas.json`            | NAS path / mount, raw + render destinations                    |
| `preferences.json`    | Per-user prefs: EQ preset path, loudness target, default render profile, python venv path, installed tools |
| `presets/`            | Bundled-in copies of user-supplied presets (e.g. talking-head EQ) |
| `venv/`               | Shared `uv`-managed Python virtualenv for moviepy, auto-editor, etc. |
| `tools/`              | Cloned repos for tools without a PyPI release (VideoAgent, vit) |

## Conventions

- All paths stored absolute (use `realpath` to canonicalize).
- All timestamps ISO-8601.
- Read with `jq` when available; fall back to `grep -oP` for portability.
- Write atomically: write to `<file>.tmp` then `mv`.
- Never silently overwrite — always show the existing value and ask before changing.

## Linux-only assumption

This plugin assumes Linux: `lspci`, `nvidia-smi`, `vainfo`, `pactl`, `rsync`, `ffmpeg`, `mlt`, `kdenlive`. Windows or macOS users should fork and adapt.
