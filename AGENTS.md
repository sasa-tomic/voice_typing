# Agents Guide: Voice Typing Project

## Overview
This repository contains a collection of bash scripts for privacy-focused, offline voice typing. It leverages `whisper` or `whisper.cpp` for speech-to-text and `ydotool` to type the transcribed text into the active window.

## Core Components

- **`voice_typing`**: A standalone script that runs the `whisper` CLI for every audio clip. Best for occasional use and low resource consumption.
- **`voice_client_local`**: A client/server setup where a local `whisper.cpp` server is managed as a user service (`whisper.service`). Faster than `voice_typing` but uses more resources.
- **`voice_client`**: A client script designed to connect to a remote `whisper.cpp` server.
- **`llama_edit`**: An optional text-correction utility that uses an LLM (via `llama.cpp` server) to refine transcribed text (e.g., correcting "um", "uh", "oops, I mean..." or grammar).

## Modes of Operation

| Mode | Script | Backend | Connection Type |
| :--- | :--- | :--- | :--- |
| **Standalone** | `voice_typing` | `whisper` CLI | Local process |
| **Local Client/Server** | `voice_client_local` | `whisper.cpp` | Local `whisper.service` |
| **Remote Client/Server** | `voice_client` | `whisper.cpp` | Networked server |

## Architecture & Patterns

### Producer-Consumer Pattern
All main scripts use a FIFO-based producer-consumer architecture:
1. **Producer**: A background loop records audio using `sox` (`rec`) and writes the filename to a named pipe (FIFO).
2. **Consumer**: A foreground loop reads filenames from the FIFO, processes them (transcription), and then types the result.

### Service Management
The client/server scripts rely on `systemd` user services:
- `whisper.service`: Manages the `whisper.cpp` server.
- `llama.service`: Manages the `llama.cpp` server for `llama_edit`.

### Text Injection
Text is injected into the OS using `ydotool`. This requires the `ydotoold` daemon to be running and the `YDOTOOL_SOCKET` environment variable to be correctly set.

## Essential Commands

### Running the tools
```bash
# Standalone mode
./voice_typing

# Standalone mode with LLM text correction
./voice_typing -flow

# Local client/server mode
./voice_client_local

# Local client/server mode with LLM text correction
./voice_client_local -flow

# Remote client/server mode
./voice_client

# Remote client/server mode with LLM text correction
./voice_client -flow
```

### Testing `llama_edit`
```bash
./llama_edit "Your uncorrected text here"
```

## Dependencies & Requirements

- **Audio**: `sox`, `lame`, `ffmpeg`
- **Transcription**: `whisper` (OpenAI) or `whisper.cpp`
- **Text Injection**: `ydotool`
- **Data Processing**: `jq`, `curl`

## Gotchas & Troubleshooting

### `ydotool` Permissions
If you encounter `failed to connect socket '/tmp/.ydotool_socket': Permission denied`, ensure:
1. The user is in the `input` group.
2. `ydotoold` is running.
3. `export YDOTOOL_SOCKET=/tmp/.ydotool_socket` is in your `.bashrc`.
4. You may need `sudo chmod +s $(which ydotool)`.

### Silence Detection
The `rec` command uses silence detection thresholds. If transcription doesn't start or stops too early:
- Adjust the silence thresholds in the `rec` command (e.g., `silence 1 0.2 4% 1 1.0 2%`).
- Increasing the percentage (e.g., `6%` or `8%`) makes the detector more tolerant of background noise.

### Server Configuration
- **`voice_client`**: By default, it targets `http://127.0.0.1:7777`. If using a remote server, you **must** edit the script to point to the correct IP/hostname.
- **`llama_edit`**: Targets `http://127.0.0.1:8087/v1/chat/completions`. Ensure your `llama.service` is configured to listen on this port.

### Transcription Noise
If the output contains "Thanks for watching!" or other artifacts from the model, the scripts include logic to filter these out based on string matching or length checks.
