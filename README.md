# 🏔️ Alpine Rush

A complete 3D arcade racer in a **single HTML file**. Three laps, three AI rivals, one mountain.
No build step, no bundler, no assets — every texture, sound and mesh is generated at runtime.

```bash
open alpine-rush.html
```

Needs a network connection on first load for the Three.js module (from unpkg). Everything else is self-contained.

---

## Controls

| Input | Action |
|---|---|
| `W` / `↑` | Throttle |
| `S` / `↓` | Brake / reverse |
| `A` `D` / `←` `→` | Steer |
| `SPACE` | Handbrake — initiates a drift |
| `SHIFT` | Boost (spends the meter) |
| `C` | Cycle camera (chase / bonnet / cinematic) |
| `R` | Respawn on the racing line |
| `P` or `ESC` | Pause · `M` Mute |

Gamepad is supported: left stick steers, RT/LT are throttle and brake, A drifts, B boosts.

Typing in the seed field is safe — driving keys are ignored while an input has focus.

**Drift is the boost economy.** Tap `SPACE` into a corner and steer through the slide — the meter
fills fast and a sustained slide builds a score multiplier up to ×6. Cruising above ~145 km/h
only trickles it. You start 4th of 4 on purpose.

---

## Circuits

The default circuit is `ALPINE` — the hand-tuned original. Any other seed string generates a
brand-new one: layout, elevation, mountain range, forest, kerbs and the AI's racing line all
regenerate together.

- Type a seed into the **CIRCUIT** field, or hit **SHUFFLE** for a random one.
- The URL tracks it, so `?seed=JAKDA` is a shareable link to an exact circuit.
- The same seed always produces the same track, on any machine.

The title card reports what you're about to drive:

```
2.29 KM · 11 CORNERS · TIGHTEST 48M · 42% FLAT OUT · TECHNICAL · 3 DRAWS
```

### Why every seed is drivable

Two different guarantees, applied at two different levels:

**Structural.** The centreline is a radial function `r(θ)`, so self-intersection is *impossible* —
no seed can ever produce a track that crosses itself. That isn't checked, it simply cannot happen.

**Measured.** Everything else — minimum corner radius, gradient, lap length, corner count — is not
guaranteed by the representation, so it is measured after generation and enforced:

```js
const LIMITS = { minRadius: 46, minR: 165, maxGrade: 0.27, len: [1650, 3000] };
```

A candidate that fails any limit is discarded and the next is drawn from the same seeded stream.
Because the stream is deterministic, the same seed always walks the same sequence and accepts the
same track. Over 66 test seeds: **zero fallbacks**, 4.7 draws on average, 15 at worst against a cap
of 60. Generated circuits ranged 1.81–2.31 km with 6–11 corners and tightest radii of 46–104 m.

The AI drives them unassisted — on the most technical generated seed (11 corners, 52 m tightest,
only 26% flat out) all four cars lapped in 38.6–40.4s.

---

## How it works

### The track is an analytic curve, not a spline

The centreline is a radial function `r(θ)` sampled at 1500 points, rather than a hand-authored spline:

```js
r(θ) = 300 + w(θ) · (62·sin(3θ+0.6) + 28·sin(5θ+2.1) + 34·sin(2θ−0.9))
```

Because `r` is **single-valued in θ**, the curve provably cannot self-intersect. That one property
means lap counting can't double-fire, position ranking can't invert, off-track detection can't
confuse two nearby pieces of road, and the AI can't path onto the wrong lobe — all correct by
construction rather than by testing.

Everything else derives from that single function: terrain height, road and kerb ribbons, snow
banks, tree placement, the minimap and the AI's racing line. Nothing can drift out of sync with
anything else.

The `w(θ)` term fades the modulation out around θ=0, carving a constant-radius main straight
under the start gantry — somewhere to actually reach top speed.

The harmonics, amplitudes, phases and straight-width are drawn from a seeded PRNG, which is what
makes circuit generation nearly free: the representation was already a small bag of numbers.

### Rebuilding the world

Every track-derived mesh lives under a single `world` group. Re-seeding disposes it — geometries,
materials **and any texture the material owns**, since `material.dispose()` does not free textures
and that leaks one per rebuild — then rebuilds terrain, road, kerbs, banks, gantry and scenery.
Sky, lights, shared textures, cars and HUD are track-independent and survive untouched.
Verified at 12 consecutive rebuilds with zero growth in geometry or texture count.

### Drift physics

Velocity is decomposed into the car's local frame each step, forces are applied, then it is
recomposed against the **old** basis — so the body rotates while momentum keeps pointing the old
way. That's the drift.

Two deliberately *unphysical* details make it feel good rather than punishing:

- **Momentum recovery.** Scrubbed lateral velocity is partly fed back as forward drive, so a
  committed slide carries speed. Decaying it honestly to zero scrubs the car to a dead stop.
- **Self-catch.** The aligning torque ramps up past ~50° of slip, so the car rescues itself
  instead of spinning. You can throw it in harder than you'd otherwise dare.

### AI

All three rivals run the **identical physics function** as the player, just with synthetic inputs —
so they drift, smoke their tyres and burn boost exactly like you do. Each plans corner speed from
precomputed curvature ahead (`v = √(grip/κ)`), follows a band-passed curvature signal for an
out-in-out racing line, avoids other cars, and rubber-bands gently.

Measured over a full race: laps of 39–41s, off-track ~4%, drifting ~13–18% of the time, and a
finishing spread of **0.06 seconds** across two minutes of racing.

### Everything is generated

- **Audio** — Web Audio only. The engine is detuned sawtooth + square oscillators through a
  filter tracking simulated RPM across six gears; tyre screech, wind and off-track rumble are
  filtered noise; boost is a swept band-pass.
- **Terrain** — ridged fBm over a 280×280 grid, vertex-coloured for rock / snow / pine.
- **Textures** — asphalt, kerbs and the chequered line are drawn to `<canvas>` at load.
- **Effects** — pooled GPU particles for tyre smoke and snow spray; skid marks are a ring buffer
  of 2400 quads that fade via a birth-time attribute in the shader (zero per-frame CPU cost).

Roughly 45 draw calls and 590k triangles.

---

## Debug handle

`window.__rush` is exposed for tinkering from devtools:

```js
__rush.Game.autopilot = true    // hand your car to the AI
__rush.player.boostMeter = 1    // fill the boost bar
__rush.Game.respawn()           // pop back onto the racing line
```

Autopilot is off by default and takes effect on the next frame — no restart needed.

---

## A note on verification

The steering was inverted in the first playable build, and it survived *every* automated check —
because all of them were internally consistent. The AI derived its steering from world geometry
and silently self-corrected; the physics agreed with themselves; every number checked out. The
underlying cause was that `cross(up, forward)` gives the *left* normal in Three.js's right-handed
space, not the right.

A machine verifying its own coherence will never catch a convention that is wrong at the root.
It took one human at the keyboard about ten seconds. There's a comment at the definition in
`buildCentreline()` that ends *"Flip one, flip all."*
