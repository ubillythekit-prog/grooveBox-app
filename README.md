# GrooveBox

A single-file, browser-based drum machine / step sequencer. No build step, no dependencies beyond one CDN script (`jsmidgen`, used only for MIDI export) — open `grooveBox-app.html` in a browser and it runs entirely client-side using the Web Audio API.

## What it does

- An 8-track × 16-step sequencer (kick, snare, hihat, tom, clap, perc, cymbal, cowbell) driven by a click grid.
- Adjustable tempo (40–200 BPM), swing, humanize, and master volume.
- Reverb and delay send effects.
- Six built-in patterns (4-on-Floor, House, Techno, Breakbeat, Trap, Drum & Bass) loadable with one click.
- Quantize (snap existing hits to a 1/16, 1/8, 1/4, or 1/2 grid) and Randomize.
- Export the current pattern as a Standard MIDI File (`.mid`) for use in a DAW.

## Core logic

### Sound generation — synthesized, not sampled
Each of the 8 drum voices is defined by a tiny waveform descriptor in `wf`:

```js
kick: { f: 150, d: .3, t: 'sine' }   // frequency (Hz), decay (s), oscillator type
```

`play(sound)` creates a fresh `OscillatorNode` + `GainNode` pair per hit, sets the oscillator's frequency/type from that descriptor, and applies an exponential gain envelope (`setValueAtTime` → `exponentialRampToValueAtTime(.01, ...)`) so each hit has a percussive attack-decay shape instead of a flat tone. Nodes are created and discarded per note — there's no polyphony limit beyond what the browser's audio graph can handle.

### Audio routing graph
```
oscillator → per-note gain ──┬──► dg (dry gain) ──────────────► mg (master gain) → destination
                              └──► wg (wet gain) → conv (convolver, reverb) ──►┘

                                    dwg (delay send) → del (DelayNode) ⇄ dfb (feedback gain) → mg
```
- `dg`/`wg` split each note between a dry path and a reverb (convolver) path; toggling Reverb crossfades between them (`dg.gain = 1` dry vs `dg.gain = .7` / `wg.gain = .3` when wet).
- The delay line (`del`) feeds back into itself through `dfb` (feedback gain) before rejoining the master bus, producing repeating echoes; toggling Delay just opens/closes the send gain `dwg`.
- Everything sums into `mg` (master gain, tied to the Volume slider) before hitting `ctx.destination`.

### Sequencer state and playback loop
State is a plain 2D array: `seq[8][16]`, `1`/`0` per cell. The grid UI (`render()`) is a thin view over this array — each `.step-button` click just flips `seq[r][c]` and toggles a CSS class; there's no framework, it's direct DOM manipulation.

Playback (`start()` → `next()`) is a **self-scheduling `setTimeout` loop**, not a fixed `setInterval`:
```js
next = () => {
  samples.forEach((s, r) => seq[r][beat % 16] && play(s));
  const bd = 60 / bpm * 1000;                 // ms per 16th step at current BPM
  const sw = beat % 2 ? swing/100 * (bd/2) : 0;      // swing delays every other step
  const hu = (Math.random()-.5) * (humanize/100) * bd; // random ±jitter
  beat++;
  setTimeout(next, bd + sw + hu);
};
```
Each call fires the current step's active sounds, then computes the *next* delay from three components — base step duration, a swing offset applied only to odd-numbered steps (the classic "shuffle" feel), and a random humanize jitter — and reschedules itself. Because BPM/swing/humanize are read fresh each tick, changing a slider takes effect on the very next step with no restart needed. `beat` free-runs and wraps via `beat % 16` for track indexing and `beat % 4` for the on-screen quarter-note counter.

Note: this setTimeout-based clock is simple but not sample-accurate — under heavy main-thread load it can drift slightly, since it doesn't use Web Audio's own clock (`ctx.currentTime`) for scheduling.

### Patterns
`patterns` is a static lookup of `{ name, desc, p: number[8][16] }`. `loadPattern(key)` deep-copies (`p.map(r => [...r])`) the preset into `seq` and re-renders — copied, not referenced, so editing the loaded pattern never mutates the preset.

### Quantize
Rebuilds each track by, for every currently-active step, snapping its index to the nearest multiple of `16 / quantizeGridValue` (e.g. grid `4` → snap to nearest multiple of 4) and writing into a fresh 16-cell array — this both aligns hits and de-duplicates any that collapse onto the same snapped position.

### MIDI export
Uses `jsmidgen` to build a single-track Standard MIDI File: each active `seq[row][step]` becomes a `Note` at pitch `36 + row` (General MIDI drum-map-ish, kick=36 upward) at tick `step * 480/4` (480 ticks/quarter-note, so each 16th step is a quarter tick), fixed length 240 ticks and velocity 100. The result is serialized to bytes, wrapped in a `Blob`, and downloaded via a synthetic `<a download>` click — no server involved.

## File layout
Everything — markup, CSS, and JS — lives in the one file `grooveBox-app.html` for portability; there's no build step or package manifest.
