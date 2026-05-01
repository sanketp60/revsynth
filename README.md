# Motorcycle Engine Sound Simulator

A real-time motorcycle engine sound simulator built entirely in the browser using the Web Audio API. Tweak engine parameters — cylinders, crank angles, RPM, stroke type — and hear how different configurations affect the firing order, exhaust note, and engine character.

**No build tools. No dependencies. Just open `index.html`.**

## Demo

> **[Live Demo](#)** *(add your hosted URL here)*

## Features

- **Real-time sound synthesis** — wavetable-based exhaust simulation with pipe resonance, waveshaper saturation, and multi-band EQ
- **10 engine presets** — Single, V-Twin 90° (Ducati), V-Twin 60° (Moto Guzzi), Parallel Twin 270° (MT-07), Inline Triple (Triumph), Inline-4, Crossplane I4 (R1), V4 (Panigale), Flat-6 (Gold Wing)
- **Gear whine** — straight-cut balance shaft gear simulation (always on)
- **Mechanical rumble & intake noise** — continuous layers that track RPM
- **Triple-range RPM slider** — separate redline (warning) and rev limiter (hard cap) positions
- **Rev limiter** — realistic ignition-cut gain chopping at the limiter position
- **6-speed gear system** — manual shifting via keyboard/touch, or automatic with delayed upshift
- **Realistic throttle response** — non-linear, gear-dependent S-curve acceleration with separate neutral/in-gear behavior
- **Engine braking** — gear-dependent RPM decay on throttle release (lower gear = faster decay)
- **Piston & crank animation** — real-time canvas visualization of all cylinders with combustion flash
- **Firing pattern visualization** — shows cylinder firing positions within the engine cycle
- **Mobile virtual joypad** — touch controls for throttle and gear shifting on mobile devices
- **Dark theme** — Pico CSS with custom accent colors

## Controls

### Keyboard

| Key | Action |
|-----|--------|
| `→` (Right Arrow) | Hold for throttle |
| `↑` (Up Arrow) | Gear up (first press engages gear 1) |
| `↓` (Down Arrow) | Gear down (from gear 1 drops to neutral) |

### Mobile

On touch devices, a virtual joypad appears at the bottom:
- **Throttle** (right, large) — hold to rev
- **↑ / ↓** (left) — gear up / gear down

## How It Works

### Sound Engine

The simulator uses **wavetable synthesis** — a proven technique from game audio:

1. A single `AudioBuffer` represents one complete engine cycle (720° for 4-stroke)
2. Exhaust pulses are placed at the correct crank angle positions
3. The buffer loops, and `playbackRate` controls RPM
4. Pulses naturally overlap at high RPM as the waveform compresses

The raw waveform passes through:
- **Waveshaper** — asymmetric tanh soft-clipping for exhaust saturation
- **Pipe resonance** — short feedback delay (comb filter) simulating standing waves
- **3-band EQ** — sub-bass shelf, low-mid peak, mid presence
- **RPM-tracking lowpass** — brighter at high RPM, darker at idle
- **Compressor** — prevents clipping

Additional continuous layers: mechanical rumble (sine + triangle oscillators), intake noise (filtered white noise), and gear whine (sawtooth oscillators tracking gear mesh frequency).

### Engine Physics

- **Firing schedule** computed from crank angle offsets and stroke type
- **Throttle response** uses an S-curve with gear-dependent acceleration
- **Engine braking** rate scales inversely with gear number (`3.5 / gear`)
- **Gear ratios** follow real sportbike progression (1→2: 0.65, progressively tighter to 5→6: 0.83)
- **Auto-upshift** triggers at redline with 350-550ms random delay (rider reaction simulation), only while throttle is held

## Tech Stack

- **Single HTML file** — all CSS and JS inline, zero build step
- **[Pico CSS](https://picocss.com/) v2.1.1** — minimal CSS framework via CDN
- **Web Audio API** — all sound synthesis
- **Canvas API** — piston animation
- **Vanilla JS** — no frameworks

## Running Locally

```bash
# Just open the file
open index.html

# Or serve it (for mobile testing on the same network)
npx serve .
# or
python3 -m http.server 8000
```

## Hosting

This is a static single-file site. Deploy anywhere:

- **GitHub Pages** — push and enable Pages in repo settings
- **Netlify** — drag and drop the folder
- **Vercel** — `vercel --prod`
- **Cloudflare Pages** — connect the repo

## Project Structure

```
bike_sound_project/
├── index.html    — entire application (~1500 lines)
├── CONTEXT.md    — detailed technical context for AI agents
├── README.md     — this file
├── LICENSE       — MIT license
└── .gitignore    — git ignore rules
```

## References

- Baldan, S. et al. (2015). *"Physically informed car engine sound synthesis for virtual and augmented environments."* IEEE SIVE 2015.
- [Antonio-R1/engine-sound-generator](https://github.com/Antonio-R1/engine-sound-generator) — waveguide-based approach
- [cemrich/js-motor-sound-generation](https://github.com/cemrich/js-motor-sound-generation) — wavetable approach inspiration

## License

[MIT](LICENSE)
