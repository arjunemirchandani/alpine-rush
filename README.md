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

**Drift is the boost economy.** Tap `SPACE` into a corner and steer through the slide — the meter
fills fast and a sustained slide builds a score multiplier up to ×6. Cruising above ~145 km/h
only trickles it. You start 4th of 4 on purpose.

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
