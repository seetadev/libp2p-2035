# AgentTrust: Decentralised Capability Delegation and Revocation for Autonomous libp2p Nodes

**Track 4 — Agentic Trust Structures**
Submission to the Resilient Networking Research Track, Devcon 8.
Author: Neha Kumari ([@Nkovaturient](https://github.com/Nkovaturient)).

<img width="2172" height="724" alt="ChatGPT Image Oct 27, 2026, 04_49_40 PM" src="https://github.com/user-attachments/assets/01fb231c-4ee2-4aeb-8d8d-57cf4a8550d9" />

- Repository: https://github.com/Nkovaturient/agentrust-libp2p
- Static HTML demo: https://nkovaturient.github.io/agentrust-libp2p/#experiments
- Paper: https://github.com/Nkovaturient/agentrust-libp2p/blob/main/paper/Agentrust-Libp2p.md
- Protocol spec: https://github.com/Nkovaturient/agentrust-libp2p/blob/main/docs/protocol-spec.md
- Threat model: https://github.com/Nkovaturient/agentrust-libp2p/blob/main/docs/threat-model.md
- Claims and canonical data: https://github.com/Nkovaturient/agentrust-libp2p/blob/main/results/experiment-run-canonical.json · https://github.com/Nkovaturient/agentrust-libp2p/blob/main/results/bench-summary.json · https://github.com/Nkovaturient/agentrust-libp2p/blob/main/results/experiment-run-canonical-docker.json

## Research question

- How can autonomous libp2p nodes issue, delegate, verify, and revoke scoped capabilities without a central authority, using only Peer IDs, Noise, and GossipSub?

## What this is really about

- libp2p authenticates who a peer is via Ed25519 Peer IDs and Noise, but leaves what that peer may do to the application layer. AgentTrust fills that authorization gap for autonomous agents without introducing a central server or external key directory.

- **Identity vs. authorization:** `PeerID + Noise` establishes peer identity; `AgentTrust` layers signed capability tokens and chain verification on top, while `GossipSub` carries revocation broadcasts rather than granting trust itself.
- **Scoped proof Spine:** This PoC measures four concrete pillars — capabilities, delegated chains, offline verification, and gossip revocation — while keeping reputation, identity continuity, privacy, and economic Sybil resistance explicitly scoped as follow-ons.

## Research objectives and hypothesis

- We test whether primitives already in `js-libp2p` 2.x are sufficient for peers to grant scoped actions, attenuate or pass them along bounded chains, verify them offline, and withdraw them over the mesh.

  - **Offline delegation:** A signed capability chain of depth 1–8 verifies using only the Ed25519 public keys embedded in each issuer Peer ID, with `verifierConnectedToIssuer: false`, while depth 9 is rejected before signature checks.
  
  - **Revocation and forgery resistance:** An issuer-signed event on `/agentrust/1.0.0/revocations` flips a previously accepted chain to rejected, and cryptographic forgery, chain theft, parent-link splicing, and non-delegatable extensions fail deterministically.

## How autonomous nodes establish verifiable trust (Track 4 alignment)

Track 4 asks how autonomous nodes establish verifiable trust across delegated trust chains, webs of capabilities, and zero-knowledge reputation. AgentTrust implements and measures the capability and delegation layer today, and defines the exact boundary where ZK and reputation fit next.

- **Webs of capabilities and delegated chains (measured):** Nodes issue Ed25519-signed capability tokens (`action`, `scope`, `delegatable`, `expiresAt`) bound to subject Peer IDs and chain them via `parentTokenId` up to 8 hops so downstream verifiers check authority locally without dialing the root issuer.
- **Revocation over the existing mesh (measured):** Any issuer in a chain can broadcast a signed `RevocationEvent` over GossipSub; verifiers record it in their local registry and cascade-invalidate descendant tokens during `TrustManager.verify()`.
- **Zero-knowledge reputation and privacy (analytic / roadmap):** Verification currently requires the full chain, exposing the root issuer plus *d* subject Peer IDs at depth *d* (paper §7.3); proving "valid chain + not revoked + trusted root" without leaking intermediate Peer IDs is framed as a future zk-SNARK extension, not claimed as implemented.

## Architecture and protocol design

Each node runs `js-libp2p` 2.x with Noise, WebSockets, GossipSub, and a `TrustManager` over a single local revocation registry. Two execution harnesses run the exact same five scenarios (`01`–`05`): a five-node in-process mesh (`local`) and a five-container Compose network (`docker`).

- **Self-certifying key resolution:** `publicKeyFromPeerId` extracts the Ed25519 public key directly from the identity multihash (`0x00`) inside the issuer's Peer ID, eliminating any key directory or issuer round-trip.
- **Chain linkage rules:** Every hop must satisfy `token[i].issuer === token[i-1].subject`, `token[i].parentTokenId === token[i-1].tokenId`, `delegatable === true` on non-terminal links, valid canonical-JSON Ed25519 signatures, and no revoked token IDs.
- **Trust bytes vs. lab orchestration:** In Docker mode, an unauthenticated HTTP API triggers `issue`, `delegate`, `verify`, and `revoke` across containers for reproducible benchmarking only; all trust decisions happen over signed tokens and GossipSub messages.

## Main claims and supporting evidence

Every measured claim in paper §6 traces to committed JSON artifacts (`experiment-run-canonical.json` on Node v24.10.0, `bench-summary.json` across 5 local runs, and `experiment-run-canonical-docker.json` on Node v22.23.4) plus 34 unit and integration tests (`npm run verify`).

- **Scenario 01 — Delegation depth (1–8 valid, 9 rejected):** Verification scales linearly at ~0.05 ms per hop locally (depth 1 median `0.081 ms`, depth 8 median `0.416 ms` across 5 runs; `0.076 ms` to `0.402 ms` in the canonical run) with `verifierConnectedToIssuer: false`, while depth 9 rejects in `0.004 ms` on the length check.
- **Scenario 02 & 05 — GossipSub revocation timing:** Single-token revocation flips accept to reject in `2.03 ms` median locally (`2.33 ms` 5-run median of medians) and `4.86 ms` in Docker (which includes two HTTP orchestration calls); concurrent revocations of two distinct tokens both land in `1.78 ms` locally (`1.90 ms` 5-run median) and `3.72 ms` in Docker.
- **Scenario 03 & 04 — Forgery, theft, and chain integrity:** Forged issuer signatures, stolen chains presented by an attacker, extensions of `delegatable: false` tokens, broken issuer/subject links, spliced `parentTokenId` chains, and post-signature action tampering are all rejected while legitimate control chains pass.

## Limitations and way forward

All measurements come from a five-node star topology on a single host; WAN latency, packet loss, and adversarial eclipse of the revocation topic are not measured. Persistence via `revocationStorePath` (JSONL reload + `syncRevocations` republish on heal) is verified in a loopback Jest test (`revocation-partition.test.js`), not claimed as a WAN partition benchmark.

- **Gap 1 — Root of trust and economic Sybil policy (`docs/SCENARIO-06-DESIGN.md`):** Ed25519 Peer IDs solve key lookup without a directory, but v0.1 does not decide which Peer ID may act as a root issuer or cap a flood of fresh Peer IDs requesting grants; Scenario 06 specifies an issuer-local sliding-window rate limit (`accepted === limit`, `issuance limit exceeded`) as the next negative test, followed by a root allowlist experiment.
- **Gap 2 — WAN partitions and revocation completeness:** GossipSub is eventually consistent and leaves a propagation window before revocation arrival; evaluating persistent gossip-replicated revocation across multi-host WAN partitions and eclipse conditions is the primary systems follow-on.
- **Gap 3 — Privacy, identity continuity, and reputation:** Roadmap order after economic Sybil is privacy (hiding intermediate Peer IDs in depth-*d* chains via ZK proofs), identity continuity across Peer ID key rotation, and peer-scored reputation on a dedicated gossip topic.