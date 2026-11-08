---
title: "Run OpenCode on local models with llama.cpp on an AMD GPU (Ubuntu)"
description: "Serve five local models (gpt-oss, Qwen3.6, Qwen3.8, Nemotron 3.5 Lightning, Muse Glimmer) from llama.cpp's router mode on a 12 GB AMD GPU over Vulkan, and use them from the OpenCode CLI and desktop app on Ubuntu 26.10."
status: working
status_note: "Every step was run on the listed system; the server starts by itself after a reboot, and all five models were used from the OpenCode CLI and desktop app."
last_verified: 2026-11-07
tested_on:
  - "Ubuntu 26.10, kernel 7.3.0-10-generic"
  - "AMD Radeon RX 6700 XT 12 GB, Mesa 26.2.4 RADV Vulkan driver"
  - "AMD Ryzen 7 5800X3D, 30 GB RAM as reported by free"
  - "llama.cpp unified llama binary 0.9.0-dev (build 12182, commit fd27b2034), Vulkan backend"
  - "OpenCode CLI 2.2.6 and OpenCode desktop 2.2.6 (AppImage)"
  - "libfuse2t64 2.9.9-9build1"
  - "Model files stored on a CIFS (SMB) network share"
ai_assisted: true
tags: [opencode, llama.cpp, local-llm, vulkan, amd, rx-6700-xt, gpt-oss, qwen, nemotron, systemd, appimage, ubuntu]
---

# Run OpenCode on local models with llama.cpp on an AMD GPU (Ubuntu)

**Goal:** Use OpenCode (an AI coding agent) with models that run entirely on your own PC instead of a cloud API. The hard parts are fitting 20–35B-parameter models into 12 GB of VRAM, keeping the model files on a network share without making loading slow, and getting OpenCode's model list (especially in the desktop app) to show local models correctly.

**End state:** A llama.cpp server starts when you log in, listens on `http://127.0.0.1:8080`, loads one of five models on demand, and unloads it after 30 minutes of no use. The OpenCode CLI and the OpenCode desktop app both list the five models under the provider `llama.cpp (local)` and can call tools with them.

**Time:** about 2 hours, mostly waiting for about 77 GB of model downloads. **Difficulty:** intermediate.

## Summary

- Point llama.cpp's model cache (`LLAMA_CACHE`) at the folder that should hold the model files, and download five GGUF models into it.
- Describe each model (file, context size, sampling) in a llama.cpp router preset file, `~/.config/llama.cpp/models.ini`.
- Run `llama serve` in router mode as a systemd user service, so it starts at login and keeps one model loaded at a time.
- Install the OpenCode CLI and add the server as an OpenAI-compatible provider in `~/.config/opencode/opencode.jsonc`.
- Install the OpenCode desktop AppImage (needs `libfuse2t64`) and add a menu entry for it.
- To add more models later, follow "Add another model later" (Steps A1 to A7).

### Results on the tested system

Measured with a 12,000-token prompt while the desktop was in normal use. Expect similar numbers on a 12 GB card; the dense models are slow because they do not fit in VRAM.

| Model (preset name) | Type | File size | Prompt speed | Generation speed |
|---|---|---|---|---|
| `gpt-oss-20b` | MoE, 3.6B active | 12.1 GB | 959 tokens/s | 16–22 tokens/s |
| `qwen3.6-35b-a3b` | MoE, 3B active | 22.4 GB | 393 tokens/s | 12–18 tokens/s |
| `nemotron-3.5-lightning` | hybrid Mamba-2/MoE, 3B active | 18.9 GB | 477 tokens/s | 13–19 tokens/s |
| `qwen3.8-27b` | dense | 10.9 GB | 213 tokens/s | about 8 tokens/s |
| `muse-glimmer-30b` | dense | 13.1 GB | 221 tokens/s | 4–6 tokens/s |

All five models made correct tool calls in testing. `qwen3.6-35b-a3b` is the default in this guide because it is the best balance of coding ability and speed on this hardware.

## Before you start

### Requirements

- Ubuntu 26.10 or close to it, with `sudo` access (only needed to install `libfuse2t64`).
- An AMD GPU with a working Vulkan driver. This guide was tested on an RX 6700 XT with 12 GB of VRAM and the Mesa 26.2.4 RADV driver.
- About 30 GB of RAM. The parts of each model that do not fit in VRAM run from RAM.
- llama.cpp's unified `llama` binary, built with the Vulkan backend, at `~/.local/bin/llama`. This guide was tested with version 0.9.0-dev, build 12182. Older builds may not have router mode or the `load-mode`, `sleep-idle-seconds`, `fit-target`, and `dedup-cache-models` options. Installing llama.cpp is not covered here.
- About 80 GB of free space in the folder that will hold the model files.
- `curl` and `python3`, for the checks.

### Assumptions

- The model files go in one folder, written as `<MODEL_DIR>` below. On the tested system that folder is on a CIFS (SMB) network share mounted at boot through `/etc/fstab`. The mount point of that share is written as `<MODEL_MOUNT>`. If your models are on a local disk, you can skip the mount checks.
- The server only listens on `127.0.0.1`, so only programs on the same PC can use it.
- You use bash as your shell and log in to a graphical desktop session that runs systemd user services (the Ubuntu default).

### Back up first

- Step 2 appends to `~/.bashrc` and `~/.profile`. Back them up with `cp ~/.bashrc ~/.bashrc.bak-llama && cp ~/.profile ~/.profile.bak-llama`.
- Step 10 creates `~/.config/opencode/opencode.jsonc`. If that file already exists, copy it first with `cp ~/.config/opencode/opencode.jsonc ~/.config/opencode/opencode.jsonc.bak`.

## Steps

### Step 1: Check that llama.cpp sees the GPU

llama.cpp's `--fit` placement (on by default) decides how much of each model goes on the GPU, so it has to find the GPU through Vulkan.

```bash
~/.local/bin/llama serve --list-devices
```

**Check:** The output ends with a line like `Vulkan0: AMD Radeon RX 6700 XT (RADV NAVI22) (12288 MiB, 10766 MiB free)`. The free amount is lower than the total because the desktop uses some VRAM.

**If it fails:** if no `Vulkan0` line appears, the `llama` binary was built without Vulkan or the Vulkan driver is missing. Fix that before you continue.

### Step 2: Set LLAMA_CACHE for terminals

`LLAMA_CACHE` tells llama.cpp where to download and look for model files. It takes priority over `HF_HUB_CACHE`. Setting it in both `~/.bashrc` and `~/.profile` covers interactive and login shells.

```bash
mkdir -p "<MODEL_DIR>"
printf '\n# llama.cpp: store downloaded models here\nexport LLAMA_CACHE="%s"\n' "<MODEL_DIR>" | tee -a ~/.bashrc >> ~/.profile
```

Replace `<MODEL_DIR>` (both times) with the full path of the folder for model files, for example `/mnt/nas/llm-models`.

**Check:** Open a new terminal and run `echo $LLAMA_CACHE`. Expected output: the path you used for `<MODEL_DIR>`.

### Step 3: Set LLAMA_CACHE for the graphical session

Programs started from the desktop menu do not read `~/.bashrc`. systemd reads `~/.config/environment.d/` at login and passes the variable to everything in the session.

```bash
mkdir -p ~/.config/environment.d
echo 'LLAMA_CACHE=<MODEL_DIR>' > ~/.config/environment.d/llama-cpp.conf
systemctl --user set-environment LLAMA_CACHE=<MODEL_DIR>
```

Replace `<MODEL_DIR>` (both times) with the full path of the folder for model files. The `set-environment` line applies it to the current session without logging out.

**Check:** Run `systemctl --user show-environment | grep LLAMA_CACHE`. Expected output: `LLAMA_CACHE=` followed by the path you used for `<MODEL_DIR>`.

### Step 4: Download the five models

Each command downloads one GGUF file from Hugging Face into `LLAMA_CACHE`. The quantizations were chosen so each model fits the 12 GB GPU plus RAM: the two dense models (Qwen3.8-27B and Muse Glimmer 30B) use 3-bit `IQ3_XXS` files so most of each model fits in VRAM.

```bash
export LLAMA_CACHE="<MODEL_DIR>"
~/.local/bin/llama download -hf ggml-org/gpt-oss-20b-GGUF -hff gpt-oss-20b-MXFP4.gguf --no-mmproj
~/.local/bin/llama download -hf unsloth/Qwen3.6-35B-A3B-GGUF -hff Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf --no-mmproj
~/.local/bin/llama download -hf unsloth/Qwen3.8-27B-GGUF -hff Qwen3.8-27B-UD-IQ3_XXS.gguf --no-mmproj
~/.local/bin/llama download -hf ggml-org/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-GGUF -hff NVIDIA-Nemotron-3.5-Lightning-30B-A3B-Q4_0.gguf --no-mmproj
~/.local/bin/llama download -hf unsloth/Muse-Glimmer-30B-GGUF -hff Muse-Glimmer-30B-UD-IQ3_XXS.gguf --no-mmproj
```

Replace `<MODEL_DIR>` with the full path of the folder for model files. The downloads total about 77 GB.

The files are stored in the Hugging Face cache layout, `models--<org>--<repo>/snapshots/<hash>/<file>.gguf`. On a CIFS share, which does not support symlinks, llama.cpp writes the files directly into `snapshots/` instead of linking them from a `blobs/` folder. That is expected.

**Check:** Run `LLAMA_CACHE="<MODEL_DIR>" ~/.local/bin/llama serve --cache-list`. Expected output starts with `number of models in cache: 5` and lists:

```
ggml-org/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-GGUF:Q4_0
ggml-org/gpt-oss-20b-GGUF:MXFP4
unsloth/Qwen3.8-27B-GGUF:IQ3_XXS
unsloth/Qwen3.6-35B-A3B-GGUF:Q4_K_XL
unsloth/Muse-Glimmer-30B-GGUF:IQ3_XXS
```

### Step 5: Create the router preset file

In router mode, `llama serve` starts with no model loaded and loads a model when a request names it. The preset file sets the model name that clients use (the section name), the file to load, the context size, and the sampling settings each model's publisher recommends.

```bash
mkdir -p ~/.config/llama.cpp
cat > ~/.config/llama.cpp/models.ini <<'EOF'
; llama.cpp router presets for OpenCode.
; Section names are the model IDs that clients (OpenCode) request.
version = 1

[*]
; one request at a time keeps the context cache small on a 12 GB GPU
parallel = 1
flash-attn = on
cache-type-k = q8_0
cache-type-v = q8_0
jinja = true
; never contact Hugging Face at load time; use the downloaded files only
offline = true
no-mmproj = true
; read weights into memory instead of memory-mapping them from the network share:
; only the CPU-side part stays in RAM, and pages are never re-fetched over the network
load-mode = none
; 8 physical cores on the 5800X3D (SMT threads slow generation down)
threads = 8
; unload the model (frees VRAM and RAM) after 30 minutes without requests;
; the next request reloads it automatically
sleep-idle-seconds = 1800
; keep ~512 MiB of VRAM free for the desktop; --fit places the rest of each model in RAM
fit-target = 512
; hide the long cache names (ggml-org/...:Q4_0) so /v1/models only lists the names below
dedup-cache-models = true

; OpenAI gpt-oss 20B (MoE, 3.6B active) - official MXFP4 build
[gpt-oss-20b]
hf-repo = ggml-org/gpt-oss-20b-GGUF
hf-file = gpt-oss-20b-MXFP4.gguf
c = 131072
temp = 1.0
top-p = 1.0
top-k = 0
min-p = 0
reasoning-effort = medium

; Qwen3.6 35B-A3B (MoE, 3B active) - fast everyday agentic coder
[qwen3.6-35b-a3b]
hf-repo = unsloth/Qwen3.6-35B-A3B-GGUF
hf-file = Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf
; 64k keeps more experts on the GPU: ~18 tokens/s vs ~12 tokens/s at 128k
c = 65536
temp = 0.6
top-p = 0.95
top-k = 20
min-p = 0

; Qwen3.8 27B (dense) - strongest coder that fits, but slow on 12 GB:
; 48k context with a 4-bit KV cache is the largest that keeps 61/66 layers on the GPU
[qwen3.8-27b]
hf-repo = unsloth/Qwen3.8-27B-GGUF
hf-file = Qwen3.8-27B-UD-IQ3_XXS.gguf
c = 49152
cache-type-k = q4_0
cache-type-v = q4_0
fit-target = 384
temp = 1.0
top-p = 0.95
top-k = 20
min-p = 0
; model default is xhigh, which is very slow on local hardware
reasoning-effort = medium

; NVIDIA Nemotron 3.5 Lightning 30B-A3B (hybrid Mamba-2/MoE) - official ggml-org Q4_0
[nemotron-3.5-lightning]
hf-repo = ggml-org/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-GGUF
hf-file = NVIDIA-Nemotron-3.5-Lightning-30B-A3B-Q4_0.gguf
c = 131072
temp = 1.0
top-p = 0.95

; Meta Muse Glimmer 30B (dense) - IQ3_XXS so most of it fits in VRAM
[muse-glimmer-30b]
hf-repo = unsloth/Muse-Glimmer-30B-GGUF
hf-file = Muse-Glimmer-30B-UD-IQ3_XXS.gguf
c = 65536
temp = 1.0
top-p = 0.95
top-k = 64
EOF
```

Change `threads = 8` to the number of physical CPU cores on your machine.

**Check:** Run `grep -c '^\[' ~/.config/llama.cpp/models.ini`. Expected output: `6` (the `[*]` section plus five models).

### Step 6: Create the systemd user service

The service runs the router server at login. It waits up to 2 minutes for the network share with the model files to be mounted, and restarts the server if it crashes.

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/llama-server.service <<'EOF'
[Unit]
Description=llama.cpp router server for OpenCode
After=network-online.target

[Service]
Type=simple
Environment=LLAMA_CACHE=<MODEL_DIR>
# wait up to 2 minutes for the network share that holds the model files
ExecStartPre=/bin/sh -c 'for i in $(seq 60); do mountpoint -q <MODEL_MOUNT> && [ -d "$LLAMA_CACHE" ] && exit 0; sleep 2; done; echo "model share <MODEL_MOUNT> not mounted" >&2; exit 1'
# presets in ~/.config/llama.cpp/models.ini, one model loaded at a time, localhost only
ExecStart=%h/.local/bin/llama serve --models-preset %h/.config/llama.cpp/models.ini --models-max 1 --host 127.0.0.1 --port 8080
Restart=on-failure
RestartSec=15

[Install]
WantedBy=default.target
EOF
```

Replace `<MODEL_DIR>` with the full path of the folder for model files, and `<MODEL_MOUNT>` (both times) with the mount point of the share that holds it (for example `/mnt/nas`). If the models are on a local disk, delete the `ExecStartPre=` line.

`--models-max 1` keeps only one model in memory. When OpenCode asks for a different model, the server unloads the current one first.

**Check:** Run `systemd-analyze --user verify ~/.config/systemd/user/llama-server.service`. Expected result: no output.

### Step 7: Enable and start the llama.cpp service

`enable --now` starts the server immediately and at every later login. The service starts when you log in, not at boot.

```bash
systemctl --user daemon-reload
systemctl --user enable --now llama-server.service
```

**Check:** Run `systemctl --user is-enabled llama-server; systemctl --user is-active llama-server`. Expected output: `enabled` then `active`. Then run:

```bash
curl -s http://127.0.0.1:8080/v1/models | python3 -c 'import json,sys; print([m["id"] for m in json.load(sys.stdin)["data"]])'
```

Expected output: `['gpt-oss-20b', 'muse-glimmer-30b', 'nemotron-3.5-lightning', 'qwen3.6-35b-a3b', 'qwen3.8-27b']`

### Step 8: Send a test request to the llama.cpp server

The first request to a model loads it, which takes from about 30 seconds to a few minutes when the files are on a network share.

```bash
curl -s -m 600 http://127.0.0.1:8080/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"gpt-oss-20b","max_tokens":300,"messages":[{"role":"user","content":"What is 17 * 23? Reply with just the number."}]}' | python3 -c 'import json,sys; print(json.load(sys.stdin)["choices"][0]["message"]["content"])'
```

**Check:** Expected output: `391`.

### Step 9: Install the OpenCode CLI

The official installer puts the `opencode` binary in `~/.opencode/bin` and adds that folder to `PATH` in `~/.bashrc`.

```bash
curl -fsSL https://opencode.ai/v2/install | bash
```

**Check:** Open a new terminal and run `opencode --version`. Expected output: `opencode v2.2.6` (or a later version; this guide was tested on 2.2.6).

### Step 10: Add the llama.cpp server to the OpenCode config

This adds a provider with the ID `llamacpp` that talks to the server's OpenAI-compatible API. Notes on the settings:

- `timeout` and `headerTimeout` are off, because loading a model from a network share can take minutes.
- `reasoningField` makes OpenCode show the models' thinking, which llama.cpp returns in `reasoning_content`.
- `"temperature": false` stops OpenCode sending its own temperature, so the sampling settings in `~/.config/llama.cpp/models.ini` apply.
- `limit.context` must match the `c =` value for the same model in `~/.config/llama.cpp/models.ini`.
- `variants` let you pick a reasoning effort per request in OpenCode.
- Do not add an old `releaseDate` to a model. The desktop app's model picker hides models it considers outdated (see "What didn't work").

```bash
mkdir -p ~/.config/opencode
cat > ~/.config/opencode/opencode.jsonc <<'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  // Default: the fast MoE coder. Switch to qwen3.8-27b (smarter, ~2x slower) for hard tasks.
  "model": "llamacpp/qwen3.6-35b-a3b",
  "providers": {
    // Local llama.cpp router server (systemd user service llama-server.service).
    // Model presets (sampling, context, offload): ~/.config/llama.cpp/models.ini
    "llamacpp": {
      "name": "llama.cpp (local)",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "http://127.0.0.1:8080/v1",
        // the first request to a model loads it from disk, which can take minutes
        "timeout": false,
        "headerTimeout": false
      },
      "compatibility": {
        "reasoningField": "reasoning_content",
        "maxTokensField": "max_tokens",
        "streamUsage": true
      },
      "models": {
        "qwen3.6-35b-a3b": {
          "name": "Qwen3.6 35B-A3B (Q4_K_XL) - fast coder",
          "family": "qwen",
          // sampling comes from models.ini, so OpenCode doesn't send a temperature
          "capabilities": { "tools": true, "temperature": false, "input": ["text"], "output": ["text"],
                            "reasoning": { "supported": true } },
          "limit": { "context": 65536, "output": 32768 }
        },
        "qwen3.8-27b": {
          "name": "Qwen3.8 27B (IQ3_XXS) - smart, slow",
          "family": "qwen",
          "releaseDate": "2026-08-05",
          "capabilities": { "tools": true, "temperature": false, "input": ["text"], "output": ["text"],
                            "reasoning": { "supported": true } },
          "limit": { "context": 49152, "output": 16384 },
          "variants": [
            { "id": "low",   "body": { "reasoning_effort": "low" } },
            { "id": "high",  "body": { "reasoning_effort": "high" } },
            { "id": "xhigh", "body": { "reasoning_effort": "xhigh" } }
          ]
        },
        "gpt-oss-20b": {
          "name": "gpt-oss 20B (MXFP4)",
          "family": "gpt-oss",
          // no releaseDate: the desktop model picker hides models it considers outdated
          "capabilities": { "tools": true, "temperature": false, "input": ["text"], "output": ["text"],
                            "reasoning": { "supported": true } },
          "limit": { "context": 131072, "output": 32768 },
          "variants": [
            { "id": "low",  "body": { "reasoning_effort": "low" } },
            { "id": "high", "body": { "reasoning_effort": "high" } }
          ]
        },
        "nemotron-3.5-lightning": {
          "name": "Nemotron 3.5 Lightning 30B-A3B (Q4_0)",
          "family": "nemotron",
          "capabilities": { "tools": true, "temperature": false, "input": ["text"], "output": ["text"],
                            "reasoning": { "supported": true } },
          "limit": { "context": 131072, "output": 32768 }
        },
        "muse-glimmer-30b": {
          "name": "Muse Glimmer 30B (IQ3_XXS)",
          "family": "muse",
          "releaseDate": "2026-08",
          "capabilities": { "tools": true, "temperature": false, "input": ["text"], "output": ["text"],
                            "reasoning": { "supported": true } },
          "limit": { "context": 65536, "output": 32768 }
        }
      }
    }
  }
}
EOF
```

**Check:** Run `opencode models --standalone | grep llamacpp`. Expected output:

```
llamacpp/gpt-oss-20b
llamacpp/muse-glimmer-30b
llamacpp/nemotron-3.5-lightning
llamacpp/qwen3.6-35b-a3b
llamacpp/qwen3.8-27b
```

### Step 11: Restart the OpenCode background service

OpenCode 2.x runs a background service that the desktop app and the CLI share. It reads `~/.config/opencode/opencode.jsonc` when it starts, so restart it after every change to that file.

```bash
opencode service restart
```

**Check:** Run `opencode service status`. Expected output: `http://127.0.0.1:49374`.

### Step 12: Test a tool call from the OpenCode CLI

This asks the default model to read a file with a planted bug. It checks that OpenCode reaches the server and that the model can call OpenCode's Read tool. Without `--auto`, OpenCode does not let the model change files. `--standalone` runs without the background service, and `--archived` keeps the test out of your session list.

```bash
mkdir -p ~/opencode-test && cd ~/opencode-test
printf 'def add(a, b):\n    return a - b\n' > calc.py
opencode run --standalone --archived -m llamacpp/qwen3.6-35b-a3b "Read calc.py and tell me in one sentence whether add() is correct. Do not modify any files."
```

**Check:** The output shows a Read call on `calc.py`, then a sentence saying `add()` is wrong because it subtracts instead of adding. On the tested system this took about 150 seconds, including loading the model. You can then delete the folder with `rm -r ~/opencode-test`.

### Step 13: Install libfuse2 for the desktop AppImage

The OpenCode desktop app ships as an AppImage, which needs the FUSE 2 library. Ubuntu 26.10 does not install it by default; the package is called `libfuse2t64`.

```bash
sudo apt install libfuse2t64
```

**Check:** Run `ls /usr/lib/x86_64-linux-gnu/libfuse.so.2`. Expected output: `/usr/lib/x86_64-linux-gnu/libfuse.so.2`.

### Step 14: Download the OpenCode desktop AppImage

```bash
mkdir -p ~/Applications
curl -fL -o ~/Applications/OpenCode.AppImage https://opencode.ai/update/api/latest/desktop/opencode/opencode-desktop-linux-x86_64.AppImage
chmod 755 ~/Applications/OpenCode.AppImage
```

**Check:** Run `file ~/Applications/OpenCode.AppImage`. Expected output contains `ELF 64-bit LSB executable`. The tested file (version 2.2.6) was about 313 MB.

### Step 15: Install the desktop app's icons

The icons are inside the AppImage. Extract them into a temporary folder and copy them into your icon theme folder.

```bash
mkdir -p /tmp/opencode-icons && cd /tmp/opencode-icons
~/Applications/OpenCode.AppImage --appimage-extract 'usr/share/icons/*' >/dev/null
for s in 32x32 64x64 128x128; do
  mkdir -p ~/.local/share/icons/hicolor/$s/apps
  cp squashfs-root/usr/share/icons/hicolor/$s/apps/ai.opencode.png ~/.local/share/icons/hicolor/$s/apps/
done
cd ~ && rm -r /tmp/opencode-icons
```

**Check:** Run `ls ~/.local/share/icons/hicolor/*/apps/ai.opencode.png`. Expected output: three paths, for the `32x32`, `64x64`, and `128x128` folders.

### Step 16: Add the desktop app to the application menu

The `.desktop` file inside the AppImage starts the app with `--no-sandbox`. That flag is not needed on Ubuntu 26.10: the app starts normally with Chromium's sandbox on, even with `kernel.apparmor_restrict_unprivileged_userns = 1`. This menu entry leaves it out.

```bash
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/ai.opencode.desktop <<EOF
[Desktop Entry]
Name=OpenCode
Comment=OpenCode desktop (AppImage in ~/Applications)
Exec=$HOME/Applications/OpenCode.AppImage %U
Terminal=false
Type=Application
Icon=ai.opencode
StartupWMClass=ai.opencode.desktop
MimeType=x-scheme-handler/opencode;
Categories=Development;
EOF
desktop-file-validate ~/.local/share/applications/ai.opencode.desktop && update-desktop-database ~/.local/share/applications
```

The heredoc is unquoted on purpose, so `$HOME` is replaced with your home folder. `.desktop` files need an absolute path in `Exec=`.

**Check:** Run `grep Exec ~/.local/share/applications/ai.opencode.desktop`. Expected output: `Exec=/home/<USER>/Applications/OpenCode.AppImage %U`, with your username in place of `<USER>`. OpenCode also appears in the application menu with its icon.

### Step 17: Pick a local model in the desktop app

Start OpenCode from the application menu. Open the model picker.

**Check:** The picker lists a `llama.cpp (local)` group with all five models: Qwen3.6 35B-A3B, Qwen3.8 27B, gpt-oss 20B, Nemotron 3.5 Lightning, and Muse Glimmer 30B.

**If it fails:** if some models are missing, see "What didn't work", under "Setting an old releaseDate on a model in opencode.jsonc".

## Verify everything works

1. Run `systemctl --user is-active llama-server`. Expected output: `active`.
2. Run `opencode models --standalone | grep -c llamacpp`. Expected output: `5`.
3. Run the CLI test from Step 12 (`opencode run --standalone --archived -m llamacpp/qwen3.6-35b-a3b "Read calc.py ..."` in a folder containing the buggy `calc.py`). Expected result: the model reads `calc.py` and says `add()` subtracts instead of adding.
4. In the desktop app, choose `Qwen3.6 35B-A3B (Q4_K_XL) - fast coder` and send a coding question. Expected result: an answer after the model loads (about 30 seconds to a few minutes for the first message when the files are on a network share).
5. Reboot, log in, and run `systemctl --user is-active llama-server`. Expected output: `active`, without starting anything by hand. On the tested system the service was active 24 seconds after boot, with `NRestarts=0` in `systemctl --user show llama-server -p NRestarts`.

Optional: check that idle unloading works. With `sleep-idle-seconds = 60` temporarily set in `~/.config/llama.cpp/models.ini` (and `systemctl --user restart llama-server`), send a request, wait 70 seconds, and run `cat /sys/class/drm/card*/device/mem_info_vram_used`. On the tested system VRAM use dropped from 11.3 GB to 1.3 GB, and the next request reloaded the model in 33 seconds. Set it back to `1800` afterwards.

## Add another model later

Use these steps to add a sixth (or later) model after the setup above works. Each model needs three things: its GGUF file in the model folder, a section in `~/.config/llama.cpp/models.ini`, and an entry in `~/.config/opencode/opencode.jsonc`. The steps are numbered A1 to A7 so they don't mix with the main steps.

These steps were tested end to end with a small test model on a separate test server (port 8081) and a separate OpenCode config folder. The five models in the main steps were added the same way.

### Step A1: Choose a GGUF file that fits

Find the model's GGUF repository on Hugging Face (official `ggml-org/...` builds, or `unsloth/...` and similar for more sizes). This command lists every GGUF file in a repository with its size:

```bash
curl -s "https://huggingface.co/api/models/<REPO>/tree/main?recursive=true" | python3 -c 'import json,sys; [print("%6.1f GB  %s" % (f["size"]/1e9, f["path"])) for f in json.load(sys.stdin) if f["path"].endswith(".gguf")]'
```

Replace `<REPO>` with the repository name, for example `unsloth/Qwen3.6-35B-A3B-GGUF`.

How to pick a file on a 12 GB GPU with 30 GB of RAM, based on the results table near the top of this guide:

- **MoE models** (names with `A3B` or similar, meaning about 3B parameters active per token): a 4-bit file (`Q4_K_XL`, `Q4_K_M`, `Q4_0`, `MXFP4`) up to about 22 GB generated 12–22 tokens/s. Larger files leave less RAM for the rest of the system.
- **Dense models** (no active-parameter count in the name): every layer that does not fit in VRAM slows down every token. Pick a file of about 10 GB or less (often `IQ3_XXS` or `IQ2_*` for 27–30B models). A 10.9 GB file ran at about 8 tokens/s; a 13.1 GB file ran at 4–6 tokens/s.
- Skip `mmproj-*` files (image input, which this setup turns off with `no-mmproj`), `BF16`/`F16` files (full size), and files split into parts (`-00001-of-00002`). Split files were not tried with this setup.

**Check:** You have one repository name and one file name, and the file size follows the rules above.

### Step A2: Download the model file

```bash
LLAMA_CACHE="<MODEL_DIR>" ~/.local/bin/llama download -hf <REPO> -hff <FILE> --no-mmproj
```

Replace `<MODEL_DIR>` with the full path of the folder for model files, `<REPO>` with the repository name, and `<FILE>` with the file name from Step A1, for example `-hf unsloth/Qwen3.6-35B-A3B-GGUF -hff Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf`.

**Check:** The last line of the output is the full path of the downloaded file, ending in `<FILE>`. For files with a quantization name, `LLAMA_CACHE="<MODEL_DIR>" ~/.local/bin/llama serve --cache-list` also lists `<REPO>:<QUANT>`, for example `unsloth/Qwen3.6-35B-A3B-GGUF:Q4_K_XL`.

### Step A3: Add a section to the router preset file

Open `~/.config/llama.cpp/models.ini` in a text editor and add this at the end:

```ini
; <SHORT_DESCRIPTION>
[<MODEL_ID>]
hf-repo = <REPO>
hf-file = <FILE>
c = 65536
temp = <TEMP>
top-p = <TOP_P>
```

Replace:

- `<MODEL_ID>` with the short name OpenCode will use, in lowercase with hyphens, for example `qwen3.6-35b-a3b`. It must not match any other section name in `~/.config/llama.cpp/models.ini`.
- `<REPO>` and `<FILE>` with the values from Step A2.
- `<TEMP>` and `<TOP_P>` with the sampling settings on the model's Hugging Face page (usually under "Best practices" or "Recommended settings"). Add `top-k`, `min-p`, or `reasoning-effort` lines if the model page recommends them.
- `<SHORT_DESCRIPTION>` with a reminder of what the model is.

`c = 65536` (64k tokens of context) is a good start for MoE models. Use `c = 32768` or `c = 49152` for dense models. Every setting in the `[*]` section at the top of `~/.config/llama.cpp/models.ini` (flash attention, 8-bit KV cache, `load-mode = none`, idle unload) applies to the new model too. Any of them can be overridden by adding the same key to the new section, for example `cache-type-k = q4_0` and `cache-type-v = q4_0` to halve the KV cache size.

**Check:** Run `grep -c '^\[' ~/.config/llama.cpp/models.ini`. Expected output: one more than before (`7` after the five models in this guide plus `[*]`).

### Step A4: Restart the llama.cpp service and test the new model

The router reads `~/.config/llama.cpp/models.ini` only when it starts. Restarting unloads any model that is currently loaded.

```bash
systemctl --user restart llama-server
sleep 5
curl -s -m 900 http://127.0.0.1:8080/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"<MODEL_ID>","max_tokens":300,"messages":[{"role":"user","content":"Write a Python function that reverses a string."}]}' | python3 -c 'import json,sys; d=json.load(sys.stdin); t=d["timings"]; print(d["choices"][0]["message"]["content"][:300]); print("%.0f tokens/s prompt, %.1f tokens/s generation" % (t["prompt_per_second"], t["predicted_per_second"]))'
```

Replace `<MODEL_ID>` with the section name from Step A3.

**Check:** Run `curl -s http://127.0.0.1:8080/v1/models | python3 -c 'import json,sys; print([m["id"] for m in json.load(sys.stdin)["data"]])'`. Expected output includes `<MODEL_ID>`. The test request prints a Python function, then a line in the form `<N> tokens/s prompt, <N> tokens/s generation`. The first request also loads the model, which takes from about 30 seconds to a few minutes. Below about 8 tokens/s generation, OpenCode will feel slow: lower `c` in the model's section, or pick a smaller file in Step A1.

**If it fails:** see Troubleshooting. The error is in `journalctl --user -u llama-server -n 50`.

### Step A5: Add the model to the OpenCode config

Open `~/.config/opencode/opencode.jsonc` in a text editor. Inside `"providers"` → `"llamacpp"` → `"models"`, add this entry before the closing `}` of `"models"`, and add a comma after the entry before it:

```jsonc
        "<MODEL_ID>": {
          "name": "<DISPLAY_NAME>",
          "capabilities": { "tools": true, "temperature": false, "input": ["text"], "output": ["text"],
                            "reasoning": { "supported": true } },
          "limit": { "context": <CONTEXT>, "output": 32768 }
        }
```

Replace:

- `<MODEL_ID>` with the section name from Step A3, exactly.
- `<DISPLAY_NAME>` with the name to show in OpenCode's model picker, for example `Qwen3.6 35B-A3B (Q4_K_XL)`.
- `<CONTEXT>` with the `c =` value from the model's section in `~/.config/llama.cpp/models.ini`, for example `65536`. If it is 32768 or less, set `"output"` to half of it.

Set `"supported": false` under `"reasoning"` if the model has no thinking mode. To pick a reasoning effort per chat, copy the `"variants"` block from the `gpt-oss-20b` entry in Step 10, but only if the model's page says it supports `reasoning_effort`. Leave out `"releaseDate"`: the desktop app's model picker hides models it considers outdated. On the tested system an entry with `"releaseDate": "2025-08-05"` was hidden, and entries with 2026 dates were shown.

**Check:** Run `opencode models --standalone | grep llamacpp/<MODEL_ID>`. Expected output: `llamacpp/<MODEL_ID>`. No output means the entry was not read; check the commas and braces in `~/.config/opencode/opencode.jsonc`.

### Step A6: Restart the OpenCode background service

The desktop app and CLI share a background service that reads `~/.config/opencode/opencode.jsonc` only when it starts.

```bash
opencode service restart
```

**Check:** Open the model picker in the OpenCode desktop app (close and reopen the app if it was already running). Expected result: `<DISPLAY_NAME>` appears in the `llama.cpp (local)` group.

### Step A7: Test a tool call with the new model

```bash
mkdir -p ~/opencode-test && cd ~/opencode-test
printf 'def add(a, b):\n    return a - b\n' > calc.py
opencode run --standalone --archived -m llamacpp/<MODEL_ID> "Read calc.py and tell me in one sentence whether add() is correct. Do not modify any files."
```

Replace `<MODEL_ID>` with the section name from Step A3.

**Check:** The output shows a Read call on `calc.py`, then a sentence saying `add()` subtracts instead of adding. If the model answers without reading the file, or prints the tool call as plain text, it does not handle OpenCode's tool calls well; use it for chat only or pick another model. Delete the test folder with `rm -r ~/opencode-test`.

### Remove a model

This permanently deletes the model file. Download it again with Step A2 if you change your mind.

1. Delete the model's section (from its `;` comment line to the line before the next `[`) from `~/.config/llama.cpp/models.ini`, and run `systemctl --user restart llama-server`.
2. Delete the model's entry from `"models"` in `~/.config/opencode/opencode.jsonc`, and run `opencode service restart`.
3. Delete the model's files:

```bash
rm -r "<MODEL_DIR>/models--<ORG>--<NAME>"
```

Replace `<MODEL_DIR>` with the full path of the folder for model files, and `<ORG>--<NAME>` with the repository name with `/` replaced by `--`, for example `models--unsloth--Muse-Glimmer-30B-GGUF` for `unsloth/Muse-Glimmer-30B-GGUF`. If you downloaded several files from the same repository, this deletes all of them.

**Check:** Run `curl -s http://127.0.0.1:8080/v1/models | python3 -c 'import json,sys; print([m["id"] for m in json.load(sys.stdin)["data"]])'`. Expected output: the list no longer includes the removed model.

## What didn't work

### Memory-mapping model files from the network share

llama.cpp memory-maps model files by default. With the files on a CIFS share, `qwen3.6-35b-a3b` generated only about 11 tokens/s. The whole 20 GB file counted against RAM (resident set size) and the system started swapping, because mapped pages had to be fetched again over the network after they were dropped. Setting `load-mode = none` in the `[*]` section of `~/.config/llama.cpp/models.ini` reads the weights into memory once instead. Only the part of the model that runs on the CPU stays in RAM.

### 128k context for Qwen3.6 35B-A3B

At `c = 131072`, the larger KV cache pushed more of the model's experts out of VRAM, and `qwen3.6-35b-a3b` generated about 12 tokens/s. At `c = 65536` with `threads = 8` it generated about 18 tokens/s.

### Qwen3.8 27B with 64k context and an 8-bit KV cache

`qwen3.8-27b` is a dense model, so every layer that does not fit in VRAM slows down every token. With `c = 65536` and `cache-type-k`/`cache-type-v` set to `q8_0`, the KV cache was larger than estimated, too few layers fit on the GPU, and generation ran at about 5 tokens/s. With `c = 49152`, a 4-bit KV cache (`q4_0`), and `fit-target = 384`, 61 of 66 layers fit on the GPU and it runs at about 8–11 tokens/s.

### Setting an old releaseDate on a model in opencode.jsonc

With `"releaseDate": "2025-08-05"` on `gpt-oss-20b` in `~/.config/opencode/opencode.jsonc`, the model appeared in the CLI (`opencode models`) but not in the desktop app's model picker. The desktop picker hides models it considers outdated. Removing the `releaseDate` line and running `opencode service restart` made it appear. Models with a 2026 release date were shown.

### Using the AppImage's .DirIcon as the menu icon

`.DirIcon` inside the AppImage is a symlink. Extracting only `.DirIcon` gives a link pointing to a path that was not extracted, so the menu showed no icon. Extracting `usr/share/icons/*` (Step 15) gives real PNG files.

## Troubleshooting

### `dlopen(): error loading libfuse.so.2`

**Cause:** The AppImage needs the FUSE 2 library, which Ubuntu 26.10 does not install by default.

**Fix:** Install it, then start `~/Applications/OpenCode.AppImage` again.

```bash
sudo apt install libfuse2t64
```

### `model name=<MODEL_ID> failed to load`

**Cause:** The llama.cpp server could not load the model named in the request. OpenCode or `curl` shows only this message, with the model's section name in place of `<MODEL_ID>`. The real reason is in the service log.

**Fix:** Read the log, then look for one of the errors below under its own heading in this Troubleshooting section.

```bash
journalctl --user -u llama-server -n 50 --no-pager | grep ' E '
```

### `exactly one out metadata, path_model, and file must be defined`

**Cause:** The server could not find the model file. Either the file was never downloaded, it was downloaded with a different `LLAMA_CACHE`, or `hf-repo`/`hf-file` in the model's section of `~/.config/llama.cpp/models.ini` do not match the downloaded file exactly. The presets use `offline = true`, so the server never downloads missing files itself.

**Fix:** Check that the file is in the model folder:

```bash
find "<MODEL_DIR>" -name '<FILE>'
```

Replace `<MODEL_DIR>` with the full path of the folder for model files and `<FILE>` with the `hf-file` value. If nothing is printed, download the file with `LLAMA_CACHE="<MODEL_DIR>" ~/.local/bin/llama download -hf <REPO> -hff <FILE> --no-mmproj`. If it is printed, correct the spelling of `hf-repo` and `hf-file`, then run `systemctl --user restart llama-server`.

### `K cache type q8_0 with block size 32 does not divide n_embd_head_k=8`

The number at the end depends on the model.

**Cause:** The 8-bit KV cache (`cache-type-k = q8_0` in the `[*]` section of `~/.config/llama.cpp/models.ini`) only works when the model's attention head size is a multiple of 32. Some small or unusual models have smaller heads. All five models in this guide work with `q8_0`.

**Fix:** Add these two lines to that model's own section in `~/.config/llama.cpp/models.ini` (not the `[*]` section), then run `systemctl --user restart llama-server`:

```ini
cache-type-k = f16
cache-type-v = f16
```

## Open questions

- Whether `loginctl enable-linger <USER>` makes `llama-server.service` start at boot before anyone logs in. Not tried; this guide's setup starts it at login.
- Whether a smaller Muse Glimmer 30B file (`IQ2_XXS`, about 10.75 GB, fully in VRAM) would be faster than 4–6 tokens/s and still good enough. Not tried.
- Whether the ROCm/HIP backend of llama.cpp is faster than Vulkan on the RX 6700 XT. Not tried.

## Sources and related links

- [llama.cpp server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md): router mode, `--models-preset`, `--models-max`, and the `/models` endpoints.
- [OpenCode config schema](https://opencode.ai/config.json): the `providers`, `compatibility`, `capabilities`, `variants`, and `limit` fields in `opencode.jsonc`.
- [ggml-org/gpt-oss-20b-GGUF](https://huggingface.co/ggml-org/gpt-oss-20b-GGUF): gpt-oss 20B model file.
- [unsloth/Qwen3.6-35B-A3B-GGUF](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF): Qwen3.6 35B-A3B model file.
- [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF): Qwen3.8 27B model file.
- [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B): recommended sampling settings for Qwen models.
- [ggml-org/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-GGUF](https://huggingface.co/ggml-org/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-GGUF): Nemotron 3.5 Lightning model file.
- [unsloth/Muse-Glimmer-30B-GGUF](https://huggingface.co/unsloth/Muse-Glimmer-30B-GGUF): Muse Glimmer 30B model file.

## Changelog

- 2026-11-07: First version.
