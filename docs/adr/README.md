# Architecture Decision Records

One ADR per non-obvious decision that was actually deliberated. Format: context →
options → decision → consequences. ADRs are not rewritten after the fact;
a superseded decision keeps its body and gains a status line.

| | Decision | Status |
|---|---|---|
| [ADR-0001](0001-self-hosted-livekit.md) | Self-hosted LiveKit instead of LiveKit Cloud | Accepted |
| [ADR-0002](0002-native-capture-encode-path.md) | A native capture and encode path instead of the browser's | Accepted |
| [ADR-0003](0003-pwa-instead-of-tauri-mobile.md) | A PWA for phones instead of a Tauri mobile target | Accepted |
| [ADR-0004](0004-rented-root-server.md) | A rented dedicated server instead of a home box | Accepted |

## What is deliberately not here

**The desktop shell (Tauri).** It was in place before the decision log existed
and there is no record of a weighed comparison against Electron. It is described
in [../architecture.md](../architecture.md) as context. Writing it up as an ADR
would mean reconstructing a deliberation that did not happen in that form, and a
retrofitted ADR is worth less than an honest gap.

The private repository carries a longer decision log, including several entries
that are now dead and marked as such — an embedded VPN mesh that was ripped out
once the server had a real public IPv4, a release feed that was replaced by a
self-hosted endpoint, and an earlier host. Those are operational history rather
than architecture, so they are summarised in
[../deployment.md](../deployment.md) instead of reproduced here.
