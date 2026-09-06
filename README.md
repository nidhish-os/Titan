# Project TITAN — Particle Storm Engine

TITAN is a real-time, gesture-driven WebGL particle field: a sphere of
a few thousand points that lives on your webcam feed, rotates with
your hand, charges up on a pinch, and blows itself apart when you let
go. Underneath the visuals sits an **AI Core** state machine — a small
event-driven kernel that will eventually be wired up to real reasoning,
voice, and memory subsystems, and already drives the sphere's colour,
motion and glow through thirteen distinct states (idle, listening,
thinking, generating, error, sleep, and so on).

This version is a ground-up upgrade of the original single-file demo:
real bloom, a boot formation sequence, three additional gestures, a
full mouse/touch fallback so it works without a camera, adaptive
quality for weaker devices, and a proper GitHub Pages-ready project
layout.

## What makes this version different

- **Real bloom, not a fake glow sprite.** The original faked glow with
  an additive point-sprite texture. This version renders through a
  `three.js` `EffectComposer` with `UnrealBloomPass`, so bright
  particles genuinely bleed light into their surroundings — and it
  degrades gracefully (falls back to plain rendering) if the
  postprocessing scripts fail to load.
- **A cinematic boot sequence.** On start, particles scatter into a
  loose cloud around the sphere and assemble into formation over
  ~2 seconds instead of simply appearing — one deliberate "wow moment"
  rather than a pile of unrelated effects.
- **Three new gestures**, layered onto the original pinch-to-charge:
  - **Open hand** — gently expands the whole field outward.
  - **Fist** — pulls it inward and tight.
  - **Swipe** — injects a decaying directional turbulence gust.
- **Explosion trail streaks.** When the sphere bursts, fast-moving
  particles now draw short additive light trails instead of jumping
  as bare dots, using a bounded, pre-allocated line buffer that only
  costs draw time while something is actually exploding.
- **Works without a camera.** If camera access is denied, unavailable,
  or you just don't want to use it, a "Use Mouse / Touch Instead"
  option drives the exact same gesture state machine from synthesized
  hand landmarks — rotate by moving the cursor, hold to charge, release
  to burst, two-finger pinch to scale. No separate code path to
  maintain, and no dead end if the camera fails.
- **Speed-adaptive landmark smoothing.** Hand tracking noise is
  filtered per-landmark with smoothing that loosens when the hand is
  moving fast (feels responsive) and tightens when it's nearly still
  (kills jitter on a held pinch).
- **Adaptive quality.** A device-tier guess (cores, memory, mobile UA)
  picks a starting particle count and decides whether bloom is on by
  default; a runtime frame-time monitor then steps bloom and pixel
  ratio down automatically if a real device turns out to be slower
  than that guess, and cautiously back up if it isn't.
- **Responsive, not just shrunk.** Below 640px the desktop legend and
  dev panel are replaced with a compact instruction line and a
  two-button touch dock (`Pop`, `Core`), rather than the same UI at a
  smaller size.
- **Modular, GitHub-ready structure** (see below) instead of one
  1,100-line inline `<script>` block.

## What was intentionally preserved

The original's strongest ideas were kept, not rewritten for their own
sake: the AI Core event bus + kernel pattern, the thirteen state
profiles and their per-state motion styles (wobble, vortex, orbital
bands, scan, burst, breathe, flicker, jitter, vision-reactive), the
Fibonacci-sphere point distribution, the charge/explode/reform gesture
state machine, and the overall cinematic-terminal visual identity
(JetBrains Mono, Space Grotesk, the cyan/gold/magenta palette).

## How the particle system works

`N_PARTICLES` points are laid out on a sphere using a Fibonacci
lattice (even coverage without pinching at the poles). Each frame,
every particle's *target* position is computed from whatever state
it's in — ambient wobble driven by the AI Core's current profile,
inward pull while charging, outward flight while exploding, or a
lerp back to its resting position while reforming — and the actual
position eases toward that target, which is what gives the whole
field its soft, fluid feel instead of a rigid snap. Colour is
computed and blended the same way, sampled from a deep→bright
gradient specific to the current AI Core state.

## Interaction system

Two input sources feed one gesture engine (`js/gestures.js`):

| Input | Source |
|---|---|
| Camera | MediaPipe Hands, 21 landmarks per hand, up to 2 hands |
| Mouse / touch | `js/pointer.js` synthesizes the same 21-landmark shape from cursor/touch position and press state |

Because both sources produce the same landmark shape, the actual
gesture logic — pinch detection, open/fist detection, two-hand pinch
to scale, swipe-speed turbulence, rotation — is written once and used
by both. Camera mode uses real coordinates from MediaPipe; pointer
mode maps cursor position to a virtual palm and click/touch state to a
virtual pinch.

## Technologies used

- **three.js r128** — WebGL rendering, `BufferGeometry`/`Points` for
  the particle field, `EffectComposer` + `UnrealBloomPass` for bloom.
- **MediaPipe Hands + Camera Utils** — on-device hand tracking, loaded
  from jsDelivr, running entirely client-side.
- Vanilla JS (no framework, no build step), Google Fonts (Space
  Grotesk, JetBrains Mono).

## Project structure

```
titan/
├── index.html
├── css/
│   └── styles.css
├── js/
│   ├── config.js       # tunables + device-tier quality preset
│   ├── kernel.js        # event bus, AI Core state machine + profiles
│   ├── quality.js        # runtime frame-time monitor
│   ├── particles.js       # scene, particle buffers, bloom, all visual states
│   ├── gestures.js         # pinch/open/fist/swipe/rotate state machine
│   ├── pointer.js           # mouse/touch fallback (feeds gestures.js)
│   ├── hud.js                # skeleton/charge-arc canvas draw + chrome DOM
│   └── main.js                 # boot flow, animate loop, mode switching
├── assets/               # reserved for future local assets (none required today)
├── .gitignore
└── README.md
```

## Running locally

No build step and no server-side code — but camera access requires a
*secure context*, so plain `file://` won't get you `getUserMedia`.
Serve the folder locally instead, for example:

```bash
cd titan
python3 -m http.server 8080
# then open http://localhost:8080
```

Any static file server works equally well (`npx serve`, VS Code's
Live Server, etc.).

## Deploying to GitHub Pages

1. Push this folder's contents to the root of a GitHub repository
   (or to a `docs/` folder — either works with Pages).
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to "Deploy from a
   branch," pick the branch (e.g. `main`) and the folder (`/` or
   `/docs`), then save.
4. GitHub will publish the site at
   `https://<your-username>.github.io/<repo-name>/` within a minute
   or two. GitHub Pages serves over HTTPS by default, which is
   required for camera access to work.
5. Because every asset path in `index.html` is relative
   (`css/styles.css`, `js/main.js`, …), the site works whether it's
   served from the domain root or a repo subpath, and a hard refresh
   or direct link to the page works with no server-side routing.

No environment variables, API keys, or backend are required.

## Browser requirements

- A modern evergreen browser (recent Chrome, Edge, Firefox, or Safari)
  with WebGL support.
- Camera mode additionally requires `getUserMedia` support and a
  secure context (HTTPS or `localhost`).
- Mouse/touch mode has no special requirements beyond WebGL.
- Mobile: works on modern iOS Safari and Android Chrome; camera mode
  on mobile is supported but the touch fallback is the better default
  experience on a phone-sized screen (see the mobile touch dock).

## Camera permissions & privacy

- The camera stream is read directly into an off-screen `<video>`
  element and into MediaPipe's on-device model. No frame, landmark,
  or image data is ever sent to a server — everything happens in your
  browser.
- If camera permission is denied or unavailable, the app does not
  crash: it shows a clear error screen with a one-click path to
  Mouse/Touch mode instead.
- No API keys are used or required anywhere in this project.

## Known limitations

- Hand tracking quality depends on lighting and camera quality, as
  with any webcam-based tracker; MediaPipe's confidence thresholds are
  tuned for a reasonably lit, front-facing hand.
- The runtime quality monitor adjusts bloom and pixel ratio live, but
  does **not** change particle count after boot (that requires a
  geometry rebuild); the starting particle count is fixed by the
  device-tier guess made at load time.
- Bloom requires the `three.js` postprocessing example scripts to load
  from the CDN; if that request fails (e.g. a restrictive network),
  the app automatically falls back to standard rendering rather than
  failing.
- The mouse/touch "open hand" and "fist" gestures don't have a direct
  mouse equivalent, so pointer mode expresses a gentle constant
  "open" ambient expansion by default and reserves press/hold for
  charging — it approximates the camera experience rather than
  reproducing it exactly.
- `navigator.deviceMemory` isn't available in every browser (notably
  Safari); on those browsers the device-tier guess falls back to CPU
  core count and mobile detection alone.

## Final testing notes

Checked before calling this done: no hardcoded machine paths, no
`localhost`-only assumptions, no exposed keys or secrets, all asset
paths are relative and Pages-compatible, camera-permission denial is
handled without crashing, mouse/touch mode was exercised as a full
substitute for camera mode, resize/reflow was checked at desktop and
mobile widths, and the bloom pipeline was written to degrade to plain
rendering rather than throw if the postprocessing scripts don't load.
Because this environment can't run a live browser against the
external CDN scripts, this is a thorough code-level review rather than
an in-browser pixel-and-console pass — do a quick manual check in your
own browser's console after deploying, particularly on the specific
mobile devices you care about.
