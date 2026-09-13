# Quickdraw

A reaction duel where the hard part is *not* firing. Single HTML file, no build
step, no dependencies, no external network requests.

## Controls

| Action | Touch | Keyboard |
|---|---|---|
| Draw | Tap anywhere | `Space`, `Enter`, `F` or `↓` |
| Pause | Pause button | `Esc` or `P` |
| Mute | Speaker button | `M` |

One input, one meaning. It is identical on every difficulty — see below.

## The hook — one rule, printed on the lamp

A lamp hangs on a post between you and your rival. It says three things, and
every one of them is a shape as well as a colour, so none of it depends on
seeing colour at all:

| | Ring | Lens | Glyph | Means |
|---|---|---|---|---|
| **Waiting** | dashed, turning | amber | a dot | not yet |
| **Draw** | **solid** | white | a starburst | tap, now |
| **Hold** | **solid** | red | a bar | do nothing |

**Only a solid ring is the call.** That single sentence is the whole rulebook,
and it covers the thing that makes this more than "tap when it goes green":

- A **feint** lights the lens white or red *before* the call settles — with the
  ring still dashed, and the lamp visibly stuttering. Acting on it is a false
  start. A feint always shows the opposite of the truth, so a white flash means
  the call is going to be red.
- A **hold** is a round you win by not moving. Red for three quarters of a
  second and the round is yours.

So the game is a go/no-go task with a bluff in it. Firing fast is worth points;
firing *early* is worth a plate; and the two impulses are the same impulse.

The solid ring also **drains** over exactly the time you are being given, so
the readout is not a description of the rule — it is the rule, running.

## Plates, and earning them back

You start with a stack of **plates** (armour). Every lost round knocks one off,
visibly, and the run ends when the last one goes. **Three clean rounds in a row
bolts a fresh one on**, up to the ceiling shown by the hollow pips.

That is the only way to gain one. You cannot play safe into safety — holding
through red counts toward the streak, but so does drawing, and a run that never
takes a draw round never earns anything.

## The ladder

Rivals arrive in order, each with a name, a colour and a bluffing habit, and
**three wins puts one down**. Losing a round does not reset your progress
against them; it costs a plate. The ladder only goes forward.

Every rival gives you less time than the last. The window falls fast across the
first eight and then keeps creeping down, 18 ms at a time, to a hard floor
below human reaction — which is where a run finally ends, with the hold rounds
as the only ones still winnable.

## Difficulty

One measured number: **the reaction window**, in milliseconds — how long you
have between the lamp settling on white and the rival firing. Median simple
visual reaction is around 250 ms, and 320–380 ms is normal for someone not
warmed up, so:

| | Window (first → eighth rival → floor) | Holds | Feints | Plates (max) | Pays |
|---|---|---|---|---|---|
| **Cruise** | 700 → 470 → 258 ms | 18% | ×0.6 | 4 (6) | ×0.7 |
| **Drive** | 560 → 360 → 198 ms | 26% | ×1.0 | 3 (5) | ×1.0 |
| **Redline** | 460 → 270 → 148 ms | 34% | ×1.35 | 2 (4) | ×1.45 |

**Nothing about the input changes between tiers.** Same tap, same handler, same
response — difficulty is entirely in how long the rival gives you and how often
they lie. The duel geometry is **fixed logical sizes**, not viewport fractions,
so a 9:20 phone and a 16:9 desktop play identically; extra width only buys more
desert.

Each tier keeps three records of its own: best score, furthest rival, and
fastest clean draw. The game also suggests a tier — three runs without taking a
rival down offers the gentler one, three rivals down offers the harder one,
each once.

## Playables compliance notes

- **Initial load ~78 KB**, one file. Limit is 30 MB.
- **Zero external requests.** All art drawn procedurally on canvas, all audio
  synthesised with Web Audio. Verified in `test/run.js`.
- **No copyrighted assets** — no image or audio file in the bundle.
- **Scales to 1:1, 16:9 and 9:16.** Screenshots in `test/shots/`.
- **60 fps** at phone and desktop resolutions, measured under load.
- **`firstFrameReady()` then `gameReady()`**, in that order.
- **Pause and mute obeyed immediately.** Pausing mid-standoff re-deals the
  round rather than resuming it, so a paused lamp is never a lit lamp.
- **Progress saved through `saveData` / `loadData`**, localStorage as fallback.
- **No ads wired up yet.**

## Repo layout

```
index.html                the whole game
.nojekyll                 serve files as-is
quickdraw-playables.zip   bundle for the developer portal
src/body.html             source of truth
build.js                  wraps src/body.html into index.html
test/driver.js            the auto-duellist the other tests share
test/run.js               aspect ratios, external requests, input, pause, perf
test/sdk.js               integration against a mocked ytgame SDK
test/gameplay.js          plate ledger, window / hold / feint differentials
test/probe.js             measures how long a duel actually lasts
test/bisect.js            disables one draw phase at a time and measures
test/shot.js              screenshot capture
```

`node build.js` rebuilds. `node test/gameplay.js 3` runs one section.

### Testing a reaction game without debug hooks

The shipped build has no test affordances, so the tests play it the way a
person does: `test/driver.js` reads one pixel of the lens every frame and
dispatches real pointer events. It cannot see the ring, which makes it a fair
model of a player who has not learned to wait for the call to settle — and it
exercises every losing path as well as every winning one.

The correctness backbone is an identity. A plate is spent by exactly one lost
round and won back only by a clean streak of three, and the run ends when the
last one goes, so on a run that ends on its own:

```
rounds lost  ===  the tier's starting plates  +  plates won
```

Both numbers are on the game-over card. That one equation covers per-tier plate
counts, the loss path and the streak reward together.

Each rule is then tested as a **differential** — the same build with one thing
changed, played by the same driver:

| Claim | Differential | Result |
|---|---|---|
| the window decides a draw | window pinned to 900 ms vs 200 ms, driver fixed at 520 ms | 19 wins / 0 losses vs 1 / 3 |
| red means hold | identical build, driver with and without impulse control | 17% of rounds lost vs 40% |
| a held red is a *win* | every call red, no feints, driver never taps | 17 rounds, 17 won, nothing lost |
| a feint is not a call | rivals at 0.8 feint vs 0, driver reacting in 20 ms | run ends vs never loses a round |

The third of those is the one worth keeping. It is easy to build a hold round
that merely fails to punish you; the test insists it pays.

#### The trap in the harness

The first version of the driver triggered on "the lens pixel changed", and
recorded several 30 ms reaction times — faster than the frame that drew the
lamp. The cause was the screen shake on a lost round: the lamp moves out from
under a fixed sample point and back again, so the driver reads white, other,
white, and schedules a second tap that lands in the *next* round as a false
start. The driver now arms on amber and fires once per arming, which is also a
fair description of what a player does.

### Measuring the fun

`test/probe.js` answers a design question rather than a correctness one: **how
long does a duel actually last, and who wins it?** It plays each tier with two
modelled hands — REGULAR at 270 ± 130 ms, SHARP at 205 ± 70 ms — on a
standoff-compressed build, then times one full-speed run to convert rounds into
minutes. A round costs about **3.7 seconds** on every tier.

| | REGULAR (270 ± 130 ms) | | | SHARP (205 ± 70 ms) | | |
|---|---|---|---|---|---|---|
| | run | rounds won | rival | run | rounds won | rival |
| **Cruise** | 3m 47s | 84% | 18 | 7m 12s | 87% | 34 |
| **Drive** | 2m 23s | 81% | 11 | 3m 29s | 79% | 16 |
| **Redline** | 1m 23s | 77% | 6 | 2m 23s | 79% | 11 |

That is the shape a hypercasual game wants: a couple of minutes a run, several
runs a sitting, and a visible reason to try the next tier up. A high win rate
is correct here — in a reaction duel the losses should feel like *your* misses,
not like a wall.

The first draft failed that question outright. The window levelled off at each
tier's floor, and a REGULAR player on Cruise reached rival 20, won 98% of sixty
rounds and was still going — a run that could not end. The ladder now keeps
descending past the floor, and every run in the table above ended on its own.

## Performance

60 fps at every resolution on the first measurement, the second game in this
portfolio to manage that, because the lessons were already in place: the desert
is baked once per layout at real canvas resolution and blitted 1:1, the lamp
glow is authored at exactly the size it is drawn rather than stretched, every
sprite with a shadow is cached, and no gradient is built per frame.

`test/bisect.js` — disable one draw phase at a time and measure — was written
before the first optimisation and went unused again. That is the best outcome
it can have.
