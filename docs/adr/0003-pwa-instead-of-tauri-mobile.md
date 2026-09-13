# ADR-0003 — A PWA for phones instead of a Tauri mobile target

**Status:** Accepted · **Date:** 2026-09-02

## Context

The requirement from the crew was narrow and worth stating exactly, because it is
what makes the decision easy: **be notified while away from the desk, and be able
to answer.** Joining voice as well, and watching a stream if it comes for free.
That is the whole scope.

The web application already worked in a phone browser — messages, voice, streams.
What was missing was the notification and the layout.

## Options

| Option | For | Against |
|---|---|---|
| Tauri mobile target | One toolchain with the desktop app | About half the desktop app is desktop-only Rust — native capture, system monitoring, screenshots, game detection — and each would need a "not here" branch. An iOS build requires a paid Apple developer account, which breaks the project's no-cost constraint. |
| A second native app | Best platform fit | A second codebase, for a one-person project, to deliver notifications and a text reply |
| PWA | The app already runs; only push and layout are missing | No app-store presence; push on iOS has its own constraints |

## Decision

A PWA, with self-hosted ntfy for push.

Phones and small windows get a list-then-detail flow. A tablet or an unfolded
foldable is above the width threshold and gets the unmodified three-pane desktop
layout, with no branch and no second layout to maintain. **The desktop
application does not change at all.**

## Consequences

- No app store, no signing identity, no yearly fee — consistent with the
  constraint that drove [ADR-0001](0001-self-hosted-livekit.md).
- One codebase, one layout system, one set of tests.
- The scope stays honest: this is a remote control and a notification endpoint,
  not the product on a phone. Anything that wants to be a real mobile experience
  would reopen this decision rather than extend it.
- Push depends on a self-hosted ntfy instance, which is one more service to run
  and one more DNS record to move if the host changes.
