# Chord Ear

Single-file web app che riconosce accordi di pianoforte (e altri strumenti polifonici) in tempo reale dal microfono del browser.

**Live demo**: https://maxturazzini.github.io/aimax-chord-ear/

Nessuna dipendenza esterna, nessuna build: tutto in [index.html](index.html) (HTML + CSS + JS).

## Come funziona

Pipeline: microfono → Web Audio API (FFT 16384) → chromagram 12 pitch class con harmonic subtraction → template matching su 16 tipi di accordo → hysteresis → UI.

Documentazione completa dei parametri, dei preset e della pipeline in [CLAUDE.md](CLAUDE.md).

## Esecuzione locale

```bash
python3 -m http.server 8000
# http://localhost:8000/
```

Safari non supporta `getUserMedia` da `file://`, serve un http server locale.

## Licenza

MIT — by [Max Turazzini](https://maxturazzini.com) / AI, MAX
