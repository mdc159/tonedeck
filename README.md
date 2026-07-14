# ToneDeck

**A touch-first, native Linux guitar amp powered by Neural Amp Modeler Architecture 2 (A2).**

Plug your guitar into any class-compliant USB audio interface, load a `.nam` capture, and play — with a big, finger-friendly UI made for touchscreens.

## Why

- Neural Amp Modeler's A2 architecture (June 2026) delivers state-of-the-art amp modeling at very low CPU cost — and it's MIT-licensed.
- There is no official NAM app for Linux, and existing Linux hosts have desktop-mouse GUIs.
- ToneDeck fills the gap: a real-time C++ engine on PipeWire + a touch-first web UI.

## Architecture

```
guitar → USB interface (dry DI) → PipeWire → ToneDeck engine (C++)
   → NeuralAudio (NAM A1/A2 inference) → PipeWire → USB interface → speakers
                        ↑
        touch web UI (localhost) via WebSocket
```

- **Engine**: headless C++ process; PipeWire `pw_filter` node; allocation-free audio path; off-thread `.nam` model loading with atomic swap.
- **UI**: single-page web app served by the engine; big knobs and model cards sized for fingers; works full-screen/kiosk on a touchscreen — or from any browser on your network.

## Status

Early development. Design spec in `docs/`.

Reference hardware: Dell Precision M6800 (17" touchscreen) + Line 6 POD Express as USB interface. Designed to scale up to modern desktops and down to Raspberry Pi.

## License

MIT. Built on [NeuralAudio](https://github.com/mikeoliphant/NeuralAudio) (MIT) and the [Neural Amp Modeler](https://github.com/sdatkinson/NeuralAmpModelerCore) ecosystem (MIT).

Amp captures are available at [TONE3000](https://www.tone3000.com) (per-tone licenses).
