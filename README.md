# NIP-95 · Nexus: Hybrid Peer-to-Peer Relay

> My own Nostr proposal: a relay that also coordinates clients to share recent events directly with each other over WebRTC.

**Author:** Luciano "Barak" Casalunga ([@lucianocasalunga](https://github.com/lucianocasalunga)) · [libernet.app](https://libernet.app)
**Full spec:** [NIP-95.md](./NIP-95.md)
**Upstream proposal:** [nostr-protocol/nips#2293](https://github.com/nostr-protocol/nips/pull/2293) (closed without merge on 2026-07-08)
**Running in:** [nexus-peer](https://github.com/lucianocasalunga/nexus-peer) and the LiberNet apps

## The idea

A Nexus relay is a normal Nostr relay (NIP-01 over WebSocket). On top of that, it
acts as a seed node and peer coordinator: clients that opt in cache recent events
and serve them to other clients over WebRTC data channels.

Every event is content-addressed and signature-verified, so any peer can serve it
safely. The network keeps delivering cached events even while the relay is offline.

## Why I wrote it

Popular relays send the same recent events to many clients over and over. Nexus lets
the clients that already hold those events help distribute them, using the relay to
coordinate and as a fallback. The result is a network that is harder to take down
and less dependent on any single server, without changing how events are signed or
validated.

Clients and relays that don't know Nexus keep working with it exactly like any
NIP-01 relay.

## Status

The proposal was closed upstream without merge, but Nexus is not abandoned: it runs in
nexus-peer and the LiberNet apps, and this repository is its home.

## Related LiberNet NIPs

[NIP-43](https://github.com/lucianocasalunga/nip-43) · [NIP-45](https://github.com/lucianocasalunga/nip-45) · [NIP-50](https://github.com/lucianocasalunga/nip-50) · [NIP-66](https://github.com/lucianocasalunga/nip-66) · [NIP-67](https://github.com/lucianocasalunga/nip-67) · [NIP-70](https://github.com/lucianocasalunga/nip-70) · [NIP-77](https://github.com/lucianocasalunga/nip-77) · [NIP-86](https://github.com/lucianocasalunga/nip-86)

---
Nostr since 2021 · Part of the LiberNet ecosystem
