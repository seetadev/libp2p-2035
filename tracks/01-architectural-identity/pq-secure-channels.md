# Post-Quantum Key Exchange on Both libp2p Secure Channels

**Track 1 — Architectural Identity** (related: Track 5 — Partition Tolerance)
Submission to the Resilient Networking Research Track, Devcon 8.
Author: Paschal Okwuosa ([@paschal533](https://github.com/paschal533)). Rust Noise implementation: Royyan Zahir ([@royzah](https://github.com/royzah)).

- Repository: https://github.com/paschal533/pq-quic
- Paper (PQ-QUIC): https://github.com/paschal533/pq-quic/blob/main/paper/pq-quic.md
- Claims and data: https://github.com/paschal533/pq-quic/blob/main/CLAIMS.md
- Demo (60 s): https://youtu.be/iUC5QSR8g3k
- Spec: https://github.com/libp2p/specs/pull/727

## Research question

What does post-quantum key exchange cost libp2p on each of its two secure channels, Noise over TCP and
TLS 1.3 inside QUIC? And does it change how connections behave when the network is hostile: under packet
loss, during DCUtR hole punching, and against an on-path attacker who tries to downgrade the handshake?

## Why it is a Track 1 question

The two channels reached post-quantum security in opposite ways, and that says something about where
libp2p's abstraction boundaries sit.

- **QUIC:** it arrived "for free". The hybrid group X25519MLKEM768 is already on by default in go-libp2p,
  rust-libp2p, js-libp2p, nim-libp2p and Chrome WebTransport, with no libp2p change. The measurable effect
  sits in the transport's loss-recovery timers, not in the KEM.
- **Noise:** libp2p owns this handshake, so it needed a spec, a new protocol ID
  (`/noise-mlkem768-hfs/0.2.0`) and an implementation in every language. The cost is set by each
  language's ML-KEM library.

## Results

| | Result | Grade |
|---|---|---|
| Census | Every libp2p QUIC stack tested, plus Chrome WebTransport, negotiates X25519MLKEM768 | verified |
| No loss | Hybrid minus classical median within 1 ms in every delay cell, go and rust | verified |
| Loss (Linux netem) | rust hybrid faster by 32.6 ms (2%) and 78.4 ms (10%); 39.9 ms at 5%, CI crosses zero. go: +1.3 / −2.3 / +22.3 ms at 2 / 5 / 10% | strong pilot |
| First datagram lost | hybrid 149.7 ms (go) / 147.4 ms (rust) vs classical 268.5 / 1,077.3 ms | pilot |
| Downgrade | 4 of 4 live ClientHello rewrites end with no session | verified |
| Hole punching | go-libp2p DCUtR across two Linux NATs: 100/100 direct for hybrid and classical at every skew 0–1,000 ms; hybrid 1–2 ms slower (`unified` NAT model) | verified, go only |
| Noise interop | TypeScript, Python, Nim and Rust implementations: 48 of 48 pairings | verified |
| Noise cost | hybrid/classical handshake time 1.13× (Nim), 1.24× (Rust), 1.51× (JS); Python 10.7× with kyber-py, about 1.4× with a C-backed ML-KEM; +2,288 bytes per handshake | Noise paper (not yet public) |

Why loss flips sign between stacks: the hybrid ClientHello (≥1,446 bytes) no longer fits in one QUIC
Initial, so every handshake starts with two packets. The second packet draws an early ACK, so a lost first
packet is detected by RFC 9002's time threshold instead of a probe timeout. quinn's first probe timeout is
999 ms and quic-go's about 200 ms, so rust gains much more than go. Swapping the timers flips the effect.

## Security

Hybrid key exchange protects recorded traffic against a future quantum attacker (store-now-decrypt-later);
authentication stays classical. On QUIC, TLS 1.3 transcript binding stops downgrade. On Noise,
multistream-select negotiates the protocol ID in the clear, so a strict mode (refuse `/noise`) is a SHOULD
in specs#727.

## Implementations

- Noise: [js-libp2p#3683](https://github.com/libp2p/js-libp2p/issues/3683) (from
  [js-libp2p-noise#665](https://github.com/ChainSafe/js-libp2p-noise/pull/665)),
  [py-libp2p#1310](https://github.com/libp2p/py-libp2p/pull/1310),
  [nim-libp2p#2811](https://github.com/iftech/nim-libp2p/pull/2811),
  [rust-libp2p#6481](https://github.com/libp2p/rust-libp2p/pull/6481)
- QUIC: no libp2p change needed; the measurement harness is in the repository.

## Reuse

The QUIC Initial decoder and UDP tap (any QUIC stack, any TLS group); the paired hybrid/classical benchmark
with seeded loss and delay; the live downgrade proxy; the network-namespace hole-punching testbed with two
NAT models; the Noise XXhfs test vectors and interop harness.

## Limitations

One machine with an emulated network, not a real WAN. The go-libp2p slowdown at 5–10% loss is only partly
explained. None of the implementation PRs has merged and specs#727 is still open. Post-quantum identity
(ML-DSA) is a sizing projection only. Reproduction steps: the repository README, "Reproducing".
