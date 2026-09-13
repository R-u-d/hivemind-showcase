# Postmortem — the screen share would not go above 60 fps

**Period:** 2026-08-05 → 2026-08-31 · **Severity:** the product's headline feature
was visibly worse than the thing it was built to replace · **Status:** closed,
with three items explicitly left open at the bottom.

---

## Symptom

Every quality preset above 60 fps arrived at the viewer as roughly 60 fps or
less. The setting was accepted and nothing changed. Later in the hunt the
complaint mutated — "not fluid all the time", "hiccups when a lot is happening",
"pixelated all the time" — which mattered, because each phrasing pointed at a
different layer and two of them pointed at layers that turned out to be innocent.

The worst single reading, and the one that shows why this was not a simple
ceiling: at the 120 fps preset, a 3440×1440 share delivered **71 fps of which
57% were duplicate frames** — about 30 fps of real motion. The 120 preset was
measurably *worse than the 60 preset* on the same machine.

That reading has its own cause, and it is deliberately not in the fault table
below. The pacer's still-gate — "has the screen genuinely stopped, so repeat the
last frame?" — was derived from the *requested* frame interval. At the 120 preset
the gate sat at 12.5 ms while the capture source was delivering a real frame
every 14.1 ms, so every gap looked still and a duplicate went in between
essentially every pair of real frames. Fixed in 1.7.2 by taking the longer of the
requested interval and 1.5× the *observed* capture cadence, and confirmed the
same day qualitatively: the viewer decoded 1440p at 90–100 fps. It is recorded
here rather than in the fault table because it was found and closed three weeks
before the hunt below began — and because it is the reading everything after it
was measured against.

## Why it resisted

Between the game's rendered frame and the viewer's eye sits a chain in which
each stage can independently cap the frame rate: the game's own presentation, the
Windows compositor, the capture API, my frame pacer, the scaler, the encoder,
the sender's uplink plus congestion control, and the SFU. A user-visible "it
stutters" is compatible with every one of them, and the first five produce
numbers that look healthy while the stream looks bad.

Two structural problems made this worse than that ambiguity:

**The instrument did not exist.** At the start, the only reading was an overall
bitrate averaged across the whole share. That number cannot distinguish "the
encoder is producing a third of its budget" from "congestion control is still in
its 20–30 second ramp". Several early conclusions were drawn from exactly that
confusion and had to be withdrawn.

**The one reading that existed was wrong.** A frame counter had been pasted into
both call sites of a timestamp helper — the capture handler *and* the pacing
thread's idle loop — so the pacer counted each of its own ticks as a captured
frame. At the 120 preset that thread ticks up to 120 times a second, which was
most of the reported figure. Every capture number from that build was inflated
and had to be discarded.

## What it actually was: six faults, and one that dissolved

In the order they were found, which is close to the reverse of their importance.

| # | Fault | Mechanism |
|---|---|---|
| 1 | Game in exclusive fullscreen | Bypasses the compositor the capture API reads from. Capture receives nothing. A game setting, not mine. |
| 2 | ~~Encoder underspending its bitrate, up to 8×~~ — **withdrawn** | Believed at the time to be CBR budgeting `bitrate / declared_fps` against a nominal rate it was never fed. It was an artefact of my own readout. See below. |
| 3 | The 120 preset overran the uplink | 368 ms of queueing delay against a 13 ms RTT, 1.43–1.49% sustained loss, ~19 loss-driven keyframe requests per minute. |
| 4 | The frame pacer discarded frames — twice | Wrong rule, then the right rule with too small a credit bucket. |
| 5 | Frames stamped with arrival time, not presentation time — **hypothesis, never confirmed** | The capture API delivers in clumps: four frames in 2 ms, then 80 ms of nothing. Recording arrival replays the clumping as judder at the viewer. It explains what four real fixes each failed to explain, and no reading was ever taken that would settle it. |
| 6 | `MinimumUpdateInterval` *was* the capture rate | I asked the capture API for half the requested frame period, believing the parameter to be a floor. It is not. |
| 7 | Windows' "Optimizations for windowed games" | Grants a *focused* borderless-fullscreen game an independent flip, past the compositor. Nothing in the application can reach this. |

### Fault 2 — the one that was not there

This row is kept because deleting it would hide the most useful thing in the
investigation. The 8× deficit came from a single field: total bytes divided by
total seconds, over the *whole* share, compared against the bitrate being asked
for *right now*. Congestion control ramps from about 300 kbps over 20–30 seconds,
so a 43-second share reading "4 987 kbps of 15 091 asked" is a ramp, not an
encoder producing a third of its budget. Three offline encoder runs the same
evening produced 95–100% of target.

So the compensation built for it was compensating for nothing — and it had no
fixpoint, so on a fast link it wound itself up to 8× and produced the
break-up the crew was complaining about. It was deleted, confirmed on hardware.

The mechanism in that row was a *hypothesis* when it was written, and the log
says so in the same breath: the fix shipped for it was "theory-free on purpose …
it does not depend on the declared-frame-rate explanation being right." Reading
it back later as a finding is how a suspicion becomes a fact without anyone
deciding that it should.

### Fault 6 — the one that was mine and mattered most

Two readings from the same evening, same machine, same game, different presets.
Both had been on the table for hours and were never put side by side:

| Preset | Interval I requested | Nominal cap | Actually delivered |
|---|---|---|---|
| 120 | 4 167 µs | 240 fps | **97 fps** |
| 60 | 8 333 µs | 120 fps | **45 fps** |

Delivered against each interval's nominal rate: **40.4%** and **37.5%**. Not
identical — the two are 7% apart, on one reading each — but the same ratio to
within the noise of a single sample, and nowhere near any other explanation. The
capture rate was not limited by the game, the GPU, the encoder, the network or
the pacer. It tracked a number I had chosen, at roughly 40% of its nominal value.

The parameter is not a floor. A comment in my own capture file already cited the
upstream issue saying so, in the words *"asking for 16.66 ms yields ~45 fps"* —
and 45 fps is exactly what my 60 preset was delivering. **The intervals do not
line up:** the quote is about 16.66 ms and I was asking for 8 333 µs, so the
match is on the outcome and not on the input, and I never established which of
the two the upstream issue actually meant. What mattered was that a sentence in
my own file said the parameter does not do what I thought it did, and I had never
read it against my own diagnostic.

Two rows of a table, one minute, settled what five successive theories had not.

The resolution took two more turns, and neither is the obvious one. Simply
*not* setting the parameter made it worse — the capture rate fell to 16 fps.
What restored it, three days later, was asking for a different fixed value: a
2 ms interval, which took delivery from 55 fps to 119. "I removed the parameter"
would be the tidy version of this story and it is not what happened.

### Fault 7 — the one that was not mine at all

The cleanest experiment of the whole investigation, and it took one alt-tab:

| Game focused | Focus moved to the other monitor |
|---|---|
| **~67 fps** | **140+ fps** |

Same share, same game, same second, nothing else changed. Windows grants a
focused borderless-fullscreen game an independent flip straight to the display,
bypassing the compositor that Graphics Capture reads from. Move focus and the
flip is revoked and every frame reappears.

This has a consequence that invalidated part of the measurement log: **a
diagnostic is copied by somebody who has just alt-tabbed to the app in order to
copy it** — by which point the flip is revoked and the capture rate has already
doubled. Numbers taken while playing and numbers taken while reading were
measuring two different operating systems. The fix is a Windows display setting
whose name is unguessable from the symptom; it is now in the in-app share
prelude.

**Confirmed the same evening**, which is what turns the observation into a cause:
with the setting switched off, the capture holds ~140 fps *with the game
focused*. The toggle is the fix, not a workaround for it.

### The cause that is in neither the table nor the code

Once the pipeline was out of the way — frame work down from 74% of wall time to
7%, and every captured frame being encoded — the same machine was measured on two
different games fifteen minutes apart, same preset, same window capture:

| | Well-behaved game, capped at 120 | Uncapped game at 175 |
|---|---|---|
| Frame-arrival evenness (`pace`) | 2.85 ms | **13.05 ms** |
| Frames the capture API delivered | 49 fps | **16 fps** |
| Encode stage | 0.94 ms | **11.02 ms** |

SFU loss at the same moment: 0.03%. The network is not involved, and the encode
figure is not about the encoder — that stage waits on the scaler's GPU work, and
an uncapped game saturates the GPU so completely that our blit queues behind it
for eleven milliseconds. **The share was being starved of GPU by the game it was
sharing.** Capping that game's frame rate gave back 3.6× the frames and 13× the
encode time.

This is the cleanest measurement in the log — one variable, same machine, same
evening — and it is not a fault in this codebase, which is why it is not in the
table. It is also the one the in-app advice had been giving since 1.7.0 and that
I had dismissed that same evening for want of evidence.

### The end of the chain, measured

Same machine, 175 Hz monitor, game capped at 144:

| | Before | After |
|---|---|---|
| Frames the capture API delivered | 55 | **~140** |
| Bitrate requested | 50 000 kbps | **24 187** |
| Of that request, actually produced | 25% | **84–92%** |
| Packet loss at the viewer | — | 0% |

That before/after is three faults together, not one: the capture interval and
Windows' windowed-game optimisation, both above, plus a third that surfaced on
the way — the bitrate request was computed from the preset's *nominal* 4K rather
than the 4.95 megapixels actually being encoded. And an over-large request turned
out not to be harmless — **congestion control does not settle at what the link
carries, it overshoots, induces queueing and collapses.** Asking for 40.5 Mbps
instead of 50 on the same link moved the offer from 7.3 Mbps to the full 40.5.
A 5.5× improvement produced by asking for *less*, measured twice on the same
machine a day apart.

## Why a row of correct fixes did not fix the complaint

Each of faults 1, 3, 4 and 5 was real or plausible, shipped, and improved a
number. None of them could put back frames that were never captured. Faults 3, 4
and 5 all sit *downstream* of fault 6 and were working on frames that had already
been lost; fault 1 sits upstream of the capture API and was a second, independent
way of losing the same frames at the source. Either way the count of real frames
entering the pipeline was set before any of those four fixes could reach it.

This is the part worth keeping. Until the per-stage timings were added, 74% of
the frame budget was spent inside my own code — so no comparison between two
configurations could have meant anything anyway. Two of those costs were
instructive:

- A full-resolution texture copy ran *before* the pacer decided whether to keep
  the frame. At 175 fps that is ~3.5 GB/s of copies whose results were then
  discarded. The copy is submitted asynchronously, so **its own measurement read
  0.00 ms** — the cost appeared in the encoder's timing instead. A cost can be
  invisible in its own measurement and enormous in the next one.
- The sharer's self-preview thumbnail cost **12.91 ms per frame — 30% of the
  entire share's wall time** over a 148-second window, and was measured as high
  as 74 ms on a single call, because a GPU read-back immediately after a copy
  waits for every command queued on a GPU the game is already saturating. Encode
  and preview together accounted for the frame to within 0.02 ms. The comment at
  that call site read *"this is the sharer looking at themselves and must never
  be what slows it down."*

## The advice that was causing the fault

From 1.7.0 the in-app share prelude said two things at once: *pick the whole
monitor, not the game window* and *run the game borderless windowed*. Those
cannot both be followed. Borderless windowed is exactly what makes the
compositor-bypassing flip available, and the first line then sends the user to
the capture path that flip defeats. It stood for six weeks, in the product,
advising people into the fault this postmortem is about. Corrected on
2026-08-28 to *pick the game's window*.

## The two rules that came out of this

**Measure the source before the pipeline.** Frames delivered by the capture API,
and the evenness with which they arrive, are both free to measure and between
them bound how good the share can possibly be. Everything downstream is a
smaller question. One of those two fields answered on its first comparison what
six rounds of reasoning about my own code could not. Nothing prevented it being
added on day one except the assumption that a stutter must be mine.

**Put the existing readings side by side before reaching for a mechanism.** The
decisive comparison in this investigation used two numbers that had both existed
for hours. Reasoning produced five wrong mechanisms in the same period.

A corollary learned the expensive way: **one reading per arm is not an A/B.** A
single sample from each of two configurations, taken at different elapsed times
on a metric that ramps, shipped a wrong default once in this project — the 2.30.0
release notes announced the 120 preset as fixed, and they were written before the
pair of readings existed that showed it was not.

## Deliberately left open

- **Where between 18 and 50 Mbps the link actually degrades.** Two data points
  exist and the useful third was never taken. The preset table must not be
  re-tuned from this data alone.
- **The viewer still records almost nothing.** Share diagnostics are written by
  the sender; a subscriber renders the numbers but does not persist them. So
  "26 fps at the source, 14 at the viewer" could not be confirmed after the fact
  for most of this investigation. A receive-side field was added late and has
  not been exercised much.
- **Two simultaneous viewers were never measured.** Every reading in the log has
  exactly one subscriber. Fan-out is one stream copy per viewer, so this is the
  next thing that can be expected to break.

## What I took from it

The investigation was not short of intelligence; it was short of instruments.
Each round of reasoning was locally sound and globally wrong, because it ran on
numbers that did not add up — and once three fields closed the frame budget to
within 0.02 ms, both remaining culprits turned out to be code this project had
written, one of which reported its own cost as zero.

The transferable part is the order of operations: before fixing, build the
cheapest measurement that can *discriminate between* the candidate causes, and
prefer comparing two readings you already have to explaining one. That is the
same reflex as isolating an interface fault in a migration project, where the
cost of guessing is also paid by someone else.
