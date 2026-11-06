NIP-NX
======

Nexus — Hybrid Peer-to-Peer Relay
---------------------------------

`draft` `optional`

## Abstract

This NIP defines **Nexus**, a hybrid peer-to-peer extension to the standard Nostr
relay protocol. A Nexus relay behaves as an ordinary Nostr relay (NIP-01 over
WebSocket) while additionally coordinating clients to exchange recent events
**directly with each other over WebRTC data channels**. The relay acts as a seed
node and peer coordinator; participating clients cache recent events and serve
them to other clients. Because every transferred event is content-addressed and
signature-verified, any peer can serve it safely. This adds redundancy and
resilience — the network keeps distributing cached events even while the relay is
unreachable — and improves decentralization. Whether it also reduces relay
bandwidth depends on client policy: an implementation that prefers P2P before the
relay saves bandwidth, while a relay-first (additive) implementation trades
bandwidth for a verified second delivery path.

Nexus is fully backwards compatible: clients and relays with no knowledge of this
NIP interact with a Nexus relay as with any NIP-01 relay.

## Motivation

Nostr's data-distribution model routes every event through relays. As usage grows,
popular relays carry the same recent events to many clients over and over. This
NIP proposes that clients opt in to re-distributing the events they already hold,
using the relay only to *coordinate* peers and as a fallback source — a model
inspired by peer-assisted content distribution. The goal is a network that is more
resilient, harder to take down, and less dependent on any single server (and, under
a P2P-first client policy, cheaper to run), without changing how events are
authored, signed, or validated (NIP-01 unchanged).

Nexus belongs to the family of Nostr proposals that favor **network-native, P2P
data sovereignty** over reliance on external HTTP servers. It complements, rather
than replaces, existing relay behavior.

## Terminology

- **Seed Node** — the Nexus relay. Authoritative source of events, peer
  coordinator, and fallback. Stores events like any relay.
- **Super Peer** — a stable, opted-in client that caches recent events and serves
  them to other peers over WebRTC. Subject to a per-peer connection limit.
- **Casual Peer** — a regular client that may receive events from Super Peers but
  does not serve. This is the default role on registration.
- **Reputation** — a persistent score bound to a peer's Nostr public key,
  accrued by serving events reliably. Governs promotion to Super Peer.

## Overview

```
Traditional:   Client ── Relay ── Client
Nexus:         Client ⇄ Client            (P2P via WebRTC data channel)
                    │
                  Relay (seed + coordinator + fallback)
```

1. A client connects to the Nexus relay over WebSocket, exactly as with NIP-01.
2. The client optionally registers as a peer (`PEER_REGISTER`).
3. Events the client receives are cached locally (e.g. IndexedDB, short TTL).
4. The client announces which event IDs it holds (`PEER_CACHE_HAVE`).
5. When another client requests events with P2P preference (`PEER_REQUEST`), the
   relay replies with the peers that hold them (`PEER_OFFER`).
6. The two peers exchange WebRTC signaling **relayed through the seed node**
   (`PEER_SIGNAL`), establish a data channel, and transfer events directly.
7. If P2P fails or times out, the relay serves the events normally. Fallback is
   transparent to the user.

## Signaling Protocol (over the existing relay WebSocket)

Nexus messages are JSON arrays `[TYPE, payload]`, sent over the same WebSocket used
for NIP-01. A relay that does not implement this NIP ignores unknown message types;
message types are namespaced with the `PEER_` prefix to avoid collision with
NIP-01 verbs (`EVENT`, `REQ`, `CLOSE`, …).

### Client → Relay

| Type | Payload | Description |
|------|---------|-------------|
| `PEER_REGISTER` | `{ bandwidth, storage, publicKey }` | Opt in as a peer. `publicKey` binds reputation to a Nostr identity. |
| `PEER_HEARTBEAT` | — | Keep-alive (RECOMMENDED every 30 s). |
| `PEER_REQUEST` | `{ event_ids: [...] }` | Request events, preferring P2P delivery. |
| `PEER_CACHE_HAVE` | `{ event_ids: [...] }` | Announce locally cached event IDs. SHOULD be batched. |
| `PEER_SIGNAL` | `{ target_peer, signal_data }` | Relay WebRTC ICE/SDP to another peer. |
| `PEER_STATS` | `{ events_served, bytes_transferred }` | Report sharing statistics (feeds reputation). |
| `PEER_P2P_CONNECTED` | `{ peer_id }` | Notify that a WebRTC channel was established. |

### Relay → Client

| Type | Payload | Description |
|------|---------|-------------|
| `PEER_REGISTERED` | `{ peer_id, status }` | Registration confirmed; `status` is the assigned role. |
| `PEER_HEARTBEAT_ACK` | `{ peer_id, ts }` | Keep-alive confirmed. |
| `PEER_OFFER` | `{ offers: { event_id: [peer_ids] } }` | Peers holding the requested events. |
| `PEER_SIGNAL_RELAY` | `{ from_peer, signal_data }` | Relayed WebRTC ICE/SDP from another peer. (A peer *sends* `PEER_SIGNAL`; the relay forwards it to the target as `PEER_SIGNAL_RELAY`.) |
| `PEER_PROMOTED` | `{ peer_id, reason }` | Client promoted to Super Peer. |
| `PEER_DEMOTED` | `{ peer_id, reason }` | Client demoted to Casual Peer. |
| `PEER_STATS_OK` | `{ reputation, total_events_served }` | Statistics acknowledged. |
| `PEER_EVENT_NEW` | `{ event_id, kind, size }` | New event available to cache (broadcast to Super Peers). |
| `PEER_RECONNECT` | `{ disconnected_peer }` | A serving Super Peer dropped; re-request via another peer. |
| `PEER_ERROR` | `{ message }` | Error. |

## Data Channel Protocol (over WebRTC)

Once a data channel is open, peers exchange events directly:

| Type | Description |
|------|-------------|
| `P2P_REQUEST` | Request a specific cached event by ID. |
| `P2P_EVENTS` | Deliver a requested event (full NIP-01 event object). |
| `P2P_HEARTBEAT` | Data-channel keep-alive. |

Events transferred over P2P are **ordinary signed Nostr events** and MUST be
validated exactly as if received from a relay (id hash + signature). A peer MUST
NOT trust event content on the basis of the delivering peer; the signature is the
sole source of authenticity. Invalid events MUST be discarded and MAY count
against the sender's reputation.

## Peer Roles, Promotion, and Reputation

- New peers register as **Casual Peers**.
- A peer that stays connected and serves events reliably accrues **reputation**,
  persisted against its `publicKey` (not an ephemeral connection id), and MAY be
  promoted to **Super Peer** (`PEER_PROMOTED`).
- Super Peers are subject to a maximum number of concurrent P2P connections to
  bound resource use; excess demand falls back to the relay.
- Misbehavior (serving invalid events, instability) leads to demotion
  (`PEER_DEMOTED`).

Reputation MAY be used by implementations to allocate incentives (e.g. Lightning
zaps proportional to events served); such incentive mechanics are out of scope for
this NIP.

## ICE / NAT Traversal

Peers behind NAT require STUN/TURN to establish WebRTC channels; symmetric NAT
(common on mobile networks) requires TURN. A Nexus relay SHOULD expose an endpoint
returning short-lived ICE server credentials. Peers that cannot establish a direct
channel fall back to relay delivery.

## Security Considerations

- **Authenticity:** every event, P2P or relayed, is verified by id + signature.
  P2P delivery changes the transport, never the trust model.
- **Privacy:** peers learn the IP addresses of peers they connect to (inherent to
  WebRTC). Clients that require IP privacy SHOULD NOT opt in as peers, or SHOULD
  route via TURN relays. Casual Peers that only fetch reveal less than serving
  Super Peers.
- **Poisoning:** invalid or mismatched events are rejected by signature check and
  penalize the sender's reputation; they cannot corrupt a peer's store.
- **DoS:** registration, signaling, and requests SHOULD be rate-limited per IP and
  per peer; connection counts SHOULD be capped globally and per Super Peer.

## Backwards Compatibility

A Nexus relay is a superset of a NIP-01 relay. Clients that do not implement this
NIP use it as an ordinary relay and never see `PEER_*` traffic. Relays that do not
implement this NIP are unaffected; clients simply find no P2P offers and use
standard relay delivery. Nexus requires no changes to event structure, kinds,
signing, or NIP-01 subscription semantics.

## Reference Implementation

- **Nexus Relay** — <https://github.com/lucianocasalunga/nexus-relay>, live at
  `wss://nexus.libernet.app`, running in production as a standard Nostr relay with
  the peer-coordination layer since April 2026.
- **Client library** — `nostr-p2p.js` (in the same repository).
- **Validation** — end-to-end P2P (browser-to-browser over WebRTC/TURN, event
  transfer with content-addressed verification, credited server-side) was confirmed
  in August 2026. A browser validation lab is served at
  `https://nexus.libernet.app/p2p-lab.html`.

## Prior Art and Relationship to Other NIPs

- **NIP-01** — Nexus is a strict, backwards-compatible extension of it.
- **NIP-11** — a Nexus relay MAY advertise Nexus support in its relay information
  document.
- Data-sovereignty family: unlike HTTP-file approaches (NIP-96 / Blossom), which
  move media *off* Nostr onto external servers, Nexus keeps distribution *inside*
  the Nostr transport and pushes it peer-to-peer.

---

*Author: Luciano Casalunga (Barak) — LiberNet. Reference deployment: Israel / Brazil.*
