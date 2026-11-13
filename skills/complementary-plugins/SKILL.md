---
name: complementary-plugins
description: List and install the plugins that complement video-editing, such as remote-video-editing for GPU jobs on Modal (AI voice cleanup). Use on "what goes with this plugin", "install the cloud video plugin", "related plugins".
---

# Complementary plugins

This plugin covers video work on the local machine. These plugins cover what it can't do well locally:

| Plugin | Marketplace | What it adds | Repo |
|--------|-------------|--------------|------|
| `remote-video-editing` | `danielrosehill` | GPU-heavy steps on rented cloud GPUs (Modal), starting with voice cleanup through the Sidon speech-restoration model (`clean-voice-sidon`) | https://github.com/danielrosehill/Remote-Video-Editing |

## Procedure

1. Show the table above. Mark any plugin already installed (`claude plugins list`).
2. For each one the user wants, install it:
   ```bash
   claude plugins marketplace add danielrosehill/Claude-Code-Plugins   # once per machine
   claude plugins install remote-video-editing@danielrosehill
   ```
3. If the user's setup discovers skills on demand instead of installing plugins (a skills catalogue or
   gateway), say so and don't install: search the catalogue for the plugin name instead.

## When to suggest it

- The user wants model-based audio or video processing (speech enhancement, denoising) and the machine
  has no suitable GPU.
- A render or processing job would hold the machine for a long time.
