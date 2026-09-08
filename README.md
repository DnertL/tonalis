# Tonalis

A polyphonic tuner and chord recogniser for guitar and bass. One HTML file, no build step, no dependencies, no network access. Open it in a browser and it works.

Strum all six strings at once and each one is measured separately. Play a single string and it switches to single-string mode on its own. Play a chord and it tells you which chord you played, with fingering diagrams.

**Version 0.3** · [GPL-3.0-or-later](#license) · ~127 KB single file

---

## Screenshots

<p align="center">
  <img src="docs/screenshot.jpg" alt="All-strings mode: headstock with per-string status" width="270">
</p>

<!--
  To capture these:
  1. Serve the file locally:  python3 -m http.server 8000
  2. Open http://localhost:8000/tonalis.html in Chrome
  3. DevTools (F12) -> device toolbar (Ctrl+Shift+M) -> iPhone 14 Pro or 390x844
  4. Enable Tools -> Demo signal -> Full strum so the display shows live readings
  5. Ctrl+Shift+P -> "Capture screenshot"
  Save as docs/screenshot-tuner.png, -chords.png, -settings.png
-->

---

## Features

**Four modes**

| Mode | What it does |
|---|---|
| Automatic | Decides between single-note and multi-string analysis by itself |
| All strings | Strum everything; every string gets its own reading |
| Single string | Tap a string to lock it — even strings that are semitones off are still measured |
| Chords | Recognises the chord and shows up to three playable shapes |

**Instruments and tunings** — 37 presets across guitar (6, 7 and 8 string), bass (4, 5, 6 string), ukulele, banjo and mandolin. Standard, drop, open and modal tunings included.

**Chord engine** — 28 chord types across 12 roots. Shapes are not a hardcoded table; they are generated for whatever tuning is currently selected, filtered for playability (fret span, finger count, barre detection, no muted strings between sounding ones) and ranked so the common shapes come first.

**Extras** — reference tone on tap, chord library with playable shapes, metronome with tap tempo, live spectrum display, concert pitch 415–466 Hz, capo/transposition, German note names (H/B), left-handed view, English and German interface.

**Demo signal** — generates a detuned guitar internally so you can try everything without a microphone.

---

## Getting started

Download `tonalis.html` and open it. That is the whole installation.

One caveat: **browsers block microphone access on `file://` URLs.** Opening the file by double-clicking will show the interface, but the microphone button will fail. Two ways around it:

```bash
# serve locally
python3 -m http.server 8000
# then open http://localhost:8000/tonalis.html
```

Or use **Tools ▸ Demo signal**, which needs no microphone at all.

For real tuning, serve over HTTPS or localhost. Anything hosted on GitHub Pages works out of the box.

### Getting good readings

- Leave **Use raw signal** enabled (it is by default). It disables the browser's echo cancellation, noise suppression and automatic gain — all three distort pitch and will make readings wander.
- Play at a normal volume and let the note ring. The analysis window is about a third of a second.
- Everything runs locally. No audio leaves the device.

---

## How the detection works

The signal path is deliberately not a single algorithm — different problems need different tools.

**Frequency estimation.** An FFT with zero-padding gives the spectrum; peaks are refined by parabolic interpolation on the log magnitude. Rather than reading the fundamental alone, the first few harmonics are fitted against the model for stiff strings, `f_h = h·f₀·√(1 + B·h²)`. Real strings are slightly inharmonic, so their upper harmonics run sharp; taking them at face value biases the reading sharp. The regression solves for the true fundamental instead. On synthetic single notes this cut the median error from 0.16 to 0.02 cents.

**Single note versus chord.** The two cases need different handling, so the mode decision must not depend on the harmonic logic it feeds. It uses the clarity value of the NSDF (McLeod pitch method) instead — an independent criterion. A single plucked string scores near 1.00, a full strum around 0.40, which separates cleanly.

**The hard part: harmonics.** The third harmonic of the low E string lands on the open B string, the fourth lands on the high E. A naive tuner reports strings that are not being played at all. Tonalis estimates the harmonic envelope of each detected string by regression across its own partials, and judges whether a peak on a higher string can be explained by the lower string alone. When it can, the string is rejected as a ghost; when the peak clearly exceeds the envelope, it is kept but flagged as *overlaid by a harmonic*, because in that case the two components merge in the spectrum and the reading is a blend.

**Chord recognition.** Notes are extracted iteratively: find the strongest pitch, subtract its modelled harmonic series, repeat. Subtraction matters rather than simple suppression — in Dsus4 the A sits exactly on the third harmonic of the D, and zeroing that region deletes a note that is genuinely there. Chord naming then matches the resulting pitch-class set plus the bass note against templates, which is why inversions come out as `G/B` rather than `G`.

---

## Accuracy

Measured against synthesised plucked-string signals — harmonic series with inharmonicity, weak fundamentals as phone microphones produce them, staggered onsets and added noise. **These are synthetic benchmarks, not measurements against a calibrated reference tuner**, so treat them as a description of the algorithm's behaviour rather than a specification.

| Case | Median error | Notes |
|---|---|---|
| Single note | 0.02 cents | max 0.11 cents over 60 runs |
| Full strum, six strings | 0.32 cents | all six detuned randomly ±45 cents |
| Bass, high inharmonicity | < 1 cent | holds up to B = 5·10⁻⁴ |
| Chord recognition | 88.9 % | 208 of 234 generated voicings |
| Mode decision | 0 errors | 60 runs, no misassigned string |

Runtime is about 8.5 ms per frame at a 16384-sample window.

Some of the remaining chord mismatches are genuine musical ambiguities rather than failures — a C6 without its fifth is also correctly called Am/C.

---

## Settings reference

Every parameter of the detection algorithm is exposed. Defaults were chosen from parameter sweeps, not guesswork.

| Setting | Default | Effect |
|---|---|---|
| Analysis window | 16384 | Longer is more precise, shorter reacts faster. 32768 doubles the cost for little gain |
| Window function | Hann | Best resolution/leakage compromise here |
| Capture range | ±170 ¢ | How far out of tune a string may be and still be recognised |
| Harmonics evaluated | 6 | More helps when the fundamental is weak |
| Harmonic tolerance | ±12 ¢ | Search window per partial. Narrowing this from 35 cut the 95th-percentile error from 16 to 3.6 cents |
| Harmonic rejection | very strict | See note below |
| Inharmonicity correction | on | The stiff-string regression described above |
| Single-note threshold | 0.90 | NSDF clarity required to switch modes |
| Noise gate | −46 dB | Ignores quiet disturbances |
| Minimum SNR | 8.5× | Peak against the noise floor |
| Minimum string level | 14 % | Share of the loudest string |

**On harmonic rejection:** at *very strict* ghost strings drop from 9 to 4 per test batch, but roughly one string in nine occasionally disappears from a full strum (it returns on the next stroke). If strings go missing too often for your taste, *normal* is the setting to change — it is the only one of the three strictness parameters that measurably affects this.

---

## Browser support

Needs Web Audio and `getUserMedia`. Works in current Chrome, Firefox, Safari and Edge, desktop and mobile. The interface is built mobile-first.

Settings persist via `localStorage` where available and fall back to memory where it is blocked, so the app also runs inside sandboxed iframes and in private browsing modes.

---

## Development

There is no build step — `tonalis.html` is the deliverable. The source is organised in two script blocks: a DSP core with no DOM dependencies, and the application layer.

The DSP core was developed and tested standalone under Node against synthesised guitar signals. The interface is covered by an integration test that drives the real application code against a minimal DOM and Web Audio stub, so the whole chain from signal to display runs headless.

---

## Changelog

**0.3**
- Rebuilt the headstock: tuners arranged left and right (3+3, 2+2, 3+2 depending on instrument)
- Fixed strings running to the wrong tuning pegs. The cause was that all pegs sat at the same distance from the centre line, which forces the string lines to cross. Peg offset now grows with distance from the nut, as on real instruments
- More realistic silhouette, tapering from top to nut; bushings, truss rod cover, pearl inlays, fretboard binding, string shadows
- Geometry is verified numerically — every string line is sampled for ordering violations across all string counts

**0.2**
- English and German interface, switchable in settings; English is the default
- Stricter defaults for noise gate, minimum SNR and harmonic rejection
- Author, version and licence metadata

**0.1**
- Initial release

---

## Licence

Copyright (C) 2026 Daniel Ertl <dnertlgh@proton.me>

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program. If not, see <https://www.gnu.org/licenses/>.

`SPDX-License-Identifier: GPL-3.0-or-later`

> **Note:** add a `COPYING` file containing the full GPLv3 text to this repository. The licence requires that a copy accompanies the program; the header in `tonalis.html` only points to it.

Parts of this project were generated with AI assistance.
