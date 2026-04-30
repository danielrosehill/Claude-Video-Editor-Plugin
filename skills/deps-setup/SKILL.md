---
name: deps-setup
description: Install optional video-editing tools (moviepy, auto-editor, video-use, VideoAgent, vit, LosslessCut, editly). Lets the user pick which to install; Python tools share a single uv-managed venv recorded in preferences.json. Use when the user says "install video tools", "set up moviepy", "install auto-editor", "deps setup", "install lossless cut".
disable-model-invocation: false
allowed-tools: Bash(test *), Bash(mkdir *), Bash(curl *), Bash(uv *), Bash(git *), Bash(npm *), Bash(flatpak *), Bash(jq *), Bash(command *), Bash(date *), Bash(realpath *), Bash(ls *), Bash(which *), Read, Write
---

# Deps Setup

Optional installer for third-party video-editing tools. The user picks a subset; each is installed using its native ecosystem's tooling and recorded in `preferences.json` so other skills (and future skills) can find it.

Python-based tools share a single `uv`-managed virtualenv at:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing/venv
```

The venv path is stored as `preferences.python_venv` so it can be sourced later (e.g. `"$PYTHON_VENV/bin/python" -m moviepy ...`).

## Tools

| Tool | Repo | Type | Install |
|------|------|------|---------|
| moviepy | https://github.com/Zulko/moviepy | Python (PyPI) | `uv pip install moviepy` |
| auto-editor | https://github.com/WyattBlue/auto-editor | Python (PyPI) | `uv pip install auto-editor` |
| video-use | https://github.com/browser-use/video-use | Python (git) | `uv pip install git+https://github.com/browser-use/video-use` |
| VideoAgent | https://github.com/HKUDS/VideoAgent | Python (git, research) | `git clone` + `uv pip install -r requirements.txt` |
| vit | https://github.com/LucasHJin/vit | Python (git) | `git clone` + `uv pip install -r requirements.txt` |
| faster-whisper | https://github.com/SYSTRAN/faster-whisper | Python (PyPI) | `uv pip install faster-whisper` |
| demucs | https://github.com/facebookresearch/demucs | Python (PyPI) | `uv pip install demucs` |
| LosslessCut | https://github.com/mifi/lossless-cut | Electron app | flatpak: `no.mifi.losslesscut` (fallback: AppImage) |
| editly | https://github.com/mifi/editly | Node.js | `npm install -g editly` (warn: needs Cairo + ffmpeg) |
| whisper.cpp | https://github.com/ggerganov/whisper.cpp | C++ binary | manual — clone, `make`, download GGML model. Record binary + model paths in preferences via `onboard`. |

## Procedure

### 1. Resolve config

```bash
DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/video-editing"
PREFS_FILE="$DATA_DIR/preferences.json"
TOOLS_DIR="$DATA_DIR/tools"
VENV_DIR="$DATA_DIR/venv"
mkdir -p "$DATA_DIR" "$TOOLS_DIR"
test -f "$PREFS_FILE" || echo '{}' > "$PREFS_FILE"
```

### 2. Bootstrap `uv` (if any Python tool selected)

```bash
if ! command -v uv >/dev/null; then
  echo "uv not installed. Run: curl -LsSf https://astral.sh/uv/install.sh | sh"
  # Ask user before running. Don't silently curl|sh.
fi
```

Confirm with the user before running the install script.

### 3. Create the shared venv (lazy)

```bash
if [ ! -d "$VENV_DIR" ]; then
  uv venv "$VENV_DIR"
fi
```

### 4. Ask the user which to install

Multi-select. Show each tool's purpose in one line:

- **moviepy** — Python video editing library (programmatic clip ops)
- **auto-editor** — auto-cut silences and dead air from video
- **video-use** — agentic video understanding from browser-use
- **VideoAgent** — research multi-agent video understanding (HKU)
- **vit** — video transformer toolkit (LucasHJin)
- **faster-whisper** — fast CPU/GPU whisper inference (used by `burn-subtitles`, `karaoke-video`)
- **demucs** — Facebook Research stem separation (used by `karaoke-video`)
- **LosslessCut** — GUI for fast lossless trim/cut
- **editly** — declarative JSON → video composition (Node)
- **whisper.cpp** — alternative pure-C++ whisper backend (manual install, see notes)

### 5. Install per selection

For each chosen tool, **show the exact command first**, ask for confirmation, run, capture exit status.

#### Python (PyPI)

```bash
uv pip install --python "$VENV_DIR/bin/python" moviepy
```

#### Python (git clone + requirements)

```bash
git clone https://github.com/HKUDS/VideoAgent "$TOOLS_DIR/VideoAgent"
uv pip install --python "$VENV_DIR/bin/python" -r "$TOOLS_DIR/VideoAgent/requirements.txt"
```

#### LosslessCut (flatpak)

```bash
if command -v flatpak >/dev/null; then
  flatpak install -y --user flathub no.mifi.losslesscut
else
  echo "flatpak missing — fall back to AppImage download from GitHub releases"
fi
```

#### editly (npm)

```bash
if ! command -v npm >/dev/null; then
  echo "Node.js not installed. Install Node first (see nodejs.org)."
  # Don't auto-install Node — too invasive for a video plugin.
fi
npm install -g editly
```

### 6. Record results

Update `preferences.json`:

```json
{
  "python_venv": "/abs/.../venv",
  "tools": {
    "moviepy":     { "installed": true, "method": "uv-pip", "venv": "...", "version": "1.0.3" },
    "auto-editor": { "installed": true, "method": "uv-pip", "venv": "...", "version": "..." },
    "VideoAgent":  { "installed": true, "method": "git+uv-pip", "path": "...tools/VideoAgent" },
    "lossless-cut":{ "installed": true, "method": "flatpak", "id": "no.mifi.losslesscut" },
    "editly":      { "installed": true, "method": "npm-global", "version": "..." },
    "faster-whisper": { "installed": true, "method": "uv-pip", "venv": "..." },
    "demucs":      { "installed": true, "method": "uv-pip", "venv": "..." }
  }
}
```

For **whisper.cpp**, deps-setup does not auto-install (build environment varies — clang vs gcc, BLAS deps, AVX support). Print a one-block install hint and ask the user to run it themselves, then re-run `onboard` to register the binary + model paths:

```
git clone https://github.com/ggerganov/whisper.cpp ~/whisper.cpp
cd ~/whisper.cpp && make
bash ./models/download-ggml-model.sh small.en
# then: re-run onboard and point subtitle_backend=whisper.cpp at:
#   subtitle_model_path=~/whisper.cpp/models/ggml-small.en.bin
#   binary on PATH: ~/whisper.cpp/main  (newer builds: whisper-cli)
```

Write atomically (`.tmp` + `mv`).

### 7. Report

Per tool: success / failed / skipped. For failures, surface stderr and a one-line hint (e.g. "Cairo dev headers missing — `apt install libcairo2-dev`"). Don't pretend a tool is installed if its command failed.
