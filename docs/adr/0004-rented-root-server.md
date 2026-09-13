# ADR-0004 — A rented dedicated server instead of a home box

**Status:** Accepted · **Date:** 2026-08-23 (the stack was moved that day)

## Context

The backend originally ran on a crew member's home machine — a Linux box
with a public IPv4 and router access, which was genuinely sufficient for
everything except the one feature the project is about.

The arithmetic that decided it: **the SFU sends one copy of a share per viewer.**
A 1440p share is ~15.6 Mbps, so four people watching one stream is ~62 Mbps of
upload from the server. The home connection had about 50 Mbit up. The wall was
not CPU, not memory, not disk — it was one number, and it sat below the
requirement with no way to raise it.

A separate measurement corrected a standing assumption along the way and is worth
recording because it stopped a wrong conclusion: the *sharer's* home uplink had
been treated as a constraint in several earlier notes, on the basis of what
congestion control had reached. Measured directly with `iperf3`, it did 95 Mbps
up. Every "the uplink is the limit" line written before that measurement was
reading a floor as a ceiling. The server's fan-out was the real limit.

## Options

| Option | For | Against |
|---|---|---|
| Stay on the home box | Free, already running, already understood | ~50 Mbit up against a ~62 Mbit requirement for four viewers. Also: somebody's house, somebody's router, somebody's power cut. |
| Managed cloud (per-hour compute + metered egress) | Elastic | Metered egress against a gaming evening's traffic is exactly the bill [ADR-0001](0001-self-hosted-livekit.md) exists to avoid |
| Rented dedicated server, flat monthly | Predictable cost, large uplink, unmetered within a fair-use cap | ~EUR 10/month, and a migration |

## Decision

One rented dedicated server: 8 GB ECC RAM, NVMe, dedicated cores, **2.5 Gbit/s**,
unlimited traffic. The whole stack moved there on 2026-08-23 as a single
`docker compose` deployment.

## Consequences

- The uplink stopped being the constraint by roughly forty times over. The
  provider throttles if a 24-hour *average* exceeds 2 TB; a four-hour evening at
  full tilt is ~110 GB, so it is twenty times under, and even the throttled rate
  would carry the whole crew.
- **An embedded VPN mesh was removed** as a direct consequence — Tailscale,
  bundled into the client so nobody had to configure a tailnet by hand. It
  existed to reach a box behind a consumer router; the rented server has a real
  public IPv4, so the whole mechanism became dead weight and was deleted.
- The flat monthly cost — about EUR 10 — is the one place the no-cost constraint
  was traded away, deliberately and for a measured reason, rather than eroded by
  a metered bill.
- The deployment is portable by construction — it was authored as a single
  compose file precisely so that the host could change without the stack
  changing. The move validated that choice.
- The client bakes its API and updater URLs in at build time, which made the
  migration order matter: stand the server up first, move DNS, then build the
  installer once. Building it early would have meant shipping a second one.
