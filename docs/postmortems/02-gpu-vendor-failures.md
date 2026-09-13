# Postmortem — the screen share only worked on current NVIDIA cards

**Period:** 2026-08-27 → 2026-08-31 · **Severity:** the headline feature was
unusable for part of the crew · **Status:** two of four causes closed on
hardware. For the other two a fix shipped and **no reading from the affected
machine exists** to confirm it, at the time of writing.

---

## Symptom

The screen share was clean on an RTX 4090. On two different AMD cards — and, by
report only, on an older NVIDIA card that never produced a diagnostic dump — it
produced one of three outcomes: a black screen at the viewer, a stream that froze
a few seconds in, or a picture that moved perfectly smoothly and broke up
constantly.

Three symptoms, and it was tempting to treat them as one bug. They were four
separate faults.

## Why this mattered enough to refuse a fallback

The crew runs mixed hardware. A feature that works on one vendor's current
generation is not a feature for this audience — it is a demo. A degraded
fallback path for AMD was explicitly rejected as a solution: the requirement was
quality parity, because the entire reason the project exists is that the thing it
replaces already degrades.

There is also a design consequence that shaped the search. The encode path uses
**Media Foundation**, which is Windows' own abstraction over NVENC, AMD VCN and
Intel QSV. There is no vendor branch in this codebase and never was. So "AMD is
unsupported" was not an available explanation — anything vendor-specific had to
be a difference in how the *drivers* treated the same calls.

## Ruling out the obvious first

Before any fix, a sweep of structural hypotheses was closed — each one by a
measurement, not by an opinion. Eight are worth listing because all eight are the
kind of thing that sounds right:

| Hypothesis | Verdict |
|---|---|
| AV1-only encode path | Ruled out — H.264 only, no AV1 anywhere |
| No AMD code path | Ruled out — no *vendor* path at all; Media Foundation is the abstraction |
| Unhandled "encoder input full" condition | Its Media Foundation equivalent was real, and was fixed |
| Device mismatch between encoder and capture | Ruled out — one device, handed over explicitly |
| NVIDIA-generation-specific encoder parameters on older cards | Ruled out — every such set is best-effort |
| Adapter / output mismatch | Ruled out |
| HDR / FP16 colour path leaking | Ruled out — no such branch exists |
| Desktop Duplication API regression | Not applicable — this project does not use it |

Two in the same sweep were **not** ruled out, and listing only the eight above
would make the sweep look cleaner than it was: the capture crate hardcodes a
frame-pool depth of one, and the submit/drain path was synchronous until 2.24.13.
Both were real and both were addressed.

Five further hypotheses of my own were killed by measurement before shipping
anything: a slow AMD encoder (it was never reached), the wrong window being
captured (it was named correctly), exclusive fullscreen (it survived an
alt-tab), a capture that never started (one frame *had* arrived — rounding hid
it), and declared-versus-actual frame rate (disproved by a reading with a perfect
frame-rate match).

**The last suspect was the measurement itself, and it was wrong from the
start.** See "the diagnostic that could not describe the failure" below.

## Cause 1 — a multithreading requirement I had never met

An AMD card, the smallest preset the application offers, 1280×720:

```
1280x720@4 (fresh 3 of 4 captured) 41kbps of 2184 asked
· AMDh264Encoder · encode 216.15ms · handler 227.19ms (lock 60.33ms)
```

**216 milliseconds to encode 720p**, on hardware that does it in single digits.
That figure is my own 250 ms input deadline very nearly expiring on every
frame — the encoder was not signalling that it wanted input, so each frame was
fed late or dropped. The rest of the line follows arithmetically: the capture
crate provides a single-buffer frame pool, so the capture rate is
`1000 / handler_ms` → 4 fps → 41 kbps → a black screen at the far end. Every
number in that reading is a consequence of the first one.

The cause: **`ID3D11Multithread::SetMultithreadProtected` was never called.**
Media Foundation drives the D3D11 device from its own threads once the device is
handed over to it, while my capture thread is simultaneously using that same
device for a texture copy and the colour-space scaler. Microsoft requires
multithread protection to be enabled on a device shared this way. Without it the
runtime's serialisation simply is not there and the driver may do as it pleases.

NVIDIA's driver tolerates it — which is exactly why every measurement taken
before this point had looked fine. AMD's does not.

The call is three lines and had never existed. **It is not an AMD workaround: it
is a requirement this project had been violating on every machine, and only one
vendor charged for it.**

**And it was not the remaining cause.** The crew dumps that arrived a day later
carry multithread protection applied on both AMD machines and still show the
failure; the 216/227 ms readings above are from builds that predate the fix
removing the blocking input wait. Cause 1 was a real specification violation and
a real fix. It did not, on its own, explain what was still broken — which is what
sent the search to causes 2 and 3.

### What it also explained

An earlier AMD report — "the picture freezes a few seconds in" — had been
diagnosed as a blocking call and fixed with the 250 ms deadline. That fix was
correct and insufficient: it converted a permanent freeze into a 4 fps trickle,
because the deadline was *hiding* this root cause. A per-stage timing field added
later is what made it visible, and a second field is what connected it to the
frame rate.

## Cause 2 — a dimension limit, not a pixel limit

```
fresh 0 of 1 captured · 0 repeated in 1s
ERROR the capture stopped: AMDh264Encoder refused 4388x1890@120 H.264 (0xC00D6D76)
```

`MF_E_INVALIDMEDIATYPE`. The fitting function capped the pixel **budget** and
never the **dimensions**, so a 21:9 window at the highest preset lands on
4388×1890 — comfortably inside the budget and 292 pixels past what the silicon
accepts. NVENC, AMD VCN and Intel QSV all top out at 4096 for H.264 width.

It presented as "nothing at all, on both vendors" because the encoder is built
on the *first* captured frame, and the error propagated out of the frame handler,
which the capture crate reads as "stop the capture". The room stayed joined, the
audio kept flowing, and every counter read zero.

One cause, three reports that had been filed separately: an ultrawide window
share, a 7680×2160 monitor share, and a highest-preset window share on the 4090
that had otherwise been working. A maximum-dimension clamp closed all three.

## Cause 3 — the right value in the wrong variant type

This one is the most instructive, because it is the case where a fix shipped and
**changed nothing**, and that was the evidence.

An AMD sharer at the 1440p60 preset, with the cap set to 18 Mbps:

```
sharer: 2926x1260@60 (fresh 60 of 119 captured) 36 535 kbps of 17 951 asked
        (encoder says 17 951)
viewer: 2926x1260@21  19 553 kbps  loss 0%
```

**The encoder produced 203% of its cap.** It had accepted the value, stored it,
and answered with it when asked — and then produced twice it. Just over half the
stream reached the viewer, at 0% packet loss, so frames arrived at the decoder
incomplete. **Fluid and pixelated at once is the signature of an encoder
overrunning its budget, not of a bad link** — all 60 frames were encoded and
paced correctly.

Two things in that reading are still unreconciled and are worth naming rather
than smoothing over. **Where the missing 17 Mbps went is not in the log:**
nothing between the encoder and the wire reported a drop and the SFU logged no
`rtpStats` at all in fifteen minutes, so the sender-side pacer shedding what
congestion control would not carry is the obvious candidate and is reasoning, not
a measurement. And **the viewer's frame-rate field reads 21** on a share whose 60
frames were all encoded and paced; "fluid" here is the crew's description of what
they saw, and the receive-side instrument that disagrees with it was new and
barely exercised.

Three theories died to cheap measurements first, in this order: duplicate-frame
overshoot (negligible, ~2 per second), the network (0% loss, no SFU warnings at
all in fifteen minutes), and a version mismatch between the two clients (both
were on the same build; the older client seen checking in was a third machine).

The first fix set the documented peak-bitrate and buffer-size properties
alongside the mean. It changed nothing at all. A ten-second-interval bitrate
trace — a new instrument, added for this — answered in one paste what six single
snapshots had not: the offer from congestion control climbed to the full 18 Mbps
within 40 seconds and sat there contentedly, while what was actually sent ranged
between 36 and 65 Mbps **and varied with the scene**.

That variation is the whole diagnosis. **A CBR encoder does not swing 36→65 with
content** — it holds its rate and moves quality instead. The encoder had never
been in CBR mode, which is also why setting two parameters *of* that mode had no
effect.

The cause: the rate-control-mode constant was being passed as a **VT_I4**
variant, because the Rust bindings type that enum as `i32`, where Microsoft
documents the property as **VT_UI4**. NVIDIA's implementation coerces the wrong
type. AMD's refuses it. And the call discarded its result code, like every other
best-effort property set in that block — so nothing anywhere reported the
refusal.

**Storing a property and answering with it is not the same as obeying it.** The
read-back that was being used to confirm the setting had applied was confirming
nothing.

Frame generation went the same way, by measurement rather than argument:
runs with it on and off both read ~38 Mbps against an 18 Mbps cap and looked
identical to the viewer. The capture rate halved between them, so the toggle had
definitely taken effect; the bitrate did not move.

## Cause 4 — whole-screen capture of a fullscreen game

A second AMD report the same evening, with a different card, was not the same
bug at all. The first machine's encoder was crawling at 216 ms per frame; this
one never received a frame to encode. `0x0` dimensions meant the encoder was
never built; zero captured frames meant Graphics Capture had delivered nothing.

The tell was in the audio field — `audio live — System` rather than a process
id — because at that point the diagnostics did not yet record *what* had been
shared, and the audio scope was the only trace of it. The share was of the
*whole screen*, which for a fullscreen game is the compositor-bypass case in its
total form. A `source:` line was added to the dump immediately afterwards. The
advice that crew member had already been given — share the game's window, not
the monitor — was correct.

This overlaps with [postmortem 01](01-screenshare-frame-rate.md), where the same
mechanism appears in partial form as a frame-rate cap rather than as a total
failure.

## The diagnostic that could not describe the failure

```
0x0@0 (fresh 0 of 0 captured) 0kbps of 0 asked · 0 repeated in 1787952278s
```

Fifty-six years. The elapsed-time clock was started when the *first frame
arrived*, so a share that captured nothing left it at zero and the elapsed figure
became the raw wall clock. Every rate then divided by 1.8 billion and read as
zero — indistinguishable from an idle share.

Worse: the stall counter read `0`, meaning "nothing wrong", because computing it
required a last-known-good frame and there had never been one. **The single most
important failure a share can have — that it captured nothing at all — was the
one state the diagnostics could not describe and the stall warning could not fire
on.**

Both fixed: the clock starts when the *share* starts, so "no fresh frame ever"
means the entire elapsed time is the stall. A related off-by-one in the warning
text was also wrong in the same direction — a capture limping below one frame per
second rounds to zero, so the threshold is now "below one" rather than "at most
zero".

A third fault in the same instrument gave users actively wrong advice. A
starvation warning — *"Windows is only handing over 4 of the 120 frames a second
you asked for — share the game's window instead"* — fired on a machine where
Windows was withholding nothing at all: our own capture callback was blocked for
435 ms a frame. The warning sent people to change their capture path to fix a
fault in our encode loop.

A separate gap in the same instrument: the diagnostics did not record which build
produced them. An entire evening of readings arrived with no way to tell, and one
of them was taken as proof that a fix had landed when it reads identically on
builds with and without that fix. It was not proof of anything. The build version
is the second line of the dump now.

## Still open

- **No reading from the affected machine exists for cause 3's fix.** The
  "unverified on AMD hardware" note in the log belongs to the *peak-bitrate* fix
  that preceded it — the one that changed nothing. Carrying the label over to its
  successor is an inference, not a measurement. The VT_UI4 fix compiles for the
  real Windows target and is the documented way to set that property, but the
  proof is the affected machine's next share reading near 100% of its cap rather
  than 203%.
- **Cause 2's fix was not confirmed by the crew either** at the time of writing,
  only by reasoning plus the error code it eliminates locally.
- **The older NVIDIA card was never measured.** No diagnostic dump from it
  exists; the one NVIDIA report that did arrive came from a build with no version
  line. Which of causes 1 and 2 its failure was — or whether it was either — is
  not something this log can answer, and saying both were present there would be
  claiming evidence that does not exist.
- **AMD has been broken since native screen share first shipped**, if cause 1 is
  the whole story for that machine. That is a six-week window in which every
  measurement in the log was taken on hardware that forgave a specification
  violation, and the log should be read with that in mind.

## What I took from it

Two of the four causes were my own code violating a documented contract in a way
that one vendor's driver silently forgave. That is a specific and repeatable
failure mode: **a platform API that tolerates a mistake is indistinguishable from
one that has no such requirement, until you meet the implementation that
doesn't.** Both were found only after adding a measurement that could attribute
cost to a single stage — neither was visible in any aggregate.

The other transferable piece is negative: a fix that changes nothing is
information, and it is information that gets thrown away if the fix is shipped
and assumed. The peak-bitrate fix not working is what proved the encoder had
never been in the mode those parameters belong to. If it had been shipped
alongside three other changes, that signal would have been lost.
