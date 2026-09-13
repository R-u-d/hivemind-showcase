# ADR-0002 — A native capture and encode path instead of the browser's

**Status:** Accepted · **Date:** 2026-08-05 (shipped as release 1.7.0)

## Context

The desktop client is a web application in a Tauri shell, so the obvious way to
screen-share is the browser's `getDisplayMedia` plus WebRTC. That was the
implementation for four releases, and it could not reach the product's stated
target of 1440p60.

**The finding that forced this, established by elimination over four releases
rather than assumed:**

- Transport was healthy UDP and not relayed.
- WebRTC's `maintain-framerate` degradation preference was working correctly —
  frames were being held and resolution spent, so a nominal "1440p60" arrived as
  536p.
- The GPU advertises H.264 hardware encoding and the browser's own diagnostics
  page confirmed it.
- The encoder nevertheless reported **OpenH264** — software.

That survived every countermeasure tried: forcing a different H.264 profile
through an SDP transform, disabling the GPU blocklist, and testing the embedded
WebView against desktop Chrome. **Chromium would not hand a `getDisplayMedia`
track to the hardware encoder**, so every browser-engine route topped out at
software H.264.

A second, independent problem pointed the same way. `getDisplayMedia` takes no
source argument by design, and the embedded WebView can only *cancel* or allow
the OS picker — there is no equivalent of the host-side handler Electron
provides. So an in-app source picker with monitor and window thumbnails could
show the choice and then have no way to act on the click, which is worse than
the OS picker.

## Options

| Option | For | Against |
|---|---|---|
| Keep `getDisplayMedia` | Nothing to build; cross-platform | Software H.264, ~536p at the stated target. Fails the requirement. |
| Accept a lower target | Honest, free | The single reason the project exists is that the incumbent's ceiling is too low |
| Native capture + hardware encode, published as a second participant | Hardware encode, full parameter access, an in-app source picker becomes possible | A Rust subsystem on the GPU API surface; Windows-only; a second participant per share to reason about |

## Decision

A Rust crate that captures with Windows Graphics Capture, converts colour space
on the GPU, encodes H.264 with **Media Foundation** — Windows' own abstraction
over NVENC, AMD VCN and Intel QSV, so there is one code path for every vendor —
and joins the same LiveKit room as a second participant publishing one
pre-encoded screen-share track.

One D3D11 device throughout, no pixel copy to the CPU, no round trip.

This was only possible at all because the LiveKit Rust SDK had gained a
**pre-encoded passthrough** path three weeks earlier. Before that the SDK encoded
raw frames with its own bundled WebRTC, which on Windows has no hardware factory
— software H.264 again, by a different route.

## Consequences

Measured machine-to-machine on the day it shipped: 56.9 fps sustained at
2968×1242 with a game running, two duplicate frames in 90 seconds, bitrate
ramping to 15.6 Mbps under an 18 Mbps cap, and the subscriber converging to 58.6
fps decoded at full resolution. The day before, the browser path was capping the
same share at software H.264 and 536p.

- **Windows-only**, and the desktop installer is Windows-only as a result.
- **The publisher is a second connection** with a derived identity, because
  LiveKit drops the previous connection when one identity joins twice. It cannot
  subscribe — subscribing would mean the sharer pulling their own 18 Mbps back
  down their own uplink. One function folds it back onto its owner for presence.
- **A publisher alone in a room cannot work, by construction.** With nobody
  subscribed, WebRTC never allocates bitrate, so it never consults the encoder
  selector and the passthrough encoder is never swapped in. A command-line viewer
  exists solely so that the path can be tested at all.
- **Three ways it fails silently**, all of them now pinned in comments: dropping
  the room event receiver kills the session; the pre-encoded backend is not
  optional, because an automatic choice picks a real encoder which treats an
  encoded access unit as a raw frame and crashes; and the bitrate-allocation
  point above.
- **The receive path is unchanged and is now the binding constraint on any codec
  discussion.** Decode and render happen in the WebView's WebRTC stack and a
  `<video>` element. HEVC is unreliable there and AV1 decode may fall to
  software, so the codec ladder collapses to H.264 until decode also moves
  native — which is a further architectural change, not a fix.
- **The cost of owning this:** the entire surface of both postmortems. Taking the
  capture API, the pacer, the scaler and the encoder in-house meant taking their
  failure modes too, including two specification violations that one GPU vendor's
  driver forgave for six weeks.

That cost was worth paying, because the alternative was not a worse screen share
— it was not having the feature the project exists for. But it is the honest
entry on this side of the ledger.
