# ANP × libp2p × IPFS Interoperability PoC Proposal

## 1. Goal

The goal is to explore interoperability between ANP, libp2p, and IPFS, allowing agents to combine:

- stable and verifiable identities based on W3C DID;
- decentralized DID Document publication and resolution through IPFS/IPNS;
- secure peer-to-peer connectivity through libp2p;
- a common ANP messaging model across multiple DID methods and transports.

The architecture follows one core principle:

**A DID represents the stable identity of an agent, a libp2p Peer ID represents a network node, and an IPFS CID represents a particular version of content. These identifiers should be cryptographically and verifiably linked, but should not be collapsed into the same identity.**

---

## 2. Overall Architecture

The proposal consists of three layers:

1. **DID Identity Layer** — identifies the agent;
2. **DID ↔ libp2p Binding Layer** — identifies which libp2p peers are currently authorized by that agent;
3. **ANP over libp2p Transport Layer** — carries ANP messages over libp2p.

```text
Agent DID
    │
    ▼
DID Document
    │
    ├── Authentication Key
    │       └── ANP business identity
    │
    └── Peer Verification Method
            │
            └── Peer ID
                  │
                  ▼
             libp2p Endpoint
                  │
                  ▼
             ANP Messaging
```

---

## 3. IPFS/IPNS-Based DID

ANP 1.2 has already decoupled authentication from a specific DID method. It currently supports `did:wba` and `did:web`, while allowing additional DID methods to be integrated.

For an IPFS/libp2p environment, the first step should be to evaluate existing IPFS/IPNS-based DID methods such as `did:ipid`.

If an existing method cannot satisfy agent-specific requirements around identity continuity, updates, revocation, and peer authorization, a new IPFS/IPNS-based DID method can then be defined collaboratively.

Such a method should define at least:

- DID identifier syntax;
- DID Document creation and publication;
- DID Resolution;
- DID Document updates;
- key rotation;
- deactivation;
- caching and rollback protection;
- cryptographic binding between the DID and IPNS;
- DID Document versioning on IPFS.

A recommended architecture is:

```text
Stable DID
    ↓
IPNS
    ↓
IPFS CID
    ↓
DID Document
```

rather than using the current CID itself as the long-term agent identifier.

This allows the DID Document, keys, or network endpoints to change while the Agent DID remains stable.

---

## 4. Binding a DID to a libp2p Peer ID

A DID represents the agent's long-term application identity, while a Peer ID represents a concrete libp2p network node.

They should be verifiably bound, but should not simply be treated as the same identity.

### Recommended Key Model

By default, we propose separating:

```text
DID / IPNS Control Key
        │
        ├── controls DID state and updates
        │
ANP Authentication Key
        │
        ├── authenticates ANP requests and Origin Proof
        │
libp2p Peer Key
        │
        └── derives the Peer ID and authenticates
            the libp2p connection
```

The libp2p Peer Public Key can itself be represented as a `verificationMethod` in the DID Document, while not being included in the DID's `authentication` relationship by default.

For example:

```json
{
  "id": "did:example:agent-a",

  "verificationMethod": [
    {
      "id": "did:example:agent-a#auth-key-1",
      "type": "Multikey",
      "controller": "did:example:agent-a",
      "publicKeyMultibase": "zAUTH..."
    },
    {
      "id": "did:example:agent-a#peer-key-1",
      "type": "Multikey",
      "controller": "did:example:agent-a",
      "publicKeyMultibase": "zPEER..."
    }
  ],

  "authentication": [
    "did:example:agent-a#auth-key-1"
  ]
}
```

In this model:

- `#auth-key-1` proves the ANP business identity;
- `#peer-key-1` proves the network peer identity;
- the two keys can rotate independently.

For the smallest possible PoC, the same key pair could be used both for DID authentication and libp2p peer identity to demonstrate the cryptographic relationship quickly. However, long-term protocol design should keep them separate by default.

---

## 5. ANPLibp2pService

We propose defining a lightweight `ANPLibp2pService`.

The service entry carries network discovery information and explicitly references the Peer Verification Method.

For example:

```json
{
  "service": [
    {
      "id": "did:example:agent-a#anp-libp2p",
      "type": "ANPLibp2pService",
      "serviceEndpoint": {
        "peerId": "12D3KooW...",
        "peerKey": "did:example:agent-a#peer-key-1",
        "multiaddrs": [
          "/dns4/agent.example/tcp/443/wss/p2p/12D3KooW..."
        ],
        "protocols": [
          "/anp/1.0"
        ]
      }
    }
  ]
}
```

This establishes the following trust chain:

```text
Agent DID
    │
    │ authorizes
    ▼
Peer Verification Method
    │
    │ derives / binds
    ▼
Peer ID
    │
    │ reachable through
    ▼
multiaddr
    │
    ▼
libp2p connection
```

A caller can then:

1. resolve the target DID;
2. validate the DID Document;
3. discover `ANPLibp2pService`;
4. obtain the peer key, Peer ID, and multiaddr;
5. verify the relationship between the Peer Public Key and Peer ID;
6. establish a secure libp2p connection;
7. verify that the connected peer matches the peer authorized by the DID;
8. exchange ANP messages over that connection.

---

## 6. ANP over libp2p

ANP 1.2 Core Binding is logically transport-agnostic, so the ANP messaging model itself does not need to be redesigned.

Today:

```text
ANP
 ├── HTTPS
 └── WSS
```

can be extended to:

```text
ANP
 ├── HTTPS
 ├── WSS
 └── libp2p
```

Existing ANP semantics remain unchanged, including:

- JSON-RPC message envelope;
- `sender_did`;
- `target.did`;
- Profiles;
- `operation_id`;
- `direct.send`;
- Origin Proof;
- E2EE Overlays.

A new **ANP over libp2p Transport Binding** would mainly define:

- libp2p protocol ID;
- stream establishment;
- message framing;
- request/response correlation;
- notifications;
- message-size limits;
- timeouts;
- retries;
- backpressure;
- connection authentication;
- DID ↔ Peer ID verification rules.

---

## 7. Two Authentication Layers

The architecture contains two separate authentication layers.

### libp2p Connection Authentication

This answers:

> “Which peer am I currently connected to?”

This is handled through the libp2p Peer ID and secure connection.

### ANP Business Identity Authentication

This answers:

> “Which Agent DID actually originated this ANP message?”

This continues to use ANP DID authentication and Origin Proof.

Therefore:

```text
Peer Key / Peer ID
        │
        ▼
Network Connection Identity


Agent DID / Authentication Key
        │
        ▼
ANP Business Identity
```

The two layers complement rather than replace each other.

This separation also keeps ANP business identity independently verifiable when messages later pass through relays, gateways, or other transport paths.

---

## 8. Initial PoC

The first PoC can be divided into two steps.

### PoC 1: IPFS/IPNS DID × ANP

First validate that an IPFS/IPNS-based DID can fully participate in ANP 1.2.

```text
Agent A
did:web / did:wba

        ↕

Agent B
IPFS/IPNS DID
```

Existing HTTP/HTTPS transport can still be used.

The implementation would include:

- an IPFS/IPNS DID Resolver;
- an ANP DID Method Adapter;
- DID Document validation;
- authentication-key verification;
- bidirectional ANP Direct Messaging.

The goal is to demonstrate:

**Agents using different DID methods can share the same ANP authentication and messaging model.**

---

### PoC 2: ANP over libp2p

The second step adds libp2p transport.

```text
Agent A
   │
   │ target DID
   ▼
Resolve DID Document
   │
   │ discover ANPLibp2pService
   ▼
Peer Verification Method
   │
   │ verify Peer ID
   ▼
libp2p secure connection
   │
   ▼
ANP direct.send
   │
   │ verify Origin Proof
   ▼
Agent B
```

A useful final demonstration would be:

**Agent A uses `did:web` or `did:wba`, while Agent B uses an IPFS/IPNS-based DID. Agent A resolves Agent B's DID Document, discovers its authorized Peer Key, Peer ID, and addresses, establishes a secure libp2p connection, and then sends a standard ANP `direct.send` message.**

This demonstrates that:

1. ANP supports multiple DID methods;
2. Web-based and P2P-based DIDs can interoperate;
3. a DID can securely authorize a libp2p peer;
4. a Peer ID can be verifiably bound to cryptographic material in a DID Document;
5. ANP messaging can operate over libp2p;
6. the agent identity remains independent from a specific network node.

---

## 9. Key PoC Tests

### Multi-DID Interoperability

```text
did:web
    ↕
IPFS/IPNS DID
```

Both sides exchange ANP messages using the same protocol semantics.

### Peer Rotation

```text
Peer Key 1 / Peer ID 1
          ↓
Peer Key 2 / Peer ID 2
```

while:

```text
Agent DID
```

remains unchanged.

The DID Document update revokes the old peer and authorizes the new one.

### Multi-Peer Agent

A single Agent DID may eventually authorize:

```text
Agent DID
   │
   ├── Peer A
   ├── Peer B
   └── Peer C
```

allowing cloud nodes, edge nodes, servers, or future device endpoints to coexist under a stable agent identity.

### Security Failure Cases

The PoC should reject:

- unauthorized Peer IDs;
- a Peer ID that does not match the declared Peer Public Key;
- tampered DID Documents;
- invalid IPNS records;
- revoked peers;
- revoked authentication keys;
- invalid ANP Origin Proofs;
- replay attacks.

---

## 10. Possible Division of Work

### ANP Community

The ANP side could focus on:

- DID Method Adapter;
- ANP DID Authentication;
- Origin Proof;
- ANP Messaging;
- `ANPLibp2pService`;
- ANP over libp2p Binding;
- multi-DID interoperability tests.

### libp2p / IPFS Community

The libp2p/IPFS side could focus on:

- Peer ID / Peer Key model;
- secure connections;
- Peer ID derivation and verification;
- IPFS publication;
- IPNS resolution and updates;
- P2P discovery and connectivity.

### Joint Work

The communities could jointly define:

- an IPFS/IPNS DID method or adaptation profile;
- the Peer Verification Method;
- `ANPLibp2pService`;
- DID ↔ Peer ID cryptographic binding;
- Peer rotation;
- the interoperability test suite.

---

## 11. Future Evolution

After the initial PoC, the work could expand into:

- multi-peer agents;
- multi-device agents;
- ANP Direct E2EE over libp2p;
- ANP Group Messaging over libp2p;
- DHT-based agent discovery;
- Agent Description and metadata publication through IPFS;
- content-addressed agent metadata;
- Web ↔ libp2p gateways;
- a common Agent Identity Layer across different DID methods.

The longer-term architecture could become:

```text
                     Agent DID
                        │
              ┌─────────┴─────────┐
              │                   │
        Web-based DID       P2P-based DID
              │                   │
           HTTPS             IPFS / IPNS
              │                   │
              └─────────┬─────────┘
                        │
                 ANP Identity Layer
                        │
              ┌─────────┴─────────┐
              │                   │
         HTTPS / WSS           libp2p
              │                   │
              └─────────┬─────────┘
                        │
                   ANP Messaging
```

In this model, ANP is tied neither to a particular DID method nor to a particular transport. It becomes a common open protocol layer across heterogeneous agent identity systems and networking technologies.
