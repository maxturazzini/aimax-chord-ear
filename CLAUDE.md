---
document: Chord Ear - Riconoscitore di accordi in-browser
context: Mini-app standalone per il riconoscimento di accordi da microfono
date: 2026-04-18
updated: 2026-04-18
author: miniMe
status: active
tags: [audio, web-audio-api, chord-detection, chromagram, dsp]
---

# Chord Ear

App single-file che riconosce accordi di pianoforte (e altri strumenti polifonici) in tempo reale dal microfono del browser.

Tutto in [index.html](index.html): HTML + CSS + JS in un unico file, nessuna dipendenza esterna (solo font Google).

## Come funziona

### Pipeline

```
Microfono
   ↓ MediaStream
Web Audio API (AudioContext)
   ↓
AnalyserNode (FFT, fftSize=16384 → ~2.7Hz/bin @ 44.1kHz)
   ↓ getFloatFrequencyData (dB)
buildChromagram()
   ├─ noise floor (mediana + PARAMS.noiseFloor dB)
   ├─ proiezione su 12 pitch class con peso triangolare (cents)
   ├─ octave decay per ridurre bias verso armoniche acute
   └─ harmonic subtraction (sottrae H3/H5/H7 delle fondamentali)
   ↓ Float32Array[12] normalizzato
chromaHistory (buffer circolare PARAMS.chromaSmooth frame)
   ↓ media temporale
matchChord()
   ├─ template matching su 16 tipi di accordo × 12 root
   ├─ score = matchedEnergy/N - α·extraEnergy - β·missingCount
   └─ prior configurabili per triadi / estesi
   ↓ top candidate
Hysteresis (confirm frames + change delta)
   ↓
UI (chord name + mini piano + note pills + chromagram bars + history)
```

### Funzioni chiave

| Funzione | Cosa fa | Riga |
|----------|---------|------|
| `buildChromagram(dataFloat, sr, fftSize)` | FFT dB → chromagram 12-vec con harmonic subtraction | ~450 |
| `matchChord(chroma)` | Template matching con penalità, ritorna top-3 candidati | ~510 |
| `startDetection()` | Loop 80ms: chromagram → smoothing → match → hysteresis → UI | ~780 |
| `averageChroma()` | Media mobile sul buffer `chromaHistory` | ~770 |
| `updateChromaPanel(chroma)` | Visualizza 12 barre pitch class in tempo reale | ~870 |

### Parametri (tutti esposti in UI)

Definiti in `DEFAULT_PARAMS` nel file. La UI `<details id="settings">` contiene uno slider per ogni parametro con tooltip esplicativo.

**Stabilità (anti-flicker)**
- `tickMs` (20-300): intervallo tra campionamenti in ms. Più basso = più reattivo ma più rumoroso. Default 80ms.
- `chromaSmooth` (1-20): quanti frame mediare. Finestra effettiva = `tickMs × chromaSmooth` (default 80×10 = 800ms)
- `confirmFrames` (1-10): frame consecutivi necessari per confermare un nuovo accordo
- `changeDelta` (0-0.5): gap score minimo per switch immediato

**Silenzio & rumore**
- `energyGate` (0-30): energia totale minima per fare detection
- `peakDb` (-100..-20): picco spettrale minimo in dB
- `noiseFloor` (0-25): dB sopra la mediana della banda utile

**Chromagram**
- `h3`, `h5`, `h7`: pesi harmonic subtraction (quinta, terza, settima)
- `octaveDecay` (0.5-1): decay per ottava sopra C4 (1 = nessun decay)

**Chord matching**
- `presentThreshold` (0-0.8): soglia chroma per considerare una pc "presente"
- `activeThreshold` (0-0.8): soglia per UI (note pills + piano lit)
- `alpha` (0-1.5): penalità note extra nello score
- `beta` (0-1): penalità note mancanti
- `priorTriad` (0.8-1.3): bonus per triadi maggiori/minori
- `priorExt` (0.7-1.2): malus per accordi estesi (9, 6, add9)

### Preset

Built-in (non modificabili):
- **Default** — valori iniziali, ambiente medio
- **Ambiente silenzioso** — gate bassi, smoothing ridotto
- **Ambiente rumoroso** — gate aggressivi, smoothing lungo
- **Piano acustico** — harmonic subtraction più marcata
- **Voce / Strumento monofonico** — bias forte verso triadi
- **Reattivo (demo)** — smoothing minimo, cambia velocemente
- **Lento / stabile** — `tickMs:150`, `chromaSmooth:12`, `confirmFrames:5`, `changeDelta:0.18` — detection rilassata, minimo flicker

I preset custom si salvano in `localStorage` con chiave `chordear_user_presets_v1`.
L'ultima configurazione attiva è persistita in `chordear_last_params_v1` e ricaricata all'apertura.

## Come modificare

### Aggiungere un tipo di accordo
Editare l'array `CHORD_TYPES` (cerca la stringa nel file):
```js
{ name: 'add11', fullName: 'Add11', intervals: [0,4,7,17] },
```

### Aggiungere un preset built-in
Editare `BUILTIN_PRESETS`:
```js
'Il mio preset': {
  ...DEFAULT_PARAMS,
  energyGate: 6,
  chromaSmooth: 8,
},
```

### Aggiungere un nuovo parametro
1. Aggiungi la chiave in `DEFAULT_PARAMS`
2. Aggiungi lo slider HTML nel blocco `<details id="settings">`
3. Aggiungi una riga in `SLIDER_MAP` (`[key, sliderId, labelId, formatFn]`)
4. Usa `PARAMS.nomeChiave` dove serve nella logica

Il resto (wire-up, persistence, preset save/load) è automatico.

## Debug

- `?debug=1` in query string → console.log di top-3 candidati e messaggi energy gate
- Il **Chromagram panel** sotto le note è il miglior strumento di tuning: mostra in tempo reale l'energia per pitch class dopo harmonic subtraction. Se vedi energia forte su note che non stai suonando → aumenta `h3/h5/h7` o `noiseFloor`.

## Troubleshooting

| Sintomo | Probabile causa | Azione |
|---------|-----------------|--------|
| Cambia accordo al minimo rumore | Energy gate troppo basso | Alza `energyGate` e `peakDb` |
| Flicker tra accordi vicini (C ↔ Cmaj7) | Smoothing troppo corto | Alza `chromaSmooth` e `confirmFrames` |
| Non riconosce accordi deboli | Soglie troppo alte | Abbassa `presentThreshold`, `energyGate` |
| Confonde root (C con F o G) | Armoniche non soppresse | Alza `h3`, `h5` |
| Scambia triadi con estesi (es. D con Dmaj7) | Prior sbagliati | Alza `priorTriad`, abbassa `priorExt` |
| Sotto 150Hz (bassi) imprecisio | Risoluzione FFT limitata | Limite fisico dell'fftSize, prova a suonare un'ottava sopra |

## Come testare

```bash
cd "/Users/maxturazzini/Library/CloudStorage/OneDrive-Personale/miniMe/projects/chord-ear"
python3 -m http.server 8000
# http://localhost:8000/
```

Oppure via Artifact Viewer VS Code:
```
vscode://maxturazzini.aimax-viewer/openBrowser?http://127.0.0.1:3134/projects/chord-ear/index.html
```

**Test manuali consigliati:**
1. Triadi: C, Am, F, G → stabili ≥ 2s ciascuno
2. Estesi: Cmaj7, Dm7, G7 → riconosciuti correttamente
3. Inversioni: C/E, G/B → la root deve rimanere corretta
4. Silenzio → display `—`, nessun falso accordo
5. Parlato / ventola → display `—` (energy gate deve escluderli)

## Struttura file

```
chord-ear/
├── index.html      # App completa (HTML + CSS + JS single file)
└── CLAUDE.md       # Questa documentazione
```

## Note tecniche

- **Safari**: `getUserMedia` non funziona da `file://`, serve HTTP server locale
- **AudioContext**: richiede resume() esplicito dopo gesto utente (gestito in `toggleListening`)
- **MediaStream cleanup**: `stream.getTracks().forEach(t => t.stop())` è necessario su Safari per rilasciare l'indicatore microfono
- **fftSize 16384**: supportato da tutti i browser moderni (max spec 32768)
- **Detection interval**: 80ms fisso, non dipende da `requestAnimationFrame` (che è legato al refresh rate)

## Roadmap (non prioritario)

- [ ] Slash chords nel display (es. C/E)
- [ ] Export storia accordi (MIDI o testo)
- [ ] Riconoscimento tonalità / key detection
- [ ] Versione PWA installabile
- [ ] Test automatici con file .wav pre-registrati
