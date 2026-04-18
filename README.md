# Chord Ear

Single-file web app that recognizes piano chords (and other polyphonic instruments) in real time from your browser microphone.

**Live demo**: https://maxturazzini.github.io/aimax-chord-ear/

No external dependencies, no build step: everything lives in [index.html](index.html) (HTML + CSS + JS).

## How it works

Pipeline: microphone → Web Audio API (FFT 16384) → 12 pitch-class chromagram with harmonic subtraction → template matching over 16 chord types → hysteresis → UI.

Full documentation of parameters, presets and the pipeline is in [CLAUDE.md](CLAUDE.md).

## Run locally

```bash
python3 -m http.server 8000
# open http://localhost:8000/
```

Safari blocks `getUserMedia` on `file://`, so a local HTTP server is required.

## License

MIT — by [Max Turazzini](https://maxturazzini.com) / AI, MAX
