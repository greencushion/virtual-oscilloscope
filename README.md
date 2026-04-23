# Virtual School Oscilloscope

A fully interactive, browser-based dual-channel oscilloscope simulator with an integrated dual-channel signal generator, designed for use in secondary school physics lessons. The entire application is a single self-contained HTML file with no external dependencies, no build process, and no server-side code.

---

## Table of Contents

1. [Overview](#1-overview)
2. [File Structure](#2-file-structure)
3. [HTML Layout Structure](#3-html-layout-structure)
4. [CSS Architecture](#4-css-architecture)
5. [The Oscilloscope — Controls and Behaviour](#5-the-oscilloscope--controls-and-behaviour)
6. [The Signal Generator — Controls and Behaviour](#6-the-signal-generator--controls-and-behaviour)
7. [JavaScript Architecture](#7-javascript-architecture)
8. [SVG Knob System](#8-svg-knob-system)
9. [Canvas Rendering Engine](#9-canvas-rendering-engine)
10. [Web Audio API Integration](#10-web-audio-api-integration)
11. [Microphone Input](#11-microphone-input)
12. [Measurement Tools](#12-measurement-tools)
13. [Auto-Scale Algorithm](#13-auto-scale-algorithm)
14. [Screenshot Export](#14-screenshot-export)
15. [Responsive Layout](#15-responsive-layout)
16. [Application State Object](#16-application-state-object)
17. [Calibration Data Arrays](#17-calibration-data-arrays)
18. [Deployment](#18-deployment)
19. [Browser Compatibility](#19-browser-compatibility)
20. [Known Limitations and Extension Ideas](#20-known-limitations-and-extension-ideas)

---

## 1. Overview

The application simulates the front panel and functionality of a dual-channel analogue cathode-ray oscilloscope, styled after the classic Hameg HM203 form factor that is common in UK school laboratories. It is paired with a virtual signal generator modelled on the Unilab bench signal generator used in many secondary school physics departments.

### Key features

- **Dual-channel oscilloscope display** with a 10×8 division graticule, phosphor-glow or white-background display modes, and live auto-scaling waveform rendering
- **Signal generator with two independent channels** (A → CH1, B → CH2), each with independent frequency, waveform type, amplitude, and on/off control
- **Real microphone input** via the Web Audio API, displayable on CH1 alongside a generator signal on CH2
- **Audio output** — generator signals play through the device's speakers at audible frequencies
- **Measurement tools** — automatic peak-to-peak voltage, frequency, and period readouts; draggable vertical and horizontal cursor lines with delta readouts
- **Auto-scale** — one-button optimisation of Volts/Div, Time/Div, and trace position for the live signals
- **Freeze** — captures both traces simultaneously for side-by-side comparison
- **Screenshot** — downloads a PNG image of the oscilloscope canvas
- **Fully responsive** — reflows for mobile devices, with the screen at the top and controls below
- **No dependencies** — zero external libraries, frameworks, or network requests at runtime

---

## 2. File Structure

The application is deliberately a single file:

```
oscilloscope.html   — the complete application (HTML + CSS + JavaScript)
README.md           — this documentation file
```

Everything — markup, styles, SVG graphics, and all JavaScript logic — is contained within `oscilloscope.html`. This makes deployment trivial: rename to `index.html` and drop it into any static host.

---

## 3. HTML Layout Structure

The page body contains two major instrument panels inside a `.lab` wrapper div:

```
<body>
  <h1>Virtual School Oscilloscope</h1>
  <div class="lab">
    <div class="scope" id="scopeEl">          <!-- Oscilloscope instrument -->
      <div class="scope-bar">                  -- Title bar
      <div class="scope-main">                 -- Main body (screen + panels)
        <div class="screen-col">               -- Canvas area
          <canvas id="crt">                    -- The oscilloscope screen
        <div class="right-col">                -- Control panels (right of screen)
          <div class="ch-panel">               -- CH1 controls
          <div class="ch-panel">               -- CH2 controls
          <div class="ch-panel">               -- Timebase controls
      <div class="rdbar">                      -- Measurement readout bar
      <div class="curbar">                     -- Cursor readout bar
      <div class="ctrl-bar">                   -- Bottom controls (power, freeze, etc.)

    <div class="unilab">                       -- Signal generator instrument
      <div class="ulab-bar">                   -- Title bar
      <div class="ulab-body">                  -- Generator body (2 channels + audio)
        <div class="ch-sect">                  -- Channel A section
        <div class="ch-sect">                  -- Channel B section
        <div>                                  -- Audio + sockets section
  </div>
</body>
```

### The canvas element

The canvas `id="crt"` has a fixed internal resolution of **600 × 400 pixels**. This is the coordinate space used for all drawing calculations. The CSS makes the canvas `width: 100%` so it scales visually with its container, but the drawing logic always works in the 600×400 coordinate space. Pixel coordinates are computed relative to the fixed width `W=600` and height `H=400`.

### SVG knobs

Each rotary knob is an inline SVG element. All knobs share the same three-layer structure:
1. An outer grey rim circle (the bezel)
2. An inner dark circle (the knob body)
3. A red indicator line (the pointer) whose endpoint is computed and updated by JavaScript

Scale markings are drawn into a `<g id="...scale">` group element that sits before the knob circles in the SVG DOM, so the scale markings render beneath the knob body rather than on top of it. The SVG elements all have `overflow="visible"` so that scale labels, which are drawn outside the nominal viewBox, are not clipped.

---

## 4. CSS Architecture

All CSS is written inline in a single `<style>` block in the `<head>`. There are no external stylesheets.

### CSS class naming conventions

| Prefix/Pattern | Purpose |
|---|---|
| `.scope`, `.scope-bar`, `.scope-main` | Oscilloscope outer structure |
| `.screen-col`, `.right-col`, `.ch-panel` | Oscilloscope layout columns |
| `.sec-lbl`, `.kl`, `.kv`, `.kg` | Knob labels and value displays |
| `.ksv` | SVG knob elements (cursor: ns-resize) |
| `.pb` | Panel button (oscilloscope controls) |
| `.pb.on`, `.pb.gon`, `.pb.yon` | Active button states (orange, green, amber) |
| `.rdbar`, `.rv`, `.rv1`, `.rv2` | Readout bar and value colours |
| `.curbar`, `.cout` | Cursor bar and cursor readout pills |
| `.ctrl-bar` | Bottom control bar (power, freeze, etc.) |
| `.pwr`, `.frz`, `.db`, `.ascale` | Control bar button types |
| `.ab`, `.ab.aon`, `.ab.muted`, `.ab.mic-on` | Audio/misc button states |
| `.unilab`, `.ulab-bar`, `.ulab-body` | Signal generator outer structure |
| `.ch-sect`, `.ch-sect-hdr` | Generator channel sections |
| `.lcd`, `.lcd-w`, `.lcd-lbl` | LCD frequency display |
| `.dg`, `.dl` | Dial group and label |
| `.wbtn`, `.wled`, `.wbl` | Waveform selector buttons and LEDs |
| `.ch-tog` | Generator channel on/off toggle |
| `.audio-sect`, `.audio-lbl` | Audio output section |
| `.sock`, `.sock-in` | Output socket decorations |

### Button visual states

Buttons use a consistent three-state visual system:
- **Off/default**: grey background (`#aaa9a0`), dark text
- **On/active**: coloured background — orange for general controls, green for CH1-related, amber for CH2-related
- The pressed state uses `border-bottom: 1px` and `transform: translateY(1px)` to simulate a physical key press

### Responsive breakpoint

A single media query at `max-width: 700px` triggers the mobile layout. See [Section 15](#15-responsive-layout) for full details.

---

## 5. The Oscilloscope — Controls and Behaviour

### Channel I (CH1) panel

| Control | Element | Function |
|---|---|---|
| VOLTS/DIV knob | `id="kv1"` | Sets vertical scaling for CH1. Drag up to decrease volts/div (zoom in), drag down to increase. 10 steps: 5mV–5V |
| POS I knob | `id="kp1"` | Shifts the CH1 trace vertically. Centred = zero offset. Range ±2 screen heights |
| CH1 ON/OFF | `id="bt-ch1"` | Toggles CH1 trace visibility. Green when on |
| DC/AC button | `id="bt-ac1"` | Toggles AC coupling label (visual only in this simulation) |
| NORMAL/INVERT | `id="bt-inv1"` | Inverts the CH1 waveform by multiplying all sample values by −1 |

### Channel II (CH2) panel

Identical controls to CH1 but for the second channel. The CH2 trace is rendered in yellow (`#ffcc00` in phosphor mode, `#775500` in white mode). The CH1 trace is green (`#39ff14` phosphor, `#007700` white).

### Timebase panel

| Control | Element | Function |
|---|---|---|
| TIME/DIV knob | `id="kt"` | Sets horizontal time scale. 11 steps: 50µs–100ms |
| H-POS knob | `id="khp"` | Shifts both traces horizontally by introducing a time offset into sample generation |
| DUAL button | `id="bt-dual"` | Shows both channels simultaneously |
| CH1 button | `id="bt-ch1m"` | Shows only CH1 |
| CH2 button | `id="bt-ch2m"` | Shows only CH2 |
| ADD button | `id="bt-add"` | Adds CH1 and CH2 mathematically and displays the result as a single trace on CH1 |

### Control bar (bottom of oscilloscope)

| Control | Function |
|---|---|
| ON / OFF | Powers the oscilloscope on and off. When off, the canvas is filled solid black and the animation loop stops |
| FREEZE | Captures the current live sample buffers for both channels into `froz1` and `froz2`. While frozen, the canvas redraws the frozen samples on every frame but does not regenerate them. Click again to unfreeze |
| PHOSPHOR | Sets the green phosphor display theme: dark green background, bright green CH1 trace with a glow effect, yellow CH2 trace |
| WHITE BG | Sets the white background theme: off-white background, dark green and dark amber traces without glow |
| AUTO SCALE | Runs the auto-scale algorithm — see [Section 13](#13-auto-scale-algorithm) |
| MIC INPUT: OFF/ON | Toggles the microphone input — see [Section 11](#11-microphone-input) |
| SCREENSHOT | Downloads a PNG of the oscilloscope canvas — see [Section 14](#14-screenshot-export) |

---

## 6. The Signal Generator — Controls and Behaviour

The signal generator has two completely independent channels, A and B, which feed CH1 and CH2 of the oscilloscope respectively.

### Per-channel controls (identical for A and B)

| Control | Function |
|---|---|
| CH A/B ON/OFF toggle | Enables or disables the channel entirely. When off, `g.on = false` and `genSig()` returns 0 for that channel. Audio oscillator for that channel is stopped |
| LCD frequency display | Shows the current output frequency. Updates in real time as the tuning dial is turned. Displays in Hz below 1000, in kHz above (e.g. `2.50k`) |
| RANGE rotary switch | A click-to-advance 4-position switch cycling through ×1 (1–20 Hz), ×10 (10–200 Hz), ×100 (100 Hz–2 kHz), ×1K (1 kHz–20 kHz). Sets the frequency bounds for the tuning dial |
| TUNING dial | Large dial with 40 tick marks. Drag up to increase frequency within the selected range, drag down to decrease. The current frequency within the range is set by the formula `freq = min + (min..max range) * normalised_angle` |
| AMPLITUDE knob | Sets the peak amplitude of the output signal in volts, from 0 to 5V. Controls both the simulated waveform amplitude on screen and the gain of the Web Audio oscillator node |
| SINE / SQUARE / TRIANGLE buttons | Select the waveform type. Each has a red LED indicator and a miniature SVG waveform icon. Selecting a new waveform type restarts the audio oscillator for that channel to use the new OscillatorNode type |

### Waveform mathematics

Signal generation happens in the `genSig(t, ch)` function, where `t` is time in seconds and `ch` is 1 or 2:

- **Sine**: `A × sin(2πft)`
- **Square**: `A × sign(sin(2πft))` — gives +A or −A depending on which half-cycle
- **Triangle**: `A × (2/π) × arcsin(sin(2πft))` — uses the mathematical identity that a triangle wave is the arcsine of a sine wave

### Audio output section

| Control | Function |
|---|---|
| SPEAKER: ON/OFF | Enables or disables audio output entirely. When turned off, both oscillators are stopped. When turned back on, both are restarted |
| MUTE: OFF/ON | Silences the audio by stopping the oscillators without changing the `spkOn` state. Unmuting restarts the oscillators. Useful for silencing mid-demonstration without losing the speaker-on setting |

---

## 7. JavaScript Architecture

All JavaScript is written in a single `<script>` block at the bottom of the `<body>`. It uses plain ES5-compatible JavaScript (except for the `async/await` in `togMic`) with `var` declarations throughout for maximum browser compatibility.

### Execution order

When the page loads, JavaScript executes in this sequence:

1. **SVG helper functions defined** — `svgEl()`, `drawScale()`, `drawDecScale()`, `drawTuningTicks()`
2. **Scale rings drawn** — all 12 scale groups are populated with tick marks and labels
3. **Constants defined** — canvas reference, context, dimensions, lookup arrays
4. **State object `S` initialised** — all oscilloscope and generator settings
5. **Audio variables declared** — all null initially (audio is lazy-loaded)
6. **Audio event listeners attached** — `unlockAudio()` bound to `mousedown` and `touchstart`
7. **Knob event listeners initialised** — all 7 oscilloscope knobs and 6 generator knobs
8. **Initial knob angles set** — volts knobs to 100mV position, time knob to 1ms position
9. **Signal generator knobs initialised** — tune dials, amplitude knobs, decade pointers
10. **Cursor drag listeners attached**
11. **LCD displays updated** — initial frequency values shown
12. **Animation loop started** — `startAnim()` begins the `requestAnimationFrame` loop

### Global variables

| Variable | Type | Purpose |
|---|---|---|
| `cv` | HTMLCanvasElement | The CRT canvas element |
| `ctx2` | CanvasRenderingContext2D | 2D drawing context |
| `W`, `H` | Number | Canvas width (600) and height (400) in pixels |
| `DX`, `DY` | Number | Number of horizontal (10) and vertical (8) divisions |
| `cW`, `cH` | Number | Width and height of one grid division in pixels |
| `VS` | Array | Volts/div step labels (string), 10 entries |
| `VM` | Array | Volts/div step values (number, in volts), 10 entries |
| `TS` | Array | Time/div step labels (string), 11 entries |
| `TM` | Array | Time/div step values (number, in seconds), 11 entries |
| `DECS` | Array | Decade range objects with `l` (label), `min`, `max` properties |
| `DECANG` | Array | Pointer angles (degrees) for each decade position: −90, 0, 90, 180 |
| `S` | Object | Master application state — see [Section 16](#16-application-state-object) |
| `froz1`, `froz2` | Float32Array | Frozen sample buffers for CH1 and CH2 |
| `animId` | Number | `requestAnimationFrame` handle |
| `audioCtx` | AudioContext | Web Audio context (null until first user interaction) |
| `masterGain` | GainNode | Master gain node; all oscillators connect through this |
| `oscA`, `oscB` | OscillatorNode | Audio oscillators for generators A and B |
| `gainA`, `gainB` | GainNode | Per-channel gain nodes for independent amplitude control |
| `micStream` | MediaStream | Live microphone stream |
| `micAnalyser` | AnalyserNode | FFT analyser connected to microphone for time-domain data |
| `micBuf` | Float32Array | 1024-sample buffer for microphone data |
| `audioReady` | Boolean | True once AudioContext has been created |
| `ampAngA`, `ampAngB` | Number | Current angle (degrees) of each amplitude knob |

---

## 8. SVG Knob System

All rotary controls are SVG elements. The system works by calculating the endpoint of a red indicator line to simulate rotation.

### Coordinate system

Each knob SVG is positioned so the knob body is centred at `(cx, cy)`. The red pointer line always starts at the knob centre `(cx, cy)` and its endpoint is calculated trigonometrically:

```javascript
function rotDot(lid, ang, cx, cy, r) {
  var rad = (ang - 90) * Math.PI / 180;
  // subtract 90 degrees because SVG 0° points right, but we want 0° to point up
  el.setAttribute('x2', cx + r * Math.cos(rad));
  el.setAttribute('y2', cy + r * Math.sin(rad));
}
```

Where `ang` is the angle in degrees (−135 = bottom-left, 0 = top, +135 = bottom-right) and `r` is the length of the pointer arm.

### Drag interaction

Each knob's SVG element listens for `mousedown` and `touchstart`. On drag, the vertical mouse/touch delta is used:

```javascript
function initKnob(id, onDelta) {
  // drag start: record starting Y position
  // during drag: call onDelta(startY - currentY)
  //   positive delta = dragged upward = knob rotates clockwise = higher value
}
```

The delta is multiplied by a sensitivity factor (1.5 for oscilloscope knobs, 1.2 for tuning dials) before being added to the current angle. The angle is clamped to the range −135° to +135°, giving a 270° travel range like a real potentiometer.

### Mapping angle to value

For all stepped controls (Volts/Div, Time/Div), the normalised angle `(ang + 135) / 270` gives a value from 0 to 1, which is then scaled to an array index:

```javascript
index = Math.round(normalisedAngle * (array.length - 1))
```

For continuous controls (Position, H-Pos), the normalised value is linearly mapped:

```javascript
// Position knobs: centred at 0, range ±2
S.pos1 = (normalisedAngle - 0.5) * 4;
```

### Scale ring drawing

The `drawScale(gid, cx, cy, innerR, labels, col)` function populates a `<g>` element with:

- **Major tick marks** — one per label, from the outer edge of the knob body to 8px beyond
- **Minor tick marks** — three evenly spaced between each pair of major ticks
- **Labels** — text elements positioned 15px beyond the outer tick edge, centred on the tick angle

All angles are distributed evenly across the 270° travel range (−135° to +135°). The labels are positioned at `innerR + 15` pixels from the knob centre, at the same angle as their corresponding major tick. SVG text elements use `text-anchor="middle"` for automatic horizontal centring.

---

## 9. Canvas Rendering Engine

The oscilloscope display is rendered frame-by-frame using `requestAnimationFrame`.

### Animation loop

```javascript
function animate() {
  drawScope();
  animId = requestAnimationFrame(animate);
}
```

`drawScope()` is the master drawing function called every frame (~60 fps). When the oscilloscope is powered off, the loop continues but only fills the canvas black. When frozen, the loop continues but draws the frozen sample buffers rather than regenerating them.

### Drawing sequence per frame

Each call to `drawScope()` executes these steps in order:

1. **`drawGrid(c)`** — fills background, draws sub-division lines (lighter), division lines (darker), and the central horizontal and vertical axis lines (brightest)
2. **`drawTrace(s1, 1, c.t1)`** — draws CH1 waveform if active and visible
3. **`drawTrace(s2, 2, c.t2)`** — draws CH2 waveform if active and visible
4. **`drawCursors()`** — draws active cursor lines and updates cursor readouts
5. **`drawLabels(s1, s2)`** — draws the Volts/Div and Time/Div labels in screen corners, and the FROZEN indicator if applicable
6. **`updateReadouts(s1, s2)`** — updates the measurement readout bar below the screen

### Grid drawing

The graticule is drawn in three layers:

- **Sub-divisions** (every `cW/5` pixels): very faint lines, `lineWidth = 0.4`
- **Main divisions** (every `cW` pixels, i.e. 10 columns and 8 rows): mid-weight lines, `lineWidth = 0.9`
- **Central axes** (at `W/2` and `H/2`): heavier lines, `lineWidth = 1.4`, slightly brighter colour

### Trace drawing

For each channel, 512 samples are built by `buildSamples(ch, 512)`. These are plotted as a continuous polyline:

```javascript
// x coordinate: proportional to sample index across full canvas width
x = (i / (samples.length - 1)) * W;

// y coordinate: centred at H/2, scaled by volts/div, offset by position knob
y = H/2 - (voltage / voltsPerDiv) * cellHeight + positionOffset;
```

In phosphor mode, `ctx.shadowBlur = 7` and `ctx.shadowColor = traceColour` produces the characteristic CRT phosphor glow. In white mode, `shadowBlur = 0`.

### Sample buffer generation

`buildSamples(ch, N)` generates N evenly-spaced time samples across one full screen width:

```javascript
totalTime = timePerDiv * DX;   // e.g. 1ms × 10 = 10ms for the full screen
t = (i / N) * totalTime + horizontalOffset;
voltage = genSig(t, ch);
```

The `hpos` (H-POS knob value) introduces a time offset `horizontalOffset = S.hpos * timePerDiv`, which scrolls both traces left or right.

---

## 10. Web Audio API Integration

The Web Audio API is used to drive audible output from the signal generator. Because browsers require a user gesture before audio can play, the entire audio system is lazy-initialised.

### AudioContext creation policy

```javascript
function unlockAudio() {
  if (audioReady) return;
  initAudio();           // creates AudioContext and masterGain
  resumeCtx(startBoth); // resumes if suspended, then starts both oscillators
}
document.addEventListener('mousedown', unlockAudio);
document.addEventListener('touchstart', unlockAudio);
```

The first mouse click or touch anywhere on the page triggers `unlockAudio()`. After that, the `audioReady` flag prevents repeated initialisation.

### Audio graph structure

```
OscillatorNode (Ch A)  →  GainNode (gainA)  →┐
OscillatorNode (Ch B)  →  GainNode (gainB)  →┤→  masterGain  →  AudioContext.destination
                                               │                  (speakers)
```

Each channel has its own OscillatorNode and GainNode. The masterGain is fixed at 0.2 (20% volume); per-channel volume is controlled through the individual GainNodes with gain values calculated as `amplitude / 5 * 0.3` (scaled so the maximum 5V amplitude produces a gain of 0.3, preventing clipping at full volume).

### Oscillator lifecycle

Web Audio oscillators cannot be restarted once stopped. Every time a frequency change, waveform change, or channel restart is needed, the pattern is:

1. Stop the old oscillator (`oscA.stop()` wrapped in try/catch)
2. Set `oscA = null` and `gainA = null`
3. Create new OscillatorNode and GainNode via `makeOsc()`
4. Connect them and call `.start()`

Frequency changes while an oscillator is running use `oscillator.frequency.value = newFreq` directly (no restart needed). This produces smooth, glitch-free frequency changes as the tuning dial is dragged.

---

## 11. Microphone Input

Microphone access uses the `navigator.mediaDevices.getUserMedia` API, which is an `async` operation and requires HTTPS (available automatically on Vercel).

### Activation sequence

When MIC INPUT is toggled on:

1. Browser prompts the user for microphone permission
2. On approval, a `MediaStream` is obtained
3. A `MediaStreamSourceNode` is created from the stream
4. An `AnalyserNode` (fftSize = 1024) is connected to the source
5. Channel A oscillator is stopped to prevent feedback
6. `S.micOn = true` — subsequent `genSig(t, 1)` calls read from the analyser buffer instead of computing a synthesised signal

### Sample reading

On every call to `genSig(t, 1)` while mic is active:

```javascript
micAnalyser.getFloatTimeDomainData(micBuf);
// micBuf is a 1024-element Float32Array of audio samples, values −1.0 to +1.0
idx = Math.floor((t / totalTime) * micBuf.length) % micBuf.length;
return micBuf[idx] * 5;   // scale to ±5V range
```

This maps the microphone buffer onto the oscilloscope's time axis. The microphone's amplitude is normalised to ±5V, which may not correspond to real voltage values but is appropriate for displaying the waveform shape.

CH2 continues to show the generator signal while mic is active on CH1, enabling comparison of microphone input against a reference signal.

---

## 12. Measurement Tools

### Automatic readouts

The readout bar below the screen displays six automatically computed values, updated on every frame:

| Display | Source | Calculation |
|---|---|---|
| CH1 Vpp | `vppOf(s1)` | `max(samples) − min(samples)` |
| CH2 Vpp | `vppOf(s2)` | `max(samples) − min(samples)` |
| Freq | `detectFreq()` | Zero-crossing method (see below) |
| Period | `1 / freq` | Reciprocal of detected frequency |
| ΔV | Cursor H readout | Difference between H-cursor V1 and V2 |
| Δt | Cursor V readout | Time difference between V-cursor t1 and t2 |

### Frequency detection algorithm

`detectFreq(samples)` counts zero-crossings in the sample array:

```javascript
// Count each time the signal changes sign (positive→negative or negative→positive)
// Number of complete cycles = crossings / 2
// Time span of the display = timePerDiv * 10
// Frequency = cycles / timeSpan
```

This works reliably for periodic signals. For noisy signals or signals with a DC offset, the detected frequency may be unreliable — Auto Scale should be used to improve accuracy.

### Cursor measurement system

Two types of cursor are available, activated independently:

**V-Cursors** (vertical dashed blue lines):
- Toggled by the V-CURSORS button
- Two vertical lines at positions `S.c1x` and `S.c2x` (pixel coordinates)
- Draggable by clicking within 14px (mouse) or 18px (touch) of the line
- Reports absolute time position of each cursor: `t = (cursorX / W) * timePerDiv * 10`
- Reports `Δt` between cursors and computes frequency as `1/Δt`

**H-Cursors** (horizontal dashed pink lines):
- Toggled by the H-CURSORS button
- Two horizontal lines at positions `S.c1y` and `S.c2y` (pixel coordinates)
- Voltage at each cursor: `V = (H/2 - cursorY) / cellHeight * voltsPerDiv`
- Reports `ΔV` between the two cursors

Cursor positions are stored in pixel coordinates and converted to physical units (time, voltage) at the moment of display, so they remain physically accurate when Volts/Div or Time/Div is changed.

---

## 13. Auto-Scale Algorithm

The `autoScale()` function selects the best display settings for the current signals. It runs in three phases:

### Phase 1 — Detect frequency and set Time/Div

```javascript
// Samples are built at the current (possibly wrong) timebase
// detectFreq() counts zero-crossings to estimate frequency
// Target: show approximately 3 complete cycles
targetTime = period * 3;
timePerDiv = targetTime / 10;  // 10 divisions across the screen
```

The algorithm then finds the smallest Time/Div step in the `TM` array that is greater than or equal to the ideal value. After setting the new timebase, samples are regenerated to get accurate readings for the next phase.

### Phase 2 — Set Volts/Div per channel

```javascript
// Find peak amplitude
peakV = max(|max(samples)|, |min(samples)|);
// Target: peak fills approximately 3 divisions
idealVPD = peak / 3;
// Find smallest Volts/Div step >= ideal
```

Each channel is scaled independently, so CH1 and CH2 can have different vertical sensitivities.

### Phase 3 — Centre traces

Position knobs for CH1, CH2, and H-Pos are all reset to zero, centring both traces vertically and horizontally. The knob pointer angles are updated to match.

After all three phases, the display labels (Volts/Div and Time/Div text) and knob angles are all updated to reflect the new settings, so the physical instrument panel remains consistent with the display.

---

## 14. Screenshot Export

```javascript
function snapShot() {
  var link = document.createElement('a');
  link.download = 'scope_' + Date.now() + '.png';
  link.href = cv.toDataURL('image/png');
  link.click();
}
```

`canvas.toDataURL('image/png')` returns the current canvas contents as a base64-encoded PNG data URL. A temporary anchor element is created with the `download` attribute and programmatically clicked, triggering a browser download. The filename includes a Unix timestamp to prevent collisions.

The screenshot captures only the canvas (the oscilloscope screen), not the surrounding control panels or the signal generator.

---

## 15. Responsive Layout

### Desktop layout (viewport wider than 700px)

```
┌─────────────────────────────────────────────────┐
│ OSCILLOSCOPE title bar                           │
├───────────────────────┬─────┬─────┬─────────────┤
│                       │ CH1 │ CH2 │  TIMEBASE   │
│   CRT Canvas          │     │     │             │
│   (flex: 1)           │     │     │             │
│                       │     │     │             │
├───────────────────────┴─────┴─────┴─────────────┤
│ Readout bar                                      │
│ Cursor bar                                       │
│ Control bar (power, freeze, display, autoscale)  │
└─────────────────────────────────────────────────┘
```

The `.scope-main` flex container is `flex-direction: row`. The canvas column has `flex: 1` so it takes all remaining width. The `.right-col` has `flex-shrink: 0` so it maintains its natural width.

### Mobile layout (viewport 700px or narrower)

```
┌────────────────────────────┐
│ OSCILLOSCOPE title bar      │
├────────────────────────────┤
│   CRT Canvas               │
│   (full width)             │
├─────────┬──────┬───────────┤
│  CH1    │ CH2  │ TIMEBASE  │
├─────────┴──────┴───────────┤
│ Readout bar                │
│ Cursor bar                 │
│ Control bar                │
└────────────────────────────┘
```

The media query `@media(max-width: 700px)` changes `.scope-main` to `flex-direction: column`, so the screen appears above the panels. The `.right-col` becomes `flex-direction: row; flex-wrap: wrap`, so the three channel panels sit side by side (wrapping if the viewport is very narrow). Border and spacing adjustments prevent double borders between panels.

The signal generator below the oscilloscope naturally reflows because its `.ulab-body` uses `flex-wrap: wrap`, so the channel sections stack vertically on narrow screens.

The canvas itself uses `width: 100%; height: auto` in CSS, which scales it proportionally while maintaining the 600:400 aspect ratio. All JavaScript drawing calculations continue to use the fixed 600×400 coordinate space — the scaling is purely visual via CSS.

---

## 16. Application State Object

The entire application state is stored in a single object `S`. This is the single source of truth for all settings:

```javascript
var S = {
  // Oscilloscope display state
  power:    true,        // Is the oscilloscope on?
  frozen:   false,       // Is the display frozen?
  disp:     'phos',      // Display mode: 'phos' or 'white'

  // Channel visibility and coupling
  ch1On:    true,        // CH1 trace visible
  ch2On:    true,        // CH2 trace visible
  ch1AC:    false,       // CH1 AC coupling (visual label only)
  ch2AC:    false,       // CH2 AC coupling (visual label only)
  ch1Inv:   false,       // CH1 signal inverted
  ch2Inv:   false,       // CH2 signal inverted

  // Vertical scaling (indices into VS/VM arrays)
  vi1:      4,           // CH1 Volts/Div index (4 = 100mV)
  vi2:      4,           // CH2 Volts/Div index (4 = 100mV)

  // Horizontal scaling
  ti:       4,           // Time/Div index (4 = 1ms)

  // Trace position offsets (in divisions)
  pos1:     0,           // CH1 vertical position
  pos2:     0,           // CH2 vertical position
  hpos:     0,           // Horizontal position (both channels)

  // Display mode
  viewMode: 'DUAL',      // 'DUAL', 'CH1', 'CH2', or 'ADD'

  // Cursor state
  cursV:    false,       // V-cursors active
  cursH:    false,       // H-cursors active
  c1x:      210,         // V-cursor 1 x position (pixels)
  c2x:      390,         // V-cursor 2 x position (pixels)
  c1y:      140,         // H-cursor 1 y position (pixels)
  c2y:      260,         // H-cursor 2 y position (pixels)

  // Generator channel A (→ CH1)
  a: {
    freq:       200,     // Current frequency (Hz)
    wave:       'sine',  // Waveform: 'sine', 'square', 'triangle'
    decIdx:     1,       // Decade range index (1 = ×10)
    tuneAngle: -100,     // Current tuning dial angle (degrees)
    amp:        2,       // Amplitude (volts peak)
    on:         true     // Channel enabled
  },

  // Generator channel B (→ CH2)
  b: {
    freq:       440,
    wave:       'sine',
    decIdx:     1,
    tuneAngle: -55,
    amp:        2,
    on:         true
  },

  // Audio state
  spkOn:    true,        // Speaker output enabled
  muted:    false,       // Muted
  micOn:    false,       // Microphone input active on CH1

  // Knob angles (degrees, range -135 to +135)
  // Used by initScopeKnob() closures to track current angle
  ka: {
    v1: 0, v2: 0,       // Volts/Div knobs CH1, CH2
    t:  0,              // Time/Div knob
    p1: 0, p2: 0,       // Position knobs CH1, CH2
    hp: 0               // H-Pos knob
  }
};
```

---

## 17. Calibration Data Arrays

### Volts/Div settings

```javascript
var VS = ['5mV','10mV','20mV','50mV','100mV','200mV','500mV','1V','2V','5V'];
var VM = [0.005, 0.01, 0.02, 0.05, 0.1, 0.2, 0.5, 1, 2, 5];
```

10 steps covering 5mV to 5V per division. The default index is 4 (100mV/div). These match the standard Volts/Div steps on a real analogue oscilloscope.

### Time/Div settings

```javascript
var TS = ['50us','100us','200us','500us','1ms','2ms','5ms','10ms','20ms','50ms','100ms'];
var TM = [50e-6, 100e-6, 200e-6, 500e-6, 1e-3, 2e-3, 5e-3, 10e-3, 20e-3, 50e-3, 100e-3];
```

11 steps covering 50µs to 100ms per division. The default index is 4 (1ms/div). At 1ms/div with 10 divisions, the screen shows 10ms of signal — enough to display one complete cycle of a 100Hz signal.

### Decade range steps

```javascript
var DECS = [
  { l: 'x1',   min: 1,    max: 20    },  // 1–20 Hz
  { l: 'x10',  min: 10,   max: 200   },  // 10–200 Hz
  { l: 'x100', min: 100,  max: 2000  },  // 100 Hz–2 kHz
  { l: 'x1K',  min: 1000, max: 20000 }   // 1 kHz–20 kHz
];
```

### Unit formatting functions

```javascript
fV(v)   // formats voltage: 0.015 → '15.0mV', 2.5 → '2.50V'
fT(t)   // formats time: 0.001 → '1.00ms', 0.000050 → '50.0us'
fF(f)   // formats frequency: 440 → '440.0Hz', 1500 → '1.50kHz'
```

---

## 18. Deployment

### Static hosting (Vercel, Netlify, GitHub Pages)

1. Rename `oscilloscope.html` to `index.html`
2. Create a new repository on GitHub and upload the file
3. Connect the repository to Vercel (or Netlify)
4. Deploy — no build configuration required

### Canvas as an iframe in an LMS (e.g. Canvas LMS)

The application can be embedded in Canvas LMS or any web page using an iframe:

```html
<iframe src="https://your-deployment-url.vercel.app"
        width="100%" height="900px"
        frameborder="0"
        allow="microphone">
</iframe>
```

The `allow="microphone"` attribute is required for the microphone input feature to work inside an iframe. Without it, the browser will deny the getUserMedia request.

### HTTPS requirement

The microphone input feature requires the page to be served over HTTPS. All of the following satisfy this:
- Vercel deployments (automatic HTTPS)
- Netlify deployments (automatic HTTPS)
- GitHub Pages (automatic HTTPS)
- Local development via `localhost` (browsers treat localhost as a secure context)

Opening the HTML file directly via `file://` will not allow microphone access. All other features (display, signal generator, audio output, screenshot) work from a local file.

---

## 19. Browser Compatibility

| Feature | Chrome | Firefox | Safari | Edge |
|---|---|---|---|---|
| Canvas rendering | Yes | Yes | Yes | Yes |
| Web Audio API | Yes | Yes | Yes | Yes |
| Microphone (getUserMedia) | Yes | Yes | Yes (HTTPS only) | Yes |
| SVG overflow visible | Yes | Yes | Yes | Yes |
| Screenshot (toDataURL) | Yes | Yes | Yes | Yes |
| requestAnimationFrame | Yes | Yes | Yes | Yes |

Chrome is recommended as the primary supported browser, particularly for microphone access and Web Audio reliability. The application has been tested and designed to work in all modern browsers released after 2018.

---

## 20. Known Limitations and Extension Ideas

### Current limitations

- **AC coupling is visual only** — the AC/DC button changes its label but does not actually filter the DC component from the signal. A real implementation would apply a high-pass filter using the Web Audio API's `BiquadFilterNode`.
- **Trigger is not simulated** — the trigger section was removed for simplicity. Without triggering, periodic signals are always stable because the simulation generates them mathematically from a fixed time origin. Live microphone signals may scroll.
- **Microphone calibration** — microphone input is scaled to ±5V, but this is arbitrary and does not correspond to real voltage values. The waveform shape is accurate, but amplitude measurements should not be taken literally.
- **No persistence-of-phosphor effect** — a real CRT shows after-glow where the beam has recently been. This could be approximated by drawing with reduced opacity over a slow-fading background.
- **Signal generator frequency is exact** — a real signal generator has slight inaccuracies. This does not affect pedagogical use.

### Possible extensions

- **Lissajous figures** — an X-Y mode that plots CH2 amplitude against CH1 amplitude, useful for demonstrating phase relationships
- **FFT spectrum view** — using the Web Audio AnalyserNode's frequency-domain data to display a spectrum
- **Preset signals** — a dropdown of common educational signals (50Hz mains hum, tuning fork A4 at 440Hz, square wave demonstrating harmonics)
- **Graticule overlay** — optional numbered division markings along the axes
- **Phase measurement** — automatic calculation of phase difference between CH1 and CH2 for signals of the same frequency
- **Multiple language support** — all label strings are in the HTML and could be internationalised
- **Save/load state** — using `localStorage` to persist settings between sessions (not possible in some iframe deployments)
