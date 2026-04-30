# Video Editing Plugin — Scope & Status

Internal roadmap for the public `video-editing` plugin. Gitignored — never ships to the published repo.

Last updated: 2026-04-30 (after Sprint 9).

## Assumptions

- **Linux only.** Skills assume `lspci`, `nvidia-smi`, `vainfo`, `ffmpeg`, `mlt`, `kdenlive`, `rsync`, `pactl`. Windows/macOS users fork.
- **Per-user data store** at `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/`.
- **No user-specific defaults baked in.** Plugin is publicly distributed; per-user state is collected via `onboard`.

## Status overview

| Plugin version | Skills | Date |
|---|---|---|
| 1.1.0 | 3 (foundation) | initial |
| 1.2.0 | +6 render/transcode | sprint 1 |
| 1.3.0 | +2 visual context | sprint 2 |
| 1.4.0 | +2 audio production | sprint 3 |
| 1.5.0 | +2 onboarding + deps | sprint 4 |
| 1.6.0 | data-store path standardization | — |
| 1.7.0 | +2 normalize-audio + auto-cut-silences, legacy cleanup | sprint 5 |
| 1.8.0 | +6 subtitles + deliverables + graphics + music-video | sprint 6 |
| 1.9.0 | +3 NAS lifecycle (setup-nas, pull-from-nas, push-to-nas) | sprint 7 |
| 1.10.0 | +3 clip management (sort-clips-by, scrub-takes, dedupe-clips); retired 3 legacy commands | sprint 8 |
| 1.11.0 | +3 editor integration (open-in-kdenlive, mlt-render, render-from-library) | sprint 9 |
| **current: 1.11.0** | **32 skills, 3 commands** | |

---

## Built and shipped

### Foundation (v1.1.0)
- `_data-store/SKILL.md` — reference doc for the per-user data store.
- `setup-index/SKILL.md` — register/create the video index (base dir holding all projects).
- `open-index/SKILL.md` — open the index in a terminal.
- `new-project/SKILL.md` — scaffold a project workspace (raw/proxies/working/renders/exports/assets/subtitles/graphics).

Two-tier concept: **index** (base dir) → **project** (per-video workspace).

### Render-profile pipeline (v1.2.0 — sprint 1)
- `profile-system/SKILL.md` — detect GPU + ffmpeg encoders → `system-profile.json`. AMD prefers VAAPI over AMF.
- `create-render-profile/SKILL.md` — interactive 30s test-clip iteration loop, saves to `render-profiles.json`.
- `render-with-profile/SKILL.md` — apply named profile to file/folder, with runtime fallback if the chosen encoder fails.
- `list-render-profiles/SKILL.md`, `delete-render-profile/SKILL.md` — CRUD.
- `transcode/SKILL.md` — generic ad-hoc transcode (replaces the deleted `commands/transcode.md`).

### Visual context (v1.3.0 — sprint 2)
- `video-timeline/SKILL.md` — folder of low-res JPEG frames at evenly spaced timestamps + `timeline.md` index, so Claude can "see" a video without ingesting it.
- `wb-preview/SKILL.md` — labeled before/after preview clip for proposed white-balance / color / exposure correction.

### Audio production (v1.4.0 — sprint 3)
- `audio-analysis/SKILL.md` — extract + EBU R128 loudness pass, per-target deltas (YouTube/Spotify/podcast/broadcast).
- `talking-head-eq/SKILL.md` — applies a per-user EQ preset to dialogue audio and re-muxes. Supports EasyEffects JSON, raw ffmpeg-string, and band-list JSON formats.

### Onboarding + optional deps (v1.5.0 — sprint 4)
- `onboard/SKILL.md` — collects per-user prefs (EQ preset, loudness target, default render profile, python venv path) into `preferences.json`. Auto-detects EQ preset format. Idempotent.
- `deps-setup/SKILL.md` — multi-select installer for moviepy, auto-editor, video-use, VideoAgent, vit, LosslessCut (flatpak), editly (npm). Python tools share a `uv`-managed venv.

### Audio apply + auto-edit (v1.7.0 — sprint 5)
- `normalize-audio/SKILL.md` — two-pass ffmpeg `loudnorm` to target LUFS/dBTP from `preferences.json`. Stream-copies video, re-encodes audio only.
- `auto-cut-silences/SKILL.md` — wraps `auto-editor` (from the shared uv venv) to remove dead air. Preview mode via `--export timeline`.
- Deleted superseded legacy commands: `commands/check-codecs.md` (→ `profile-system`), `commands/extract-audio.md` (→ `audio-analysis`).

### NAS lifecycle (v1.9.0 — sprint 7)
- `setup-nas/SKILL.md` — register raw + archive paths to `nas.json`. Supports `mount` and `ssh` transports; verifies reachability without blocking.
- `pull-from-nas/SKILL.md` — rsync raw clips into the active project's `raw/`. Read-only on source (no `--delete`), resumable via `--partial`, dry-run preview by default. Optional extension/recency/subfolder filters.
- `push-to-nas/SKILL.md` — rsync `exports/` (default) or `renders/` to `<archive>/<project-slug>/`. Additive only; dry-run preview before commit.

### Subtitles + deliverables + music video (v1.8.0 — sprint 6)
- `burn-subtitles/SKILL.md` — generate SRT (faster-whisper / whisper.cpp) and either save alongside, soft-mux as a track, or burn into picture. Backend + style read from `preferences.json`.
- `clean-transcription/SKILL.md` — heuristic + optional LLM cleanup of SRT/TXT. Preserves cue count and timing. Per-project glossary support.
- `generate-deliverables/SKILL.md` — coordinator: thumbnail (smart-pick or manual), description (LLM, YouTube/LinkedIn/plain), transcription (delegates to whisper backend). Each deliverable individually requestable.
- `burn-graphics/SKILL.md` — overlay engine for image watermarks, lower thirds, and text titles. JSON-spec multi-overlay or single-overlay shortcut. Supersedes `commands/add-watermark.md` (still present pending verification).
- `karaoke-video/SKILL.md` — Demucs stem separation → vocal attenuation/mute → ASS-styled animated lyric track with `♪` glyphs and per-word `\k` highlight. Whisper-on-isolated-vocals fallback when no LRC/SRT supplied.
- `audio-to-music-video/SKILL.md` — audio + cover art + named template (`waveform-bottom`, `spectrum-bars`, `circular-cqt-cover`, `vector-scope`, `volume-meter`) → composited music video in one ffmpeg pass. Templates live in `preferences.audio_to_video_templates`.
- `onboard` extended: subtitle backend + model + style fields, audio-to-video templates seed.
- `deps-setup` extended: faster-whisper + demucs added to install menu; whisper.cpp documented as a manual install.

### Editor integration (v1.11.0 — sprint 9)
- `open-in-kdenlive/SKILL.md` — `setsid`-detached launch of Kdenlive on a project's `working/` (most-recent `.kdenlive` auto-picked) or on a directly supplied path.
- `mlt-render/SKILL.md` — headless render of `.kdenlive` / `.mlt` timelines via the `melt` CLI. Uses a named profile from `render-profiles.json` when supplied; warns about VAAPI/NVENC and recommends a two-step (intermediate → `render-with-profile`) path for hardware-encoded deliverables.
- `render-from-library/SKILL.md` — assemble a folder of clips into one deliverable. Lossless concat path when all clips share signature; filter-graph re-encode otherwise. `name`/`mtime`/`manual` ordering, optional `xfade` crossfades, optional named render profile for the final encode.

### Clip management (v1.10.0 — sprint 8)
- `sort-clips-by/SKILL.md` — bucket a flat folder of media into subfolders by `resolution` / `aspect` / `fps` / `codec` / `kind` (photo vs video). `mv`-only, dry-run preview before commit. Supersedes `commands/separate-4k.md` and `commands/separate-photos-and-video.md` (deleted).
- `scrub-takes/SKILL.md` — quarantine clips below a minimum duration (default 3s) — moves to `_rejected/` by default; `delete` mode requires explicit second confirmation.
- `dedupe-clips/SKILL.md` — `exact` (sha256) or `perceptual` (sample-frame dHash with configurable Hamming threshold) duplicate detection; per-cluster keeper rule (largest/oldest/newest/first); flatten-and-dedupe sub-flow that supersedes the legacy `commands/videos-here.md`.
- Retired legacy commands: `separate-4k.md`, `separate-photos-and-video.md`, `videos-here.md`.

### Housekeeping
- `.gitignore` excludes `planning/` so this doc never ships.

---

## Data-store files in use

| File | Written by | Read by |
|---|---|---|
| `index.json` | `setup-index` | `open-index`, `new-project`, render skills |
| `system-profile.json` | `profile-system` | `create-render-profile`, `render-with-profile`, `transcode` |
| `render-profiles.json` | `create-render-profile`, `delete-render-profile` | `render-with-profile`, `list-render-profiles`, `onboard` |
| `preferences.json` | `onboard`, `deps-setup` | `talking-head-eq`, `normalize-audio`, `auto-cut-silences`, `burn-subtitles`, `generate-deliverables`, `karaoke-video`, `audio-to-music-video`, future skills |
| `presets/` | `onboard` | `talking-head-eq` |
| `venv/` | `deps-setup` | `auto-cut-silences`, future Python-backed skills |
| `tools/` | `deps-setup` | future skills consuming VideoAgent / vit |
| `nas.json` | `setup-nas` | `pull-from-nas`, `push-to-nas` |

---

## Not yet built (backlog)

Roughly in priority order. None of these block what's already shipped.

### Verification / smoke tests
- End-to-end test on a real AMD box: `profile-system` → `create-render-profile` → `render-with-profile`. Right now everything is unverified against real hardware.
- Sanity-check the `wb-preview` `drawtext` escaping (the `:` in "AFTER\\:" can bite).
- Smoke test `normalize-audio` on a real loud + a real quiet sample.
- Smoke test `auto-cut-silences` on a real lecture-length file once `deps-setup` has installed auto-editor.

### Subtitles + deliverables — shipped in sprint 6 (v1.8.0)

### Editor integration — shipped in sprint 9 (v1.11.0)

### Clip management — shipped in sprint 8 (v1.10.0)

### NAS lifecycle — shipped in sprint 7 (v1.9.0)

### Remaining legacy command convergence
- `commands/add-watermark.md` is now superseded by `burn-graphics` — pending verification, then delete.
- `cut-video-segment.md`, `merge-videos.md` are still v1.0 scratch notes. Per-command decision pending: keep as quick command, fold into a new skill, or delete.

### Agentic editing (from `deps-setup`)
- `agentic-edit` — wrap `video-use` / `VideoAgent` for "make a 60-second cut down of this hour-long stream".
- `editly-render` — wrap `editly` for "compose this slideshow from a JSON spec".

---

## Naming pattern across media plugins

Audio and image plugins use a single-tier workspace (no index/project split — just `workspace-setup` / `open-workspace` / `new-project`). Video is the only one with the two-tier `index` + `project` split, because video projects are heavy and benefit from a richer subfolder layout.

## Reference

Active sprint plan (per-session execution detail): `~/.claude/plans/mutable-singing-swing.md` (mirrors what's in this doc but with file-level diffs and verification steps).
