# ToneDeck v1 — Design Spec

**Date:** 2026-07-13
**Status:** Approved by user (brainstorming session, 2026-07-13)
**Repo:** https://github.com/mdc159/tonedeck

## 1. Purpose & success criteria

ToneDeck is a touch-first, native Linux guitar amp application powered by Neural Amp Modeler Architecture 2 (A2).

**v1 is a success when:** a guitar plugged into a Line 6 POD Express (acting as USB audio interface) plays through a user-selected `.nam` amp capture processed live by ToneDeck on the Linux PC, with output audible through the POD's outputs (or laptop audio), at playable latency (~6–10 ms round trip), controlled from a touch-friendly full-screen UI on the 17" touchscreen.

**Explicitly out of scope for v1:** presets, spectrum/visualizers, TONE3000 API integration, multi-effect chains, IR cab loader, looper, recording, resampling (48 kHz fixed), vocals. The architecture leaves room for all of these (see §4 slot chain, §6 protocol).

## 2. Reference environment (floor, not target)

- Dell Precision M6800, Ubuntu 24.04, PipeWire, 17" touchscreen (Wacom TPC4001).
- Line 6 POD Express Guitar as USB interface: class-compliant, fixed 24-bit/48 kHz.
  - Capture ch 1/2 = POD-processed L/R; **ch 3/4 = dry DI** (ToneDeck input).
  - Playback ch **1/2 = direct monitor out** (ToneDeck output); ch 3/4 = re-amp into POD.
  - USB is data-only: pedal needs batteries/9V. Max 2 m cable, no hubs.
- Portability principles: no hardcoded device names (interface chosen via UI/PipeWire); plain portable C++; scales up to modern desktops and down to Raspberry Pi (A2-Lite exists for exactly that). Known thermal constraint on the M6800 (dead fans) → engine exposes DSP load % and a quality toggle (A2-Full ↔ A2-Lite).

## 3. Architecture overview

Two processes:

```
guitar → POD input ─USB─→ [capture ch 3/4: dry DI]
                                   │
                        ┌──────────▼──────────┐
                        │  ENGINE (C++)        │  real-time; never blocks/allocates in callback
                        │  PipeWire pw_filter  │
                        │  in → slot chain     │  NeuralAudio inference (.nam A1+A2)
                        │  → out               │
                        └──────┬───────▲───────┘
                               │ WebSocket + HTTP (localhost:9600)
                        ┌──────▼───────┴───────┐
                        │  UI (browser, touch)  │  single page, served by engine
                        └──────────────────────┘
                                   │
[playback ch 1/2] ─USB─→ POD output → speaker/headphones
```

- Engine runs standalone; audio never depends on the UI. UI death/disconnect does not affect sound.
- Model files live in a watched folder: `~/.local/share/tonedeck/models/` (drop `.nam` files in; they appear in the UI).

## 4. Engine (C++)

**Audio node.** One PipeWire `pw_filter` node: 1 input port (mono dry DI), 2 output ports (stereo). Session manager links input ← POD capture ch 3, outputs → POD playback 1/2 by default; UI offers an output-device picker (POD default, laptop audio fallback). Unlinked ports when POD absent — engine keeps running. `PW_FILTER_FLAG_RT_PROCESS`; RT priority via rtkit (automatic on Ubuntu 24.04). Request `node.latency = 128/48000` initially; tune down after xrun testing (start 256, walk down using pw-top).

**Fixed 48 kHz, no resampler in v1.** POD is hardwired 48 kHz and NAM models are typically 48 kHz. If a loaded model declares a different rate, warn in UI and play anyway (documented limitation).

**Callback path (allocation-free, lock-free):**
dry in → input gain → NeuralAudio `model->Process()` → output gain → soft-clip guard → write both outputs → peak meters via atomics.

**Slot chain.** Internally the processing is a fixed array of processor slots: [input gain] → [NAM slot] → [output gain]. v1 hardcodes this chain, but the slot abstraction is the extension point for future effects (IR cab, delay, etc.) — adding a slot must not require reworking the callback.

**Inference engine: NeuralAudio (MIT, mikeoliphant)** — vendored as a git submodule. Loads .nam A1 + A2 (internal optimized A2 since v0.1.2), keras/RTNeural formats; allocation-free process after buffer-size config; quality scaling (Full/Lite). Fallback knowledge: official NeuralAmpModelerCore v0.5.4 (MIT) is the reference implementation if NeuralAudio ever becomes a problem.

**Model loading (never on audio thread).** Loader thread: parse/validate `.nam` → build model → prewarm → publish via atomic pointer swap; old model destroyed on loader thread. Audio continues through the swap. Errors (corrupt/unsupported file) reported to UI; current model untouched; a half-loaded model is never published.

**Control plane.** Embedded HTTP + WebSocket server on `localhost:9600` (single-header/small C++ lib, e.g. uWebSockets or similar). Serves UI static files + JSON protocol. Control → audio thread via lock-free SPSC queue.

**Health.** Engine computes DSP load % (callback time ÷ quantum), counts xruns, and reports both at ~10 Hz.

## 5. UI (touch-first web app)

Single page, no navigation, dark stage-friendly theme, min ~80 px touch targets. Served by the engine; run full-screen/kiosk on the touchscreen (also reachable from any browser on the LAN — free remote-UI bonus).

Components:
- **Model selector** (centerpiece): large touch-scrollable cards from the models folder — name + gear metadata parsed from the `.nam` file. Tap to load; loading indicator; current model name large at top.
- **Two big knobs**: input gain, output gain. Drag-up/down rotary gesture; double-tap resets to 0 dB.
- **Meters & health strip**: in/out level bars + clip indicators, DSP load %, xrun count, latency readout, POD connection dot.
- **Output device picker**: small dropdown (POD default / laptop audio).
- **Quality toggle**: A2-Full ↔ A2-Lite (thermal headroom lever).

Tech: plain HTML/CSS/JS, zero/minimal build step, canvas-based knobs/meters. No framework lock-in for v1.

## 6. Protocol (WebSocket, JSON)

Client → engine: `load_model {path}`, `set_param {id, value}` (input_gain, output_gain, quality), `set_output {device}`, `get_state`.
Engine → client: `state` (current model, params, devices) on connect/change; `meters` broadcast ~10 Hz (in/out peak, dsp_load, xruns); `error {message}`; `model_loaded {name, metadata}`.
Versioned envelope (`{v:1, type, ...}`) so future features (presets, chains) extend without breaking old UIs.

## 7. Error handling

- **POD unplugged mid-session**: ports unlink; engine keeps running; UI shows "interface disconnected"; relink on reconnect via persistent link rules. No crash/restart.
- **Corrupt/unsupported `.nam`**: loader catches, UI shows error, current model keeps playing.
- **Overload/xruns** (incl. thermal throttling): counted, surfaced, self-recovering; visible load meter guides switch to A2-Lite.
- **UI disconnect**: audio unaffected; WebSocket auto-reconnect with visible state.
- **No model loaded** (first run): pass-through dry signal at unity gain, UI prompts to add/pick a model.

## 8. Testing

- **Unit (off-hardware)**: model loader good/bad files, atomic swap lifecycle, gain/clip math, protocol handlers.
- **Allocation discipline**: allocation-tracking test (NAM-core style) failing if audio-callback path allocates.
- **End-to-end offline**: WAV → engine chain → WAV, compared against direct NeuralAudio output.
- **Live checks (scripted)**: loopback latency measurement through the POD; xrun soak test at target quantum.
- **UI**: manual touch testing + scripted WebSocket protocol client. No browser test framework in v1.

## 9. Licensing & sharing

- ToneDeck: MIT. Dependencies: NeuralAudio (MIT), Eigen (MPL2, unmodified), nlohmann/json (MIT), PipeWire (client lib, MIT). Clean for public distribution, source and binaries.
- Captures are user-supplied; TONE3000 per-tone licenses forbid redistributing most `.nam` files — ToneDeck ships none.

## 10. Key implementation references

- Research doc: Obsidian `03-Resources/Research/NAM-A2-Guitar-App-Research-2026-07-13.md` (full API/build/latency details, host landscape, gotchas).
- Patterns to copy: LV2 worker-style off-RT model load (neural-amp-modeler-lv2); ResamplingNAM (official plugin) if resampling ever needed; stompbox for engine/UI split precedent; NeuralRack for chain architecture reference.
- PipeWire duplex filter example: https://docs.pipewire.org/audio-dsp-filter_8c-example.html
- Gotchas: Eigen alignment on some compilers (`EIGEN_MAX_ALIGN_BYTES 0` escape hatch); NAM core version metadata is stale in tags; `NAM_ENABLE_A2_FAST` must be defined if bypassing CMake (NeuralAudio manages its own equivalent).
