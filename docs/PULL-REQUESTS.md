# Development history

Alpine Rush was built as a series of pull requests, each one carrying the reasoning and the
measurements behind a change. The repository was later recreated with a rewritten commit history
to remove a personal email address from the commit metadata, which deleted the original PR pages.
Their write-ups are preserved here.

The commits themselves survive unchanged — `git log` carries the same detail.

---

## #1 — Seeded circuit generation

Any seed string now produces a distinct circuit. `ALPINE` stays the default and is **bit-for-bit the original track** — verified at 2102.79 m against an independent recomputation of the old hard-coded formula.

```
CIRCUIT  [ JAKDA ]  [ SHUFFLE ]
2.29 KM · 11 CORNERS · TIGHTEST 48M · 42% FLAT OUT · TECHNICAL · 3 DRAWS
```

## Why this was cheap

The track was already an analytic radial curve, so its entire shape is a small bag of numbers — harmonics, amplitudes, phases, base radius, straight width. Drawing those from a seeded PRNG *is* the feature. Terrain, road, kerbs, banks, forest, minimap and the AI's racing line all already derived from that one function, so they regenerate for free.

## Two guarantees, two mechanisms

**Structural** — `r(θ)` is single-valued in θ, so self-intersection is *impossible* for any seed. This isn't validated; it cannot happen.

**Measured** — corner radius, gradient, lap length and corner count are *not* guaranteed by the representation, so they're measured after generation and enforced:

```js
const LIMITS = { minRadius: 46, minR: 165, maxGrade: 0.27, len: [1650, 3000] };
```

A failing candidate is redrawn from the same deterministic stream, so the same seed always accepts the same track.

## Verification

| Check | Result |
|---|---|
| Default circuit unchanged | 2102.79 m, exact match |
| 66 seeds generated | **0 fallbacks**, 4.7 draws avg, 15 worst (cap 60) |
| Variety | 1.81–2.31 km, 6–11 corners, 46–104 m tightest |
| Determinism | same seed reproduces after perturbing the RNG |
| AI on hardest seed | 11 corners / 52 m / 26% flat out — all 4 cars lapped 38.6–40.4s |
| 12 consecutive rebuilds | zero growth in geometry **and** texture count |

## One real bug found

`material.dispose()` does not free the material's textures, so the start-gantry banner leaked **exactly one texture per rebuild** (confirmed: +6 over 6 rebuilds). Banner texture hoisted to module scope, plus a `SHARED_TEX` guard in `clearWorld()` so future per-build textures are freed but module-owned ones are never touched.

## UI

- Circuit field + **SHUFFLE** on the title card, with live track stats
- `?seed=JAKDA` in the URL — shareable links to an exact circuit
- Driving keys suppressed while an input has focus
- Seed shown on the results card

## Try it

```
?seed=JAKDA   11 corners, 48 m tightest — technical
?seed=6P8RA   11 corners, 26% flat out — the AI test case
?seed=AP4HW   76 m tightest, 50% flat out — fast and flowing
```

---

## #2 — Difficulty fix, headless simulation, and projected finish times

Two fixes that share one enabling capability. Both were found by testing rather than by reading code.

---

## 1. Difficulty tiers weren't reliably ordered

ROOKIE and ACE had never been tested. On tight circuits a **higher difficulty could lap slower than a lower one**:

| Circuit | ROOKIE | PRO | ACE |
|---|---|---|---|
| ALPINE — 8 corners | 40.52 | 39.80 | 38.89 ✓ |
| **6P8RA — 11 corners** | **38.59** | **38.71** ✗ | 37.99 |

`skill` fed only `latGrip` — the *planned* corner speed. It raised ambition without improving execution, so faster-planning drivers overcooked tight corners and lost more running wide than the entry speed gained. Off-track tracked it exactly: ROOKIE 2.7% vs PRO 7.6%.

**Grounded the constant in measurement.** Sampled the lateral acceleration the car actually pulls (`v · yawRate`, 67k samples): p50 14.8, p90 26.7, **p95 28.9 m/s²**. ACE was planning at **28.0** — the p95, achievable ~5% of the time. Planned grip now tops out near 25.5; skill instead buys tighter line-holding plus a wide-running recovery term.

| | Before | After |
|---|---|---|
| Monotonic ordering | 3/4 | **22/22** |
| Avg ROOKIE→ACE spread | ~1.6 s/lap | **2.84 s/lap** |
| Worst off-track | 7.6% | **6.1%** (0% on most) |

Against a fixed-strength reference driver: ROOKIE **+1.92 s/lap**, PRO **+0.10**, ACE **−0.91**.

---

## 2. Rivals showed DNF when you finished ahead of them

They hadn't failed to finish — they were still on track. Rather than extrapolate from average pace, the remainder of the race is **simulated headlessly** and the real times reported.

```
1  KAIJU   2:01.833
2  FROST   +0.800
3  YOU     +1.617
4  VIPER   +4.283 ◦     ← projected, dimmed
◦ projected — simulated to the flag after you finished
```

Every car's physics state is snapshotted and restored — **verified byte-identical** across all four cars and the full `Game` state — so the scene behind the results card is undisturbed. Early-exits when the last car is home: typically 0–8 simulated seconds, single-digit milliseconds.

Projected entries are marked. They come from the real physics, but the player didn't watch them happen and the UI shouldn't pretend otherwise. Results now show gaps to the winner rather than four absolute times.

Verified across 5 seed/difficulty combinations: every car receives a time in every case, projected counts vary correctly from 0 (player finishes last) to 3 (player wins).

---

## The enabling capability

Measuring balance through the rendered loop took ~40 minutes per sweep and stalled whenever the browser throttled the tab. Extracted `stepSim()` from `frame()` so the physics advance has a single definition, and added `Game.fastForward(seconds, sample)` which drives it without rendering — **~850× real time**. A 22×3 sweep now runs in ~7 seconds.

It deliberately calls the same `stepSim()` the render loop uses, so measurements exercise the real physics rather than a reimplementation that could drift from it. The projected finish times are the same mechanism turned toward gameplay.

```js
__rush.Game.fastForward(105)   // 105 simulated seconds in ~120 ms
```

---

Also fixes a pre-existing reset bug found in passing: `placeOnGrid()` zeroed velocity but not the derived fields, so the speedometer could flash the previous race's speed for a frame on restart.

---

## #3 — Slipstream, and a correction to the difficulty figures

Tuck in behind another car and you punch into their hole in the air — extra drive under throttle, thinner drag, slightly higher terminal speed. Only above ~65 km/h and only under power, so it rewards committing to a tow rather than lifting. HUD meter, screen streaks, rising air noise, and vapour peeling off the car ahead.

Rivals use it too: they ease off avoidance while tucked in on a straight, then slingshot past with boost.

## My first guess was wrong

The aggressive setting felt like the obvious choice — and it made the racing **worse**:

| | Field spread | Gap 1st→2nd | Lead changes |
|---|---|---|---|
| off | 4.98 s | 2.20 s | 4.4 |
| accel 8.5, aiHold 0.22 | **6.40 s** ✗ | 3.68 s ✗ | 6.4 |

The tow let cars close up, then the AI sat in it instead of passing and arrived off-line at the next corner. A slipstream train — realistic, but bad racing.

A 10-seed sweep found exactly one setting that beats no-slipstream on field spread, gap to the leader **and** lead changes simultaneously:

| | Field spread | Gap 1st→2nd | Lead changes |
|---|---|---|---|
| off | 4.98 s | 2.20 s | 4.4 |
| **shipped** | **4.66 s** ✓ | **2.19 s** ✓ | **5.1** ✓ |
| stronger | 5.95 s | 2.14 s | 5.4 |

Shipped: `accel 5, vmax +2.5%, drag −40%, aiHold 0.6`. Exposed as `__rush.TOW` so it can be A/B'd live.

## A correction to the numbers I reported in #2

A regression check appeared to show slipstream breaking difficulty ordering — 22/22 down to 12/20. **It hadn't.** The same run with slipstream fully disabled gave 6/12. The metric itself is noisy: each rival draws a random wander phase per page load, worth ~0.1–0.2 s of lap time, enough to flip the order wherever two tiers sit close.

Re-measured with **three wander realisations averaged per tier** across 10 circuits:

| | Reported in #2 | Actually |
|---|---|---|
| Monotonic | 22/22 | **9/10** |
| Tier spread | 2.84 s/lap | **1.91 s/lap** |
| ROOKIE / PRO / ACE vs reference | +1.92 / +0.10 / −0.91 | **+1.16 / −0.14 / −0.74** |

The earlier figures were a single uncontrolled realisation and were optimistic. README corrected, with a note that tier comparisons must average over realisations to mean anything.

Slipstream itself is neutral-to-positive on ordering: **10/10 with it on, 9/10 with it off**, on identical circuits and phases.

## Verified

- Tow engages in real racing: up to 27% of the time per car, max strength 0.96
- HUD meter, panel glow, screen streaks, vapour and audio all confirmed live
- Snapshot/restore and grid reset carry the new field, so finish projection is unaffected

---

## #4 — Time of day: dawn, midday, golden hour and dusk

Each circuit is lit by its own hour, drawn from the seed — so `?seed=MOUNTAIN` always arrives at golden hour. `ALPINE` stays at midday. A selector on the title card overrides it.

## Lighting alone wasn't enough

My first few passes just tinted the sun orange, and the alpine valley turned into a desert — everything shifts warm together and the snow reads as sand. Raising the cool fill to compensate produced overcast blue instead.

What actually sells low sun is **shadow**: ridges throwing long shade, with those areas lit only by cool skylight. The real-time shadow map can't provide it — it covers a 70 m box around the player and only cars cast into it, so the terrain casts nothing.

So sun occlusion is **baked into the terrain** at build time. Each vertex marches a ray toward the sun across the height grid; anything blocked by a ridge is tinted toward a cool shadow colour. The step grows with distance, so it costs ~30 ms during a rebuild and nothing per frame. That one change is the difference between "orange filter" and golden hour.

## Two bugs found on the way

**The sun was never initialised.** I'd moved its placement into `updateCamera`, which only runs while the race is live — so on the menu, when paused, or in any headless frame, light and target both sat at the origin and lit from a degenerate direction. This was already live before this feature: the title-screen backdrop was mis-lit. It also meant my first round of colour tuning was done against a broken render, which is why it kept fighting me.

**The hour was skewed.** Drawn as the last value of each generated candidate — and since the generator discards whole candidates on retry, the surviving draw came out at **43% dusk, 10% midday** over 120 seeds. Moved to its own seeded stream with a short warm-up (mulberry32's first outputs track its seed):

| | DAWN | DAY | GOLDEN | DUSK |
|---|---|---|---|---|
| before | 13% | 10% | 34% | 43% |
| after | 26% | 24% | 21% | 29% |

400 seeds across two independent seed families, max 4.2% from uniform — within sampling noise.

## Verified

- Sky textures disposed on relight: **zero texture growth** over 8 hour changes
- Rebuild including the bake: ~150 ms, menu only
- Physics untouched — full race on a generated circuit, 38.3–39.2s laps, no console errors
- `ALPINE` still resolves to midday

---

## #5 — Make boost engagement reliable

> "sometimes it would boost, and sometimes not"

It was **neither the key handling nor slipstream** — I verified both are clean. Both Shift keys engage and release correctly through the whole chain, and the tow value was 0 throughout every boost test. The two systems don't share any state.

## What it actually was

Instrumenting the chain:

| | |
|---|---|
| Full meter drained | 1.00 → 0.53 in **1.2 s**, empty in 2.4 s |
| Press below ~0.15 | sub-0.2 s burst — imperceptible |
| Bar at meter 0.09 | still showed a **lit segment** |
| Refused press | **no feedback at all** |

So the meter emptied quickly, and its last fifth bought nothing you could feel while still reading as available charge. Press, nothing happens, no explanation.

## Fix

- **`BOOST_MIN` 0.12 to engage.** Below that it refuses and *keeps* the fuel rather than dribbling it away.
- **`BOOST_BURST` 0.5 s minimum** once engaged — a tap always produces something you can feel. Holding re-arms while fuel lasts.
- **Drain 0.42 → 0.32** — a full meter is now ~3.0 s instead of 2.4 s.
- **HUD tells you why**: reserve segments marked magenta, label switches between `HOLD SHIFT` / `CHARGING` / `NEED MORE BOOST — DRIFT TO CHARGE`, and a refused press flashes it.
- **Whoosh fires from the physics edge**, not a 60 ms `setInterval` poll that could miss short bursts entirely — which reinforced "it didn't boost".

## Measured after

| | Before | After |
|---|---|---|
| Hold from full | 2.4 s | **3.0 s** |
| Tap | imperceptible dribble | **0.47 s burst** |
| Press on 0.07 meter | spent fuel, no effect | **0 s, refuses, keeps fuel, tells you** |

Rival boost usage rose from 7% to 11–18% of race time (the minimum burst makes each activation count). Lap times unchanged at 38.5–40.4 s. Finish projection unaffected — the new state is carried in snapshot/restore.

Branched from `main`, so it's independent of #4 and can merge in either order.

---

## #6 — Capture the results card at the finish, not when it renders

`showResults()` runs on a 1.4 s timeout and read **live** car state. `Enter` restarts the race while `Game.state` is still `'done'` — i.e. during exactly that window — so restarting mid-delay blanked the best lap and total, and showed the new circuit's seed against the finished race.

Found while verifying the #4 + #5 merge: a blank `BEST LAP --:--.---` in a test turned out to be a genuine reachable state, not just a test artefact.

Fixed by snapshotting rank, best lap, total, drift score, seed and difficulty at the moment of finishing, and cancelling any pending card in `begin()`.

**Verified** by re-seeding during the delay window: the card still renders `0:38.100` / `1:57.966` / `CIRCUIT MOUNTAIN` rather than the reset state. Zero console errors.

---

## #7 — Fix shimmering in the bonnet camera

> "it looks fine when the vehicle is stationary, but when in motion, there seems to be a lot of glittering"

Your instinct that it was about position changing was right, but it wasn't the camera drifting — it was the **near plane cutting through your own car**.

## Cause

The bonnet camera sat at the driver's eye point, which is inside the cabin mesh. Measured:

```
nearestOwnBodywork: 0.000     (camera is inside the geometry)
cameraNearPlane:    0.4
```

Stationary, that clip boundary is static and looks fine. In motion the car pitches and rolls over the terrain, so the near plane sweeps back and forth through your own bodywork every frame — which is exactly the shimmer, and exactly why it only appeared once moving.

## Fixes

- **Hide the shell in first person.** The blob shadow stays, so the car keeps ground contact. Camera nudged 0.55 m forward now that self-occlusion isn't a constraint.
- **Camera shake was white noise.** `Math.random()` per frame, uncorrelated. Nearly invisible from the chase cam, but from inside the car with the look-at point 16 m ahead it adds high-frequency angular noise on top of the clipping. Now a smooth multi-frequency rumble (6–12 Hz), scaled to 35% in first person.
- **Restore visibility in `showResults()`** — `updateCamera` stops running when the race ends, so the car would otherwise stay hidden behind the results card.

## Verified

| | Before | After |
|---|---|---|
| Own bodywork vs near plane | inside it (0.000) | not in view |
| Shell visible in first person | yes | no |
| Blob shadow | — | retained |

## One trade-off

Your car's real cast shadow is absent in first person (hiding a mesh removes it from the shadow pass too). The blob shadow covers ground contact, and you can't see under your own car anyway — but at a low golden-hour sun you might notice the missing long shadow when sliding sideways. If so, the proper fix is render layers (main camera skips the mesh, shadow camera keeps it) rather than plain visibility. Happy to do that if it bothers you.

---

## #8 — Add a way back to the main menu

There was no route from a race back to the title screen — changing circuit, difficulty or hour meant reloading the page. **MAIN MENU** now appears on both the pause and results screens, styled as a ghost button so it reads as the secondary exit rather than competing with RESUME / RACE AGAIN.

`Game.toMenu()` hides the HUD and both overlays, shows the title, re-grids the cars, resets the drift score and idle camera, and repaints the minimap and circuit readout.

## Two details that needed handling

**Stranded audio.** The frame loop only mixes audio while the race is live, so leaving mid-race left the engine and tyre loops at their last gain — they'd have kept droning on the menu. Added `Sfx.silence()`, called *after* `resume()` since gains can't be set against a suspended clock.

**The hidden shell.** First person hides the player's car and `updateCamera` stops running once you leave the race, so visibility is restored explicitly — otherwise quitting from the bonnet cam would leave an empty slot on the title backdrop.

## Verified round trip

`race → pause → menu → change circuit + difficulty → race → finish → results → menu`

| Step | Result |
|---|---|
| After MAIN MENU from pause | state `menu`, title shown, HUD off, cars re-gridded, score cleared, shell visible |
| New race with changed settings | state `count`, timer 0, seed `MOUNTAIN`, hour `GOLDEN` |
| Race to finish | results card, best lap `0:38.133` |
| MAIN MENU from results | state `menu`, results hidden, cars reset |
| Audio after leaving | engine / wind / screech / tow all at gain 0 |

Zero console errors.

---

## #9 — Draw the notched corners on outlined panels and buttons

`clip-path` cuts the notch out of an element **including its `border`**. On a filled shape the fill follows the cut and reads correctly — which is why RESUME and RESTART always looked right. On an outlined one the border is sliced clean off at the diagonal, leaving the top-left and bottom-right corners visibly open.

Outlines are now drawn as a **shape** rather than a border: an outer layer in the line colour, with the fill inset by 1px on a slightly smaller notch, so a 1px stroke follows the diagonals too.

## The inner notch size is derived, not eyeballed

For an outer notch `n` with a 1px inset, the two diagonals are the lines `x+y = n` and `x+y = 2+c`. The perpendicular distance between them is `(n − 2 − c)/√2`, so a true 1px stroke needs:

```
c = n − 2 − √2
```

| Element | outer `n` | inner `c` |
|---|---|---|
| `.panel` (every HUD box and card) | 10 | 6.6 |
| `.btn.ghost` (MAIN MENU) | 11 | 7.6 |
| `#gear` | 7 | 3.6 |

Eyeballing this would have given a stroke that looked right on the straight edges but thin or fat on the diagonals.

## Scope

Only the three elements that combine an outline with a notch. Filled buttons are untouched — their notch was already correct, and the two cases now match visually.

## Verified

At 4× magnification on the ghost button and pause card, plus the title card, results card, and all in-race HUD panels (lap, position, standings, minimap, boost, speedometer, gear). Backdrop blur and the inset glow are preserved.

---

## #10 — Correct the notched-outline geometry: the diagonals were still missing

#9 claimed to draw the notched corners but had the inequality backwards, so the diagonals were never actually rendered. You were right to push back.

## The error

`clip-path` **keeps** the half-plane `x+y ≥ d`. So for a stroke to appear at the corner, the inner shape's diagonal must sit further **out** than the outer one — which means a **larger** inner notch, not a smaller one.

```
outer diagonal   x+y = n
inner diagonal   x+y = 2i + c
gap              (2i + c − n) / √2 = w
=>               c = n − 2i + w·√2
```

I had `c = n − 2 − √2`, which put the inner diagonal *inside* the outer one. The parent clip then trimmed the inner shape back to exactly the outer edge, so it covered the corner completely and no stroke could show — while the straight edges still looked right, which is exactly what made it easy to miss.

| | outer `n` | was | now |
|---|---|---|---|
| `.panel` | 10 | 6.6 | **9.41** |
| `.btn.ghost` | 11 | 7.6 | **10.41** |
| `#gear` | 7 | 3.6 | **6.41** |

## How it was verified this time

With an **exaggerated probe** — 60px notch, 8px stroke in magenta — applied to a live HUD panel:

- **Before:** magenta on all four straight edges, nothing across the diagonal
- **After:** magenta continuous around the whole shape including the diagonal

At that scale the failure is unmistakable. Last time I inspected the 1px case at 4× zoom and read in the result I expected. A deliberately oversized probe removes the judgement call.

---

## #11 — Drive-test generated circuits before accepting them

`measureTrack()` proves a layout is geometrically *legal*. It says nothing about whether it's any *good*. The difficulty sweeps kept surfacing circuits that passed every geometric check and still had the field running wide for 10% of the lap.

A candidate that clears the geometry filter is now **actually driven** — the full grid, headless, ~46 s at PRO through `fastForward` — and rejected if the AI spends too long off-track, gets stranded and respawns, or fails to complete a lap. Budget is 5 tests per seed; if nothing comes back clean the least-bad candidate is used rather than falling all the way back to `CLASSIC`.

This is the payoff from the two earlier features meeting: seeded generation needed a quality signal, and `fastForward` turned out to be one.

## Determinism was the hard requirement

The verdict has to be reproducible, or a seed would yield different circuits on different machines — breaking the property the whole feature rests on. The physics had two nondeterministic inputs:

- the collision yaw kick (`Math.random()`)
- the per-car AI wander phases (drawn in the constructor)

Jitter now runs off a seeded stream reset per race and per test, and the drive test pins the wander phases and difficulty. **Verified**: the same seed reproduces the same circuit *and* the same off-track figure after perturbing the generator with another seed in between.

## Results over 28 seeds

| | Geometry only | Drive-tested |
|---|---|---|
| Mean time off-track | 4.12% | 3.68% |
| Worst circuit | 10.56% | 8.57% |
| **Circuits over 9%** | **2 of 28** | **0** |

It's a **tail-cutting filter, not a broad improvement** — most circuits were already fine, and the mean barely moves. The value is removing the ~7% that were genuinely bad. Cost is ~1.07 drive tests per seed (about 60 ms), no best-effort fallbacks needed.

## It does not homogenise the pool

Worth checking, since rejecting on off-track could quietly bias toward easy flowing layouts. Across 30 seeds, **only the 2 bad circuits changed**:

| | Geometry only | Drive-tested |
|---|---|---|
| Length | 1.83–2.35 km | 1.83–2.35 km |
| Corners | 6–11 (avg 7.9) | 6–11 (avg 7.8) |
| Tightest radius | 46–112 m | 46–112 m |
| Flat-out | 22–54% | 22–54% |

Broken circuits removed, demanding ones untouched.

## Boot order

`generateTrack` now needs the cars and sim loop, so the initial world build moved from before the car definitions to the bottom of the module via `applySeed()`. A `simReady` flag stops the generator attempting a drive test if it's ever called earlier.

## UI

The title card reports it: `… BALANCED · GOLDEN · 4 DRAWS · DRIVE-TESTED 2.18% OFF`, or `BEST OF 5 DRIVEN` in amber on the rare occasion nothing came back clean. `ALPINE` skips the test — it's the curated default and known good, which keeps the default load fast.

---

## #12 — Fade the race out at the flag and loop a results theme

Crossing the line left four engines droning behind the results card. `Game.state` stays `'done'` after finishing, which the frame loop treats as live — so both the simulation *and* the audio mix carried on indefinitely.

## Audio

- **`raceOut()`** eases engine, rival, wind, screech, rumble and tow down over ~0.35 s at the flag.
- The frame loop now **skips the audio mix while `state === 'done'`** — otherwise it rewrote those gains every frame and the fade never took. That was the actual reason a naive fade wouldn't have worked.
- **A four-bar loop fades in 1.2 s later**, under the results card: Am–F–C–G at 92 BPM, soft triangle lead over a sine bass with a quiet sawtooth pad. 10.43 s per loop.
- Notes are scheduled against `ctx.currentTime` **two seconds ahead**, not fired from timers — so the loop stays in time and a throttled tab can't open a gap in it. `setInterval` alone would drift and stutter.
- Stops on restart and on returning to the menu; routed through the master gain so `M` still mutes it.

## One detail worth noting

The first pass put the bar-three bass root at C2 — 65 Hz, essentially silent on laptop speakers. Bass now sits at or above 87 Hz.

## Verified

| | |
|---|---|
| Engine gain 2 s after flag | 0.34 → **0.0006** |
| Rival / wind / tow | **0** |
| Music bus | ramping to 0.5 |
| Scheduler | next loop queued ~10 s ahead, continues advancing |
| Restart / menu | stops cleanly on both |
| Mute | still applies (routed through master) |

Zero console errors.

## What I could not verify

Whether it actually *sounds* good. I checked that it plays, loops seamlessly, sits in an audible register and is mixed at a sensible level — but the melody itself is a judgement call I can't make from here. If it's too busy, too sweet, or the wrong mood for the alpine setting, the score is a small data table (`MUSIC`) near the top of the audio section and easy to rewrite.

The cars keep circulating behind the blurred card, which reads fine now the noise is gone — easy to freeze them instead if you'd prefer.

---

## #13 — Reunite a README paragraph split by the drive-test section

The drive-testing write-up was inserted directly after "A candidate that fails any limit is discarded…", which split that sentence from the rest of its paragraph. "Because the stream is deterministic…" ended up orphaned below the results table — describing the geometry filter while sitting under the drive-test figures. Moved back where it belongs.

Docs only, no code change.

---

## #14 — Personal bests and a ghost car to race against

Nothing persisted before — a good lap vanished when the tab closed, and there was no reason to return to a circuit you liked. That was the biggest remaining gap, and the pieces to close it already existed.

**Bests are stored in `localStorage`, keyed by seed.** That's what makes them meaningful: `JAKDA` is the same circuit on any machine, so a stored time is genuinely comparable rather than a number about a track nobody else can load.

**The lap itself is recorded and replayed as a translucent ghost**, with a live delta on the HUD — `−0.34` green when you're up on your record, `+0.12` magenta when you're down. Toggle on the title card, which also shows your record for the selected circuit. Beating it flashes **★ NEW RECORD**.

## The trace format does two jobs with one array

Sampled at 20 Hz, with each sample carrying its **arc length**:

- index by **time** → where the ghost is right now
- binary-search by **arc length** → how long your best lap took to reach this exact point

That second one is the delta, and it's why arc length is stored rather than derived. Quantised to 0.1 m and 0.001 rad: **~17 KB per lap, 0.03 m replay error, `timeAtS` accurate to 1 ms**. Storage capped at 25 circuits with oldest evicted; quota and private-mode failures degrade quietly instead of throwing.

## Three bugs found while building it

**`Records.load()` merged instead of replacing.** It assigned over the existing object, so stale records survived a storage clear — a reload didn't actually reload. Found because a test cleared storage and the old record was still there, silently rejecting the new submission.

**The arc-length search assumed an unstated invariant.** It only works while `s` rises monotonically through the lap. A trace that ran past the finish line wraps back to zero, and the binary search read it silently wrong — `timeAtS` returned **40.06 s where 1.5 s was correct**. Now truncated on write *and* rejected on read, so the invariant is enforced at both ends rather than assumed.

**The ghost pinned every material to 0.28 opacity** — including the exhaust flames, which normally sit at 0 and are driven per-frame by boost state. They were stuck permanently visible as a solid cone trailing the ghost. Caught by looking at it, not by any assertion.

## Verified

| | |
|---|---|
| Storage round-trip | saves, persists, survives a fresh `load()` |
| Trace | 802 samples, 17.7 KB, covers s 2→2100 of 2103 |
| `timeAtS` across a lap | 0.07 → 8.56 → 18.88 → 39.98 s, monotonic |
| Replay accuracy | 0.026 m position error |
| Guards | wrapped trace rejected on read, truncated on write |
| Submission | slower lap rejected, faster accepted |
| Live delta | `+0.24 → +2.89` as the player loses ground, colour-coded |
| Ghost render | translucent, no flames, no shadow, 25 meshes |

Autopilot laps are never recorded — records should mean you drove it.

---

## #15 — Fix the ghost toggle and add a per-circuit CLEAR

## The toggle already existed — it was broken

The **GHOST ON/OFF** button shipped in #14, but it was `disabled` whenever the circuit had no record yet, and I never styled the disabled state. So it rendered at full opacity, looked live, did nothing when pressed, and read as a missing feature. That's exactly what you hit.

Gating it on having a record was the wrong call regardless — it's a *preference*. You should be able to say you don't want a ghost before you've set a lap.

- Always active now, and the state **persists across sessions** in `localStorage` alongside the records
- Turning it off hides the ghost immediately rather than at the next race
- It leaves the record untouched — toggle and clear are separate concerns

## CLEAR

Wipes the record and ghost trace for the selected circuit, so you can chase it fresh.

- Appears **only when there's something to clear** — no dead button when there's no record (rather than repeating the disabled-button mistake)
- **Two presses**: the first arms it (`ERASE?`, magenta fill) for three seconds, the second commits. It can't be undone, and a stray click shouldn't cost a record — but a modal would be heavy for this
- Clears the stored time, the ghost trace, and the in-memory playback together

## Also

Clamped the delta lookup at lap edges. A trace starts a metre or two past the line and ends a metre or two short, so the readout blanked briefly at each lap boundary. It now resolves to `0` and the full lap time at the extremes, while genuinely out-of-range values still return `null`.

## Verified

| | |
|---|---|
| Toggle with **no** record | enabled, flips label, preference stored |
| Preference after reload | survived |
| Ghost off mid-race | hidden immediately, record kept |
| CLEAR with no record | hidden |
| First press | `ERASE?`, record still stored |
| Second press | record gone, ghost dropped, button hidden, storage updated |
| Delta at lap start / end | `0` / `39.90` instead of blank |

Zero console errors.

---

## #16 — Serve the game at the origin instead of a directory listing

Serving the folder showed a file index at `http://localhost:8123/` rather than the game. `index.html` now hands straight off to `alpine-rush.html`.

## The detail that mattered

`?seed=` has to survive the hop, or a shared circuit link that lands on the origin would silently lose its circuit and drop you on `ALPINE` instead — the failure would look like the seed system being broken rather than the redirect eating it.

```js
location.replace('alpine-rush.html' + location.search + location.hash);
```

- **`search` and `hash` carried across** — shared seed links keep working
- **`replace()` not `assign()`** — the redirect doesn't sit in your back history
- **Dark background** matching the game, so there's no white flash on the hop
- **`<noscript>` fallback link**

## Why not just rename it `index.html`

That would have been one fewer file, but the descriptive filename is worth keeping — the README, your muscle memory and any existing links all point at `alpine-rush.html`. The game is still genuinely a single file; this is twenty lines of server plumbing sitting beside it.

It also means **GitHub Pages would serve the repo root as-is** if you ever publish it — `arjunemirchandani.github.io/alpine-rush` would just work.

## Verified

| | |
|---|---|
| `GET /` | 200, redirect page (878 bytes), no listing |
| `/?seed=JAKDA` | lands on `alpine-rush.html?seed=JAKDA` |
| Circuit loaded | **JAKDA**, 2.29 km, 11 corners |
| Seed field | populated with `JAKDA` |
| Back history | redirect not retained |

---
