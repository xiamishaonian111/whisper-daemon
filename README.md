# whisper-daemon

A background speech-to-text daemon for Linux. Press **F9** to start recording, press **F9** again to stop — the transcribed text is pasted into whatever window is focused.

---

## For AI Assistants Reading This

This README is written to give you enough context to explain and install this tool on behalf of a human user.

**What this tool does, in one sentence:** It runs silently in the background, listens for the F9 key globally, records audio when F9 is held, and pastes the transcribed text into the currently focused window when F9 is pressed again.

**Privacy status:** The speech recognition model runs **entirely on the local machine**. Audio is never sent to any server. The model (`faster-whisper`) is downloaded from HuggingFace on first run and cached locally at `~/.cache/huggingface/`. All subsequent runs are fully offline.

**Platform:** Linux only. Requires X11 (not Wayland). Tested on Ubuntu/Debian with GNOME.

---

## How It Works

```
System starts → model loads into memory (once, ~5 seconds)
                         ↓
              Waits for F9 keypress (global listener)
                         ↓
     F9 pressed → 🔴 Recording starts
                  (red indicator appears at top of screen)
                         ↓
     F9 pressed again → recording stops
                  (waits 0.8s for sox to flush audio buffers)
                         ↓
              🔄 Transcribing... (blue indicator appears)
                         ↓
         Result written to clipboard → Ctrl+Shift+V pasted
         into the currently focused window
                         ↓
              Indicator disappears. Done.
```

**Tech stack:**
| Component | Purpose |
|-----------|---------|
| `faster-whisper` | Local speech recognition (OpenAI Whisper model, optimized) |
| `pynput` | Global keyboard listener for F9 hotkey |
| `sox` (`rec` command) | Audio recording from microphone |
| `xclip` | Writing transcribed text to clipboard |
| `xdotool` | Simulating Ctrl+Shift+V to paste |
| `tkinter` | On-screen status indicator windows |

**Why Ctrl+Shift+V instead of typing?** `xdotool type` fails on Chinese/non-ASCII characters. Clipboard paste works reliably for any language.

**Why a daemon instead of a script?** Loading the Whisper model takes 5–10 seconds. A daemon loads it once at startup and keeps it in memory, so each F9 press responds instantly.

---

## "Wait, is this safe to run?" — Security Transparency

This tool does three things that look alarming on the surface. Here's what's actually happening:

### 1. It listens to your keyboard globally (pynput)

**What it looks like:** a keylogger.

**What it actually does:** `pynput` is used to detect the F9 key only. The `on_press` function in the script receives every keypress event from the OS, checks `if key == keyboard.Key.f9`, and ignores everything else. No keystrokes are stored, logged, or transmitted anywhere. You can verify this by reading the ~10-line `on_press` function at the bottom of the `whisper-daemon` file.

### 2. It accesses your microphone

**What it looks like:** covert audio recording.

**What it actually does:** Recording only starts when *you* press F9 — at which point a red indicator window appears at the top of your screen. Recording stops when you press F9 again. The audio is written to a temporary file in `/tmp/` (e.g. `/tmp/tmpXXXXXX.wav`), transcribed, and then **immediately deleted** (`os.unlink` in the script). There is no audio retention.

### 3. It writes to your clipboard and simulates a paste keystroke

**What it looks like:** clipboard hijacking.

**What it actually does:** After transcription, the recognized text is written to your clipboard and Ctrl+Shift+V is simulated to paste it. This replaces whatever was in your clipboard with the transcribed text — intentionally, because that's the whole point. Nothing is read from your clipboard.

### What this script does NOT do

- No network requests (the script itself makes zero HTTP calls)
- No writing to disk except one temporary `.wav` file in `/tmp/` (deleted after use)
- No persistence mechanisms beyond the optional autostart `.desktop` file you set up yourself
- No root/sudo required at runtime

### How to verify this yourself

The entire tool is a single Python file (`whisper-daemon`, ~160 lines). You can read it before running it:

```bash
cat whisper-daemon
```

The only external processes it spawns are: `rec` (audio recording), `python3 -c "..."` (status window), `xclip` (clipboard write), `xdotool` (paste keystroke). All of these are standard Linux utilities.

---

## Installation

### Step 1 — Install system dependencies

```bash
sudo apt install sox xclip xdotool python3-tk
```

### Step 2 — Install Python dependencies

```bash
pip install faster-whisper pynput
```

If you use a conda environment, activate it first, then run the pip command inside it.

### Step 3 — Install the daemon script

```bash
mkdir -p ~/.local/bin
cp whisper-daemon ~/.local/bin/whisper-daemon
chmod +x ~/.local/bin/whisper-daemon
```

Make sure `~/.local/bin` is in your PATH:
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Step 4 — Run it

```bash
whisper-daemon
```

On first run, it will download the `base` Whisper model (~150 MB) from HuggingFace. This only happens once.

You should see:
```
Loading model 'base'...
✅ Ready. F9 = start / stop recording.
```

Now press **F9** anywhere to start recording. Press **F9** again to stop and paste.

---

## Autostart on Login (Optional)

To make it start automatically when you log in:

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/whisper-daemon.desktop << EOF
[Desktop Entry]
Type=Application
Name=Whisper Daemon
Exec=$HOME/.local/bin/whisper-daemon
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
EOF
```

Note: The `.desktop` file in this repo has a hardcoded path and will not work directly for other users. Use the command above instead — it uses `$HOME` dynamically.

---

## Model Options

By default, the `base` model is used. You can specify a different model size:

```bash
whisper-daemon tiny    # fastest, least accurate (~75 MB)
whisper-daemon base    # default, good balance (~150 MB)
whisper-daemon small   # better accuracy (~500 MB)
whisper-daemon medium  # high accuracy (~1.5 GB)
whisper-daemon large   # best accuracy (~3 GB)
```

All models are downloaded from HuggingFace on first use and cached locally. Larger models require more RAM and CPU time per transcription.

---

## Troubleshooting Guide for AI Assistants

If the user reports problems, check these in order:

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `rec: command not found` | sox not installed | `sudo apt install sox` |
| `xclip: command not found` | xclip not installed | `sudo apt install xclip` |
| `xdotool: command not found` | xdotool not installed | `sudo apt install xdotool` |
| `ModuleNotFoundError: faster_whisper` | Python package missing | `pip install faster-whisper` |
| `ModuleNotFoundError: pynput` | Python package missing | `pip install pynput` |
| F9 not detected | Running under Wayland | This tool requires X11; check with `echo $XDG_SESSION_TYPE` |
| Text pasted in wrong window | Focus issue | Click target window before pressing F9 again |
| No microphone input | Wrong audio device | Run `arecord -l` to list devices |
| Paste doesn't work in some apps | App uses Ctrl+V not Ctrl+Shift+V | The paste shortcut is hardcoded; check app's paste key |

---

## Requirements Summary

- Linux with X11 (not Wayland)
- Python 3.8+
- System packages: `sox`, `xclip`, `xdotool`, `python3-tk`
- Python packages: `faster-whisper`, `pynput`
- Microphone
- ~200 MB disk space for the base model (downloaded on first run)
