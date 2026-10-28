# sailOS
AI-native microkernel operating system based on RedoxOS and sail-framework.

> **Tandem LLM Engineering & System Design Guide**: See [LLM_GUIDE.md](LLM_GUIDE.md) for the complete architecture detailing the speech-to-speech visual shell, the camera tool-calling contract, the [Are.na](https://www.are.na)-inspired research repertoire, and multi-agent development guidelines.

## Near-Kernel Realtime Speech-to-Speech Subsystem

sailOS implements the [huggingface/speech-to-speech](https://github.com/huggingface/speech-to-speech) pipeline **locally in pure Rust at near-kernel level** via native Redox OS Schemes and Orbital windowing.

### 1. The `voice:` Microkernel Scheme (`schemes/sail-voice`)

The `voice:` scheme daemon registers directly with the Redox microkernel and exposes local pipeline endpoints:

- `voice:vad` — Raw 16kHz PCM audio stream in, [silero-vad](https://github.com/snakers4/silero-vad) turn-taking & confidence score out (<3ms detection).
- `voice:stt` — Streaming speech-to-text token output ([Qwen3-ASR](https://huggingface.co/Qwen/Qwen3-ASR-1.7B) / local whisper streaming).
- `voice:vlm` — Multimodal vision-language reasoning & local kernel tool routing (`os:status`, `proc:list`, `fs:query`, `vision:frame`). Zero external APIs.
- `voice:tts` — Real-time neural speech synthesis streaming ([Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) / [Qwen3-TTS](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice)).
- `voice:orb` — Real-time visualizer telemetry (RMS level, sinusoidal wave phase, 8-band audio spectrum, conversational turn data).
- `voice:control` — Microkernel hardware tier profiles, mute toggle, and turn control.

### 2. Native Redox Orbital Desktop Application (`apps/sail-orb`)

A native Redox OS desktop application that creates an Orbital window and communicates with `voice:orb`:

- Renders dynamic concentric ripple rings and the glowing audio orb in real time.
- Connects directly to the kernel scheme via standard Redox file descriptors.
- Displays live transcripts of user speech, assistant speech, and kernel tool badges (`YOU`, `TOOL [os:status]`, `ASSISTANT`).
- Provides native mouse click interaction for the microphone and stop controls.

### 3. Local Hardware Profiles (Fitting GPU & CPU Constraints)

| Stage | Reference | Balanced (8GB–12GB VRAM) | Ultra-Lightweight (4GB–6GB VRAM / CPU) |
| :--- | :--- | :--- | :--- |
| **VAD** | `snakers4/silero-vad` | `silero-vad` (Rust/ONNX, ~2MB) | `silero-vad` (Rust/ONNX, ~2MB) |
| **STT** | `Qwen3-ASR-1.7B` | `Qwen3-ASR` (int8, ~1.1GB) | `whisper-tiny` / `faster-whisper` (~150MB) |
| **VLM/Reasoning** | [`yuxinlu1/gemma-4-12B-agentic-GGUF`](https://huggingface.co/yuxinlu1/gemma-4-12B-agentic-fable5-composer2.5-v2-3.5x-tau2-GGUF) | `gemma-4-12B-agentic` (Q4_K_M GGUF, ~7.2GB) | `sail::HybridLm` MoE (1.8B active, ~2.1GB) |
| **TTS** | `Qwen3-TTS-1.7B` | `Kokoro-82M` (82M params, ~300MB) | `Kokoro-82M` (82M params, ~250MB) |
| **Total**| **Local GGUF/MoE** | **~7.5 GB RAM / VRAM** | **~2.5 GB RAM / CPU** |

### Fast Iteration & Hot-Reload (~1s to 2s)

Instead of rebuilding the entire OS on every code change, use the fast-iteration script:

```bash
./scripts/hotreload.sh
```

- **Mode 1 (Host Watch)**: Auto-recompiles and restarts the desktop orb in ~1.5s on every file save.
- **Mode 2 (Live Disk Injection)**: Mounts `harddrive.img` via FUSE and swaps the binary in ~2s (no full OS rebuild).
- **Mode 3 (Live VM Sync)**: Pushes updated binaries directly into a running VirtualBox instance over SSH on port 2222 with **zero reboots**.

See [docs/hot-reload-iteration.md](docs/hot-reload-iteration.md) for full details.

### Building the Full OS & Running on VirtualBox

Complete instructions are available in [docs/build-and-run-virtualbox.md](docs/build-and-run-virtualbox.md).

1. Build the desktop image:
   ```bash
   make all
   ```
2. Convert and launch automatically in VirtualBox:
   ```bash
   make virtualbox
   ```
   Or convert manually using the helper script:
   ```bash
   ./scripts/convert_to_vbox.sh
   ```

Build both native binaries using the local offline cargo workspace:

```bash
cargo build --offline --workspace
```

Run the `voice:` scheme daemon:

```bash
./target/debug/sail-voice-scheme balanced_mid_tier
```

Run the native Redox Desktop Orb application:

```bash
./target/debug/sail-orb
```
