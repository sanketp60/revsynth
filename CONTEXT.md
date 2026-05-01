# RevSynth — Project Context

## Overview

A single-file (`index.html`) web application that simulates motorcycle engine sounds in real-time using the Web Audio API. Users can tweak engine parameters (cylinders, crank angles, RPM, stroke type) and hear how different configurations affect the firing order, exhaust note, and engine character. No build tools, no dependencies beyond a single CSS CDN link.

## Tech Stack

- **Single HTML file** — all CSS and JS inline, zero build step
- **Pico CSS v2.1.1** — minimal CSS framework via CDN, dark theme (`data-theme="dark"`)
- **Web Audio API** — all sound synthesis (no audio samples/files)
- **Vanilla JS** — no frameworks

## Architecture

### 1. EngineConfig (data model)

Holds all engine parameters and computes derived values:

- `rpm` — current engine RPM
- `cylinders` — number of cylinders (1-6)
- `crankAngles[]` — crank offset in degrees per cylinder
- `strokeType` — 2 or 4 stroke
- `redlineRpm` — where warning zone starts (amber slider thumb)
- `limiterRpm` — hard RPM cap with ignition cut (red slider thumb)
- `revSpeed` — throttle response multiplier (per-bike, 0.35-1.35)
- `cycleDegrees` — 720 for 4-stroke, 360 for 2-stroke
- `cycleDuration` — duration of one engine cycle in seconds
- `getFiringSchedule()` — returns sorted array of `{ cylinder, angleDeg, offsetSec }`
- `getFiringIntervals()` — returns array of time gaps (ms) between consecutive firings
- `isInRedline` — RPM is between redline and limiter (warning zone)
- `isAtLimiter` — RPM has hit the hard rev limiter
- `redlineProximity` — 0-1 normalized position within warning zone

### 2. Engine Presets

21 real motorcycle presets with accurate specs:

| Category | Bikes |
|----------|-------|
| Singles | Royal Enfield Classic 350, KTM 390 Duke |
| V-Twins | Ducati 959, Harley Sportster 883, Moto Guzzi V7, Indian Scout |
| Parallel Twins | Yamaha MT-07, Kawasaki Ninja 650, RE INT 650, Aprilia RS 660, Triumph Scrambler 1200 XE |
| Triples | Triumph Street Triple 765 RS, Yamaha MT-09 |
| Inline Fours | Honda CBR600RR, Kawasaki ZX-6R, Yamaha YZF-R1, BMW S1000RR |
| V4 | Ducati Panigale V4 |
| Flat/Boxer | Honda Gold Wing (Flat-6), BMW R1250GS |
| Custom | User-configurable |

Each preset includes: `angles`, `cylinders`, `stroke`, `idle`, `redline`, `limiter`, `revSpeed`.

### 3. EngineAudio (sound engine)

Wavetable synthesis approach based on research by Baldan et al. 2015 and game audio techniques.

#### Core concept
Instead of scheduling individual pulse one-shots, a single `AudioBuffer` (8192 samples) represents one complete engine cycle (720° for 4-stroke). Exhaust pulses are placed at correct crank angle positions within the buffer. The buffer loops, and `playbackRate` controls RPM.

**Why this works:**
- Pulses naturally overlap at high RPM (waveform compresses)
- No scheduling glitches or timing jitter
- Consistent timbre at all RPMs
- RPM changes are smooth (just a rate change)
- Config changes regenerate the wavetable with crossfade

#### Exhaust pulse shape
Each cylinder firing is a shaped waveform placed in the buffer:
- Sub-harmonic at 0.5x fundamental (chest-punch bass)
- Fundamental + 5 harmonics (slightly inharmonic for realism)
- Piston motion component (cosine)
- Subtle noise for turbulence
- Exponential decay envelope
- Fundamental frequency: 55Hz + cylinder offset (lower = more bass)
- 2V vs 4V valve config affects harmonic content (2V = gruntier, fewer upper harmonics)

#### Audio signal chain
```
Wavetable source → sourceGain → exhaustBus → waveshaper (asymmetric tanh) →
  ├─ dry (0.65) ──┐
  └─ pipe delay ───┤→ eqSub (lowshelf 80Hz +8dB) → eqLow (peaking 140Hz +7dB) →
                       eqMid (peaking 500Hz +2dB) → exhaustLP (RPM-tracking lowpass) →
                       highCut (highshelf 3kHz -8dB) → masterGain → compressor → output
```

#### Pipe resonance
Short feedback delay (4.5ms) creating a comb filter that simulates exhaust pipe standing waves. Delay time shortens with RPM (higher pitch pipe resonance at high RPM). User-adjustable via Pipe Length slider.

#### Additional sound layers

**Mechanical rumble** (continuous):
- Sine oscillator at 22Hz (fundamental) + triangle at 44Hz
- Frequency tracks half the firing frequency
- Louder at idle, fades as exhaust dominates
- Lowpass filtered at 180Hz

**Intake noise** (continuous):
- Looping white noise through bandpass filter (350Hz center)
- Volume and filter frequency scale with RPM

**Gear whine** (always on):
- Simulates straight-cut balance shaft gear noise (characteristic Triumph triple whine)
- 3 oscillators: sawtooth fundamental + sawtooth 1.5x + sine 2x
- Frequency = `(RPM / 60) * 16 teeth` — pitch tracks RPM linearly
- White noise layer for gear mesh grit
- Bandpass + peaking EQ (3.5kHz resonance) simulating engine casing
- Volume: quadratic RPM curve (barely audible at idle, prominent at high RPM)
- User-adjustable intensity via Whine slider

**Idle RPM jitter** (at idle):
- Organic ±20-40 RPM wavering using layered sine waves + random noise
- Active within 200 RPM of idle
- Updates both audio playback rate and RPM display

#### Rev limiter
When RPM hits the limiter position, rapid gain chopping simulates ECU ignition cut:
- ~60% of 20-35ms intervals get cut to 10% gain
- Playback rate stays constant (pitch doesn't wobble)
- Creates authentic sputtering/barking sound

### 4. UI — TFT Instrument Cluster

The UI is designed to look like a modern motorcycle TFT dashboard.

#### Cluster layout (top to bottom)
- **Preset selector** — dropdown at top, grouped by engine type
- **Shift lights** — 10 LEDs (green→amber→red) that light up as RPM approaches redline
- **Tach bar** — horizontal bar with gradient fill, redline/limiter markers
- **RPM display** — large monospace number (green/amber/red based on zone)
- **Gear indicator** — box showing current gear (N, 1-6)
- **Redline/Limiter/Idle sliders** — separate controls
- **Info strip** — RPM, fires/sec, cycle duration
- **Toggle switches** — Auto Decay (default ON), Auto Shift (default ON)
- **Keyboard hints** — shows arrow key controls

#### Full-screen redline flash
When RPM enters redline/limiter zone, the entire screen flashes red via a `::after` pseudo-element overlay with `pointer-events: none`.

#### Keyboard controls
| Key | Action |
|-----|--------|
| `→` (Right Arrow) | Throttle — hold to increase RPM |
| `↑` (Up Arrow) | Gear up (first press engages gear 1 from neutral) |
| `↓` (Down Arrow) | Gear down (from gear 1 drops to neutral) |

#### Throttle response (getRpmDelta)
RPM acceleration is non-linear and context-dependent, scaled by per-bike `revSpeed` and user `throttleSensitivity`:

**In Neutral** (`currentGear === 0`):
- Instant snap response (280 RPM/tick × revSpeed, 60ms ramp to full)
- Nearly linear — free-revving engine, no drivetrain load

**In Gear:**
- Slow ramp (0.8s to full throttle)
- S-curve: slow at bottom, fast in mid (powerband), slow at top
- Gear-dependent: gear 1 revs ~2.5x faster than gear 6
- All normalized to limiter RPM (not redline)

#### Auto decay (togglable)
When throttle is released (right arrow keyup), RPM decays to idle:

**In Neutral:** Fast decay ~5500 RPM/s × revSpeed
**In Gear:** Engine braking ~1400 RPM/s × (3.5/gear) × revSpeed

#### Gear system

**Manual shifting** (arrow keys): Always works regardless of auto shift toggle.
- ArrowUp from neutral → engages gear 1 (no RPM change)
- Subsequent ArrowUp → upshift with RPM animation
- ArrowDown from gear 1 → drops to neutral
- ArrowDown from gear 2+ → downshift with RPM blip up

**Auto gear shift** (toggle, default ON):
- Upshift triggers at **redline** (amber slider position), NOT limiter
- Only when `currentGear >= 1` (never auto-shifts from neutral)
- **Delayed**: 350-550ms random delay before shift fires (rider reaction simulation)
- Only fires when **throttle is actively held** (`isThrottleActive` flag)
- Cancelled if throttle released before delay expires
- Downshift: instant at 18% of redline RPM

**Gear shift animation** (performGearShift):
3-phase realistic animation (~390ms total):
1. **Phase 1** — Clutch pull (60ms): RPM hangs, drifts 5% toward target
2. **Phase 2** — RPM sweep (250ms upshift / 180ms downshift): S-curve ease-in-out with slight overshoot
3. **Phase 3** — Settle (80ms): Overshoot returns to exact target

Gear ratios (fraction of current RPM on upshift):
| Shift | Ratio |
|-------|-------|
| 1→2 | 0.65 |
| 2→3 | 0.72 |
| 3→4 | 0.76 |
| 4→5 | 0.80 |
| 5→6 | 0.83 |

Downshift: RPM = `min(limiter * 0.95, currentRPM / ratio)`

#### Virtual Joypad (mobile)
On touch-capable devices, a fixed overlay appears at the bottom of the screen:
- **Gear Up / Gear Down** buttons (left side, stacked vertically)
- **Throttle** button (right side, large — hold to rev)
- Uses `touchstart`/`touchend` events (not click) for proper hold-to-rev behavior
- `touchcancel` handled for reliability
- Semi-transparent blurred background with `env(safe-area-inset-bottom)` for notched phones
- Keyboard hints hidden on mobile (replaced by joypad)
- Detected via `'ontouchstart' in window || navigator.maxTouchPoints > 0`

### 5. Sound Tuning Panel

User-adjustable parameters that modify the audio engine in real-time:

- **Exhaust Type** (select): Stock Muffler / Slip-On / Straight Pipe — changes filter cutoff, waveshaper drive, master volume
- **Valves per Cylinder** (2V / 4V toggle): 2V = fewer harmonics (gruntier), 4V = richer high-end. Regenerates wavetable.
- **Bass** (slider 0-16): Controls sub-bass shelf EQ gain
- **Whine** (slider 0-100): Controls gear whine volume
- **Pipe Length** (slider 1-10): Changes feedback delay time (short = high resonance, long = deep)
- **Throttle Sensitivity** (slider 10-200): User override multiplier on top of preset revSpeed

### 6. Piston & Crank Animation

Canvas-based real-time animation showing all cylinders in horizontal layout:
- Each cylinder gets its own row, stacked vertically
- **Slider-crank mechanism**: crankshaft circle (left), crank arm, connecting rod, piston, cylinder bore walls spanning full width
- Piston stroke calculated to fill the bore: TDC (piston face) reaches the cylinder head wall
- Crank rotation speed tracks RPM in real-time (`rps = RPM / 60`)
- Each cylinder's crank pin is offset by its configured crank angle
- **Combustion flash**: orange fill appears inside the cylinder near TDC
- Piston color changes to orange on firing
- Cylinder labels (C1, C2, etc.) color-coded to match `CYL_COLORS`
- Canvas height scales with cylinder count (60px per row)
- Runs only when engine is started, stops when engine stops
- HiDPI aware (uses `devicePixelRatio`)

### 7. Firing Pattern Visualization

- Horizontal bar representing one engine cycle (0°-720° or 0°-360°)
- Color-coded dots at each cylinder's firing position
- Degree labels update for 2-stroke vs 4-stroke
- Recalculates on resize

### 8. Readout Panel

4-cell grid showing:
- Cycle duration (ms)
- Fires per second
- Firing intervals (ms between each consecutive firing)
- Firing order (cylinder sequence)

## File Structure

```
revsynth/
├── index.html     — entire application (HTML + CSS + JS, ~1900 lines)
├── wrangler.json  — Cloudflare Workers config for static asset serving
├── CONTEXT.md     — this file (project context for AI agents)
├── README.md      — project documentation
├── LICENSE        — MIT license
└── .gitignore     — git ignore rules
```

## Key Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `BUFFER_LEN` | 8192 | Wavetable buffer length in samples |
| `IDLE_RPM` | per preset | Engine idle RPM (dynamic, 750-1500) |
| `MAX_GEAR` | 6 | Maximum gear |
| `DOWNSHIFT_RPM_FRACTION` | 0.18 | Auto-downshift below 18% of redline |
| `TEETH` | 16 | Gear teeth for whine frequency calculation |
| `CYL_COLORS` | 6 colors | Cylinder color palette for visualization |

## Design Decisions

1. **Wavetable over pulse scheduling**: Previous iterations tried scheduling individual audio pulses per cylinder firing — caused timing glitches, machine-gun repetitiveness, and digital artifacts. Wavetable approach (proven in game engines) eliminates all of these.

2. **RPM slider as indicator only**: Removing slider interactivity and using keyboard-only control creates a more immersive, game-like experience. The orange fill bar provides clear visual feedback.

3. **Separate redline vs limiter**: Real motorcycles have a redline (warning) and a rev limiter (hard cut) at different RPMs. The triple-slider design lets users set both independently.

4. **`isThrottleActive` flag**: Auto-upshifts during throttle-off decay caused a major glitch (sudden RPM drops). Gating auto-upshifts behind an explicit throttle-held flag fixed it permanently.

5. **`isNeutral = currentGear === 0` only**: Previously used `currentGear === 0 || !autoGearShift` which wrongly treated manual gear engagement as neutral when auto-shift was off.

6. **Per-bike `revSpeed`**: Different motorcycles have vastly different throttle response. A Royal Enfield 350 (20hp) should not rev as fast as a Panigale V4 (214hp). The `revSpeed` multiplier scales both acceleration and decay.

7. **Pico CSS**: Chosen for minimal integration effort — semantic HTML elements get styled automatically, native switch components replace custom toggle divs, single CDN link with zero build step.

8. **TFT cluster UI**: Designed to look like a modern motorcycle instrument cluster with shift lights, tach bar, and prominent gear indicator.

## Hosting

Deployed on Cloudflare Workers with static asset serving via `wrangler.json`. Connected to GitHub repo `sanketp60/revsynth` for automatic builds on push.

## References

- Baldan, S. et al. (2015). "Physically informed car engine sound synthesis for virtual and augmented environments." IEEE SIVE 2015.
- Antonio-R1/engine-sound-generator (GitHub) — waveguide-based approach using AudioWorklet
- cemrich/js-motor-sound-generation (GitHub) — wavetable/speed-based approach inspiration
