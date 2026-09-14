# Architecture

> Sanitised. No hostnames, IP addresses, ports or credentials appear in this
> repository.

## Context

A private, self-hosted community platform for a crew of eight — around five on a
typical evening, and growing — who previously used Discord. Every user is known;
there is no public sign-up, no moderation surface and no multi-tenancy. That
single fact removes most of what makes a platform like this hard, and it is the
reason a one-person project can be in daily production use.

External systems it talks to: Steam (library, wishlist, currently-playing,
achievements), Spotify, YouTube, and a self-hosted ntfy instance for phone push.
Everything else is inside the box.

## Containers

| Container | Responsibility | Technology |
|---|---|---|
| Desktop client | Everything the user sees; hosts the native subsystems | Vite + React 18 + TypeScript, Zustand, in a Tauri shell. Windows installer. |
| `native-share` | Screen capture, scaling, H.264 encode, publishing | Rust: Windows Graphics Capture → `ID3D11VideoProcessor` → Media Foundation → LiveKit Rust SDK |
| Phone client | Notifications and replying; voice if wanted | The same web application as a PWA |
| API | Auth, domain logic, REST, WebSockets, presence | Django 5.2 + Django REST Framework, ASGI |
| Worker + scheduler | Background and periodic work | Celery + beat |
| Datastore | Persistence | PostgreSQL |
| Cache / broker | Celery broker, WebSocket channel layer | Redis |
| Media | Voice, video, screen-share transport | Self-hosted LiveKit SFU |
| Object storage | Avatars, covers, attachments, clips | MinIO, S3-compatible |
| Push | Phone notifications | Self-hosted ntfy |
| Ingress | TLS termination, routing | Caddy |

Sixteen Django apps carry the domain: chat, communities, events, clips, feed,
friends, guides, notifications, deals, watchparty, remoteplay, updates, ai,
spotify, youtube, users.

**No fixture data.** Every UI surface reads and writes through the real backend.
That was not true for the first two months — the desktop client was a complete UI
on fixtures, with no HTTP client and faked login — and replacing that was a
deliberate, sequenced build: an API-client seam first so screens could be
converted one at a time, then real auth, then chat and presence over WebSocket,
then the LiveKit client. The seam came first specifically so that it would never
need to be a big-bang rewrite.

## The screen-share path

This is the only part of the system whose shape is not conventional, and
[ADR-0002](adr/0002-native-capture-encode-path.md) explains why it exists.

```
Windows Graphics Capture  (single-buffer frame pool — the crate's, not a choice)
  └─ frame arrives  [capture thread]
       ├─ presentation timestamp, NOT arrival time     ← postmortem 01, fault 5
       ├─ source-evenness measurement
       ├─ frame pacer: token bucket, 8-frame burst     ← postmortem 01, fault 4
       │    └─ no credit → frame discarded here
       ├─ GPU→GPU texture copy, after the pacer        ← postmortem 01
       └─ scaler: BGRA8 → NV12 on the GPU
            └─ Media Foundation async H.264 encode
                 └─ pre-encoded Annex-B → LiveKit, as a second participant
  └─ pacing thread
       └─ no fresh frame for a while → re-publish the last texture
```

One D3D11 device throughout, no CPU round trip. Audio is a separate path: WASAPI
process loopback → raw PCM → Opus via WebRTC.

**Two structural properties of this path, both load-bearing:**

The capture API is **change-driven** — it delivers one frame per presentation. So
the stream's frame rate is the game's frame rate and cannot exceed it. No preset
can produce frames the compositor never delivered, which is the sentence that
resolves most screen-share complaints.

Encoding happens **inline on the capture thread**, so per-frame work is the
capture ceiling: with a one-buffer pool the capture rate is `1000 / handler_ms`.
The depth of that pool is hardcoded by the `windows-capture` crate, not chosen
here. This is why the per-stage timings in
[postmortem 01](postmortems/01-screenshare-frame-rate.md) mattered so much: at one
point 74% of the frame budget was my own code, holding the lock the capture
handler needs to accept the next frame. It is also no longer the binding
constraint — once that budget was cut, a measured handler of 1.23 ms puts the
ceiling near 810 fps, and every 49–57 fps reading in the log predates that.

## Instrumentation

The thing this project got most wrong and then most right. The screen share now
reports, once a second and copyable from the account screen: captured frames
versus frames actually encoded, bitrate asked for versus offered by congestion
control versus actually produced, the encoder's name, duplicate-frame count,
per-stage timings for scale, encode, frame, copy, pacing, handler, lock and
preview, the source that was shared, GPU/CPU/OS, the build version, and a
ten-second-interval bitrate trace for the first and last five minutes.

Almost none of that existed when the screen share shipped. Each field was added
because a specific argument could not be settled without it, and several of them
answered on their first use what rounds of reasoning had not. Both postmortems
are really about that.

**Two gaps remain.** The *viewer* persists almost nothing, so
sender-versus-receiver disagreements cannot be confirmed after the fact, and
nothing in the log was measured with two simultaneous subscribers.

## Known limits

- **Windows only** for the desktop client, and the screen-share subsystem is
  structurally Windows-bound (Graphics Capture, Media Foundation, D3D11). The
  macOS and Linux release paths were removed once it was clear they were
  shipping a build without the feature the project exists for.
- **H.264 only**, and this is not a preference. Decode happens in the WebView's
  WebRTC stack, where HEVC is unreliable and AV1 may fall to software. The codec
  ladder cannot open up until the receive path also moves native.
- **Fan-out is linear in viewers** — one stream copy each, and **simulcast is
  off because it has to be**: the passthrough path publishes one already-encoded
  stream and the hardware encoder produces a single layer. An SFU with several
  layers can hand a slow viewer a smaller one; with one layer it can only forward
  or drop, so a saturated uplink degrades the stream for *everyone*, not just the
  person who arrived last. Sized for a crew, not for a community.
- **Single server, single region.** No redundancy; backups are the recovery plan
  — exercised once for real, as the 2026-08-23 move was a restore onto a new box.
- **One AMD fix has never been measured on AMD hardware** — see
  [postmortem 02](postmortems/02-gpu-vendor-failures.md).
- **Two simultaneous viewers of a high-bitrate share have never been measured.**
  It is the most likely next failure.
