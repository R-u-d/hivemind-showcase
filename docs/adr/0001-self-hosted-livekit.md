# ADR-0001 — Self-hosted LiveKit instead of LiveKit Cloud

**Status:** Accepted · **Date:** 2026-07 (pre-deploy grilling pass)

## Context

The project's number-one feature is a 1440p60 screen share, and one of its
founding constraints is *nothing that charges*. A 1440p share is ~15.6 Mbps and
an SFU sends one copy per viewer, so four people watching is ~62 Mbps of egress.
Egress is exactly what managed real-time-media providers bill for.

The backend already had an endpoint minting LiveKit access tokens from an API key
and secret, so either option was a configuration target rather than a rewrite.

## Options

| Option | For | Against |
|---|---|---|
| LiveKit Cloud | No operations, available immediately, scales past this crew | The free tier covers roughly four gaming evenings a month, then bills — against a hard no-cost constraint |
| Self-hosted LiveKit | Free, unlimited, full access to server configuration | Operations, monitoring and updates are mine |
| Peer-to-peer mesh | No server at all | Every sharer uploads one copy per viewer; at five participants the sharer's upstream is the ceiling, and several German consumer connections cannot do it |

## Decision

Self-hosted `livekit-server`, in the same `docker compose` stack as everything
else, pointed at by the existing token endpoint.

## Consequences

- Operations are mine, including the SFU's UDP port range and TLS.
- The constraint moves from a bill to a bandwidth figure: the server's uplink
  must carry one stream copy per viewer. That figure is what
  [ADR-0004](0004-rented-root-server.md) is about.
- Client-side quality control came free rather than as a custom bitrate
  controller: a ceiling preset plus WebRTC's own degradation preference, with
  LiveKit's dynacast pausing layers nobody is watching.
- **Simulcast is off, and not by choice.** The native share publishes one
  already-encoded stream and the hardware encoder produces a single layer, so the
  SFU can only forward or drop it. That is what makes the fan-out arithmetic
  above a hard wall rather than a quality slope: a saturated uplink degrades the
  share for every viewer, not only for the one who arrived last.
- **Unintended and decisive:** having full access to the server and its logs is
  what made the screen-share investigation possible. The SFU's receiver-side
  statistics turned out to be a better instrument than anything in the client,
  because they are measured at the receiver and exist whether or not anybody
  thought to press "copy diagnostics". Both postmortems lean on them. A managed
  provider would not have handed that over.

## Note

The mesh option was not dismissed on theory. The fan-out arithmetic above is the
whole argument, and the same arithmetic is what later forced the hosting change.
