# Descale design

Status: draft · Last updated 2026-10-29

Descale is a WireGuard mesh VPN with a control plane built on ATProto. Nodes connect directly over UDP wherever possible and fall back to relays run by other nodes. There is no coordination server: every node works out the network's state itself from signed records.

## Goals

- WireGuard between every pair of nodes, direct wherever possible.
- Control plane on ATProto: identity, keys, membership and revocation.
- PDS-agnostic: any PDS can host any repo, and no PDS is trusted.
- One user first, a few users later.
- Any node can opt in to relaying for the others.

Not in v1, though nothing in the design blocks them:

- Access rules between nodes. Every approved node can reach every other.
- DNS names for nodes. Handles are a natural fit.
- Connecting to peers on demand. v1 connects to every peer up front.
- Subnet routes and exit nodes.
- Large networks. The design assumes tens of nodes, not thousands.

## Overview

```
  network repo (did:web:vpn.example.com, on any PDS)
  ├── is.neuros.descale.network.config        signed by the root key
  ├── is.neuros.descale.network.approval/*    signed by admins (grant + lease), one per node, keyed by IPv4
  └── is.neuros.descale.network.revocation/*  signed by admins

  owner repos (any DID, on any PDS)
  └── is.neuros.descale.node.declaration/*    signed by the node itself

                  every node reads both, and trusts only the signatures

  node A ── sendSignal ──▶ relay R ── subscribeSignals ──▶ node B
  node A ◀═══ WireGuard over the path ICE picks (direct UDP, or TURN via R) ═══▶ node B
```

- **Control plane (ATProto):** signed records say who is in the network and with which keys.
- **Data plane:** WireGuard over UDP paths found by ICE. Relays provide STUN, TURN and a small signaling service used to set up those paths.

## Identities and keys

| Who | Identity | Keys |
|---|---|---|
| Network | A `did:web` with its own repo, e.g. `did:web:vpn.example.com` | **Root key** (`#neuros_root` in the DID document), kept offline. Signs the network config. |
| Admin | Any ATProto account | One or more **admin keys**, listed in the config, each with a role. **Full** keys sign grants, leases and revocations. **Renew** keys sign leases only, for renewal agents. |
| Node | A device, owned by an account | **Signing key** (P-256 or K-256). Its `did:key` is the node's ID. Signs the node's declaration and relay requests. |
| | | **WireGuard key** (X25519). |
| | | **Signal key** (X25519). Seals signaling messages. Kept separate from the WireGuard key. |

- Signing keys use ATProto's curves and `did:key` encoding, so existing ATProto crypto libraries work. P-256 also works with hardware key stores such as the Secure Enclave and TPMs.
- The network's DID document also carries the usual `#atproto` key and `#atproto_pds` service for its repo. ATProto ignores `#neuros_root`.
- Members and admins can use any DID method. Only the network DID must be `did:web`.
- An admin key with any role other than `full` or `renew` can sign nothing. This leaves room for new roles later.

## Trust model

Everything else follows from four rules:

1. **Authority comes from signatures, never from where a record is stored.** Every Descale record carries its own signature. PDSes, firehoses, relays and other nodes are only transport.
2. **A PDS is storage.** It can withhold, delay, delete or replay records, but it can't forge them.
3. **Deletes carry no authority.** Deleting a record is housekeeping. Trust ends only through a signed revocation, expiry, or a new config.
4. **Newer wins, and nodes remember.** Configs carry a `seq` and declarations an `updatedAt`. Nodes reject anything older than what they've already seen, and keep revocations until those can no longer matter.

### Signing records

- **What is signed:** the record's DAG-CBOR encoding with the `signature` field removed. `$type` and `network` are part of it, so a signature can't be reused for another record type or another network.
- **Algorithm:** ECDSA over SHA-256, as a 64-byte compact low-S signature. This is the same scheme ATProto uses to sign commits.
- **Signer:** `signature.key` names the signing key as a `did:key`.
- **Approvals have two signatures:**
  - The grant's signature covers the grant alone, encoded with `$type` set to `is.neuros.descale.network.approval#grant` (whether or not it's stored).
  - The lease's signature covers the whole record, including the grant and its signature.

## Records

### Network repo

| Collection | Record key | Signed by | Holds |
|---|---|---|---|
| `is.neuros.descale.network.config` | `self` | Root key | Network name, address prefixes, admins with their keys, roles and IPv4 ranges, carried-over revocations, maximum lease length. |
| `is.neuros.descale.network.approval` | The node's IPv4 address | Grant: a full admin key. Lease: a full or renew key of the same admin | One admitted node. The grant holds owner, keys, address and relay permission, and never changes. The lease holds the expiry and is re-signed on renewal. |
| `is.neuros.descale.network.revocation` | TID | A full admin key | A node signing key that is permanently revoked. |

Each admin, and each renewal agent, logs in to the network account with their own session. A login only grants write access to storage. Authority still comes from the signatures.

### Owner repos

| Collection | Record key | Signed by | Holds |
|---|---|---|---|
| `is.neuros.descale.node.declaration` | TID | The node's signing key | The node's keys, its home relays and, if it relays, its endpoint and TLS key pin. Doubles as the join request. |

Declarations are public. The optional `name` field reveals what devices someone owns, so leave it out if that matters.

## Validity rules

A node accepts a peer only if all of these hold:

1. **Config:** signed by the root key in the network's current DID document, with a `seq` at least as high as any seen before.
2. **Approval:**
   - the grant is signed by a `full` key of an admin in the config;
   - the lease is signed by a `full` or `renew` key of that same admin;
   - the record key equals the grant's `ipv4`, which lies inside the network prefix and that admin's range;
   - `issuedAt` ≤ now < `expiresAt`, and the lease is no longer than `maxApprovalSeconds`;
   - no other approval names the same signing key. If two do, both are rejected.
3. **Not revoked:** the node's signing key isn't named by a revocation signed with a current `full` key, or by an unexpired `revokedKeys` entry in the config.
4. **Declaration** (only needed for relays and signaling): signed by the approved signing key, with matching keys and `network`, and an `updatedAt` at least as new as any seen before.

## Lifecycles

### Creating a network

1. Generate the root key on an offline device.
2. Create an account for the network DID on a PDS that supports `did:web` accounts. Publish its DID document at `https://<domain>/.well-known/did.json`, containing the PDS's `#atproto` key, the `#atproto_pds` service and `#neuros_root`.
3. Sign the first config (`seq: 1`) with yourself as the only admin, and write it to the network repo.

### Adding a node

1. The device generates its three keys.
2. It writes a declaration to its owner's repo, which needs a login for the owner's account, and shows its [fingerprint](#fingerprints).
3. The admin tool spots the declaration on the firehose (see [Discovering declarations](#discovering-declarations)).
4. The admin checks that the fingerprint shown on the device matches the declaration. A malicious PDS can plant a fake declaration in a member's repo, so this check is what keeps joining safe. A remote member reads their fingerprint to the admin over a channel the admin trusts.
5. The admin picks a free address in their range and **creates** the approval at that record key. The grant is signed with a full key, and the first lease with any of the admin's keys. Creating fails if the address is already taken. Never use put for new approvals, because put overwrites.
6. The device sees its approval and comes up.

### Fingerprints

A node's fingerprint is the first 20 characters of the base32 (RFC 4648) encoding of SHA-256 over its node ID string, shown in four groups of five, e.g. `KQ7MX-4HBRD-2PLWZ-A6TNE`. That's 100 bits, far more than an attacker could match by generating keys. It only covers the signing key, but the declaration is signed by that key, which vouches for the other two.

### Discovering declarations

The admin tool subscribes to `com.atproto.sync.subscribeRepos` on the PDS of each member it knows, starting with just the admin, and picks out `is.neuros.descale.node.declaration` commits from those members.

- **Known members only:** strangers can't flood the admin with join requests. The member list lives in the admin tool's own settings.
- **PDS moves:** on `#identity` or `#account` events, the tool re-resolves the member's DID document and follows them to their new PDS.
- **Alternative:** one connection to a relay's firehose covers every PDS, but carries all of ATProto's traffic.

### Renewal

Approvals expire, by default after 30 days. A renewal agent is an always-on node holding one of an admin's `renew` keys. Once less than a third of a lease is left, it re-signs it, keeping the record key and grant and writing a new `issuedAt` and `expiresAt`.

- **Limited power:** a renew key can't grant or revoke, so a stolen agent can't admit new nodes or knock existing ones offline.
- **One agent per admin:** a lease must be signed by the same admin as its grant, so every admin who grants approvals needs their own agent.
- **Why expiry exists:** it's the only defense when a PDS hides a revocation and a node coming back online reaches the revoked device first. That device can echo back the node's sync state, so the handshake check can't catch it.

### Revocation

1. An admin writes a revocation naming the node's signing key, signed with a full key.
2. Nodes pick it up from the network repo (see [Sync](#sync)). The handshake check catches any they missed.
3. Nodes keep a revocation until `createdAt + maxApprovalSeconds`. By then, every lease signed before it has expired.

Revocation is permanent. A device that rejoins must generate new keys. Its old address stays reserved until the revocation is dropped.

### Admin changes

The root signs a new config with a higher `seq`.

- **Removing an admin, or replacing one of their full keys:**
  - Grants signed with the old key stop being valid, so a remaining full key re-grants the nodes that should stay.
  - Revocations signed with the old key stop counting too. So that removing an admin never brings a revoked node back, the root copies the ones still in force into the same config's `revokedKeys`, each expiring at the original `createdAt + maxApprovalSeconds`. Admin tooling should do this automatically.
- **Replacing a renew key:** leases signed with it stop being valid. List both keys until the agent has re-signed with the new one.

## Sync

Records carry their own signatures, so nodes can read them from anywhere.

- **Network repo (primary):** subscribe to `com.atproto.sync.subscribeRepos` on the network's PDS and keep the commits for the network DID. On reconnect, resume from the last cursor; if the cursor is missing or too old, re-read the collections with `com.atproto.repo.listRecords`. A PDS's firehose carries every repo it hosts, so on a large shared PDS, polling `com.atproto.sync.getLatestCommit` (for example every 60 seconds) is cheaper.
- **Network DID document:** resolve it again periodically to catch a root key change.
- **Declarations:** fetch from owner repos when needed, and again when signaling a node fails.
- **Handshake check:** every connection setup carries each side's `configSeq` and a digest of its revocation set. The digest is SHA-256 over the sorted, newline-joined `did:key`s of all revocations it currently holds, including `revokedKeys` entries. On a mismatch, the side that's behind syncs from the network repo before continuing. If the repo is still behind, for example because the PDS is withholding records, the peer sends the missing signed records directly (`is.neuros.descale.signal.defs#records`).
- **Coming back online:** a node that was offline syncs from the network repo before accepting any peer.

## Addressing

- The config sets an IPv4 prefix, e.g. `100.80.0.0/16`, and optionally an IPv6 ULA `/96`. Pick prefixes that won't clash with local networks or other VPNs on the same machines. Tailscale uses `100.64.0.0/10`.
- Each admin gets their own IPv4 range inside the prefix. Ranges don't overlap.
- A node's IPv6 address is the `/96` prefix followed by its 32 IPv4 bits, the same mapping NAT64 uses. One allocation covers both addresses.
- An approval's record key is the node's IPv4 address, so a repo can never hold two approvals for the same address.

## Connectivity

### Relays

A relay is an approved node whose grant has `relay: true` and whose declaration has a `relay` endpoint. It runs:

- STUN (RFC 8489) and TURN (RFC 8656) over UDP. TURN over TCP or TLS is a last resort for strict firewalls.
- The `is.neuros.descale.relay.*` XRPC methods over HTTPS at its endpoint:

| Method | Type | Purpose |
|---|---|---|
| `getIceServers` | query | STUN/TURN URLs, plus short-lived TURN credentials. |
| `sendSignal` | procedure | Deliver a sealed signal to a node listening on this relay. |
| `subscribeSignals` | subscription | Receive sealed signals addressed to the caller. |
| `checkReachability` | procedure | Dial-back test before a node starts relaying. |

Relays can see who signals whom and how much traffic they carry. They can't read signals or WireGuard traffic.

### TLS

HTTPS and TURN over TLS share one TLS key per relay. That key is separate from the node's signing key.

- **Relays with a domain:** a normal CA-issued certificate, checked the usual way.
- **Relays without one:** a self-signed certificate. The declaration's `relay.tlsKeyPin` holds the SHA-256 of the key's DER-encoded SubjectPublicKeyInfo. When a pin is set, clients accept only a certificate with that key, and skip CA and hostname checks. The declaration is signed by the node, so the pin can be trusted.
- **Renewing:** keep the same key, and the pin doesn't change. A new key means publishing a new declaration.

### Reachability check

Before advertising itself as a relay, a node calls `checkReachability` on an existing relay. The relay only ever probes the caller's observed IP, never an address the caller supplies:

- **UDP:** a STUN Binding request to `udpPort`. Its transaction ID is the caller's 12-byte nonce, so the caller's listener can confirm the probe was real.
- **TCP:** a connection attempt to `tcpPort`.

The probed ports should be the node's relay listener ports. Those ports never send first, so no existing NAT mapping can make a closed port look open.

The first relay in a network has no relay to ask. Configure it by hand. It's usually a server with a public IP.

### Relay authentication

Callers authenticate with a JWT shaped like ATProto service auth, but signed by the node's signing key instead of an account key:

- **Claims:** `iss` is the caller's node ID (`did:key`), `aud` is the relay's node ID, `lxm` is the method's NSID, `exp` is at most 60 seconds after `iat`, and `jti` is random.
- **Checks:** the relay verifies the signature, that `iss` is approved and not revoked, `aud`, `lxm` and expiry, and that it hasn't seen the `jti` before.
- **Subscriptions:** for `subscribeSignals`, the token goes in the `Authorization` header of the WebSocket request.

No PDS is involved, so a PDS can never obtain relay access.

### Signaling

- Each node keeps `subscribeSignals` open on its home relays, up to four, which are listed in its declaration.
- To reach node B, node A calls `sendSignal` on one of B's home relays. The relay either pushes the signal to B or returns `RecipientUnavailable`, in which case A tries the next relay. Relays don't store signals.
- The `payload` is a 24-byte nonce followed by a NaCl `box` from A's signal key to B's. The plaintext is the DAG-CBOR encoding of an `is.neuros.descale.signal.defs` object. Only B can read it, and it proves the message came from A.
- Recipients drop messages more than 60 seconds old by `createdAt`, and messages for sessions they have already finished.

### Connection setup

1. A sends an `offer` with its ICE credentials, candidates and sync state.
2. B runs the handshake check (see [Sync](#sync)), checks A against the [validity rules](#validity-rules), and replies with an `answer`.
3. Both sides send further candidates as they find them (`candidate`). ICE (RFC 8445) tests host, server-reflexive (STUN) and relayed (TURN) candidates and picks the best working pair.
4. That path carries WireGuard. ICE consent checks (RFC 7675) keep it alive. After a network change, the node sends a new offer to restart ICE.

The offerer takes the ICE controlling role. If both sides send offers at once, the offer from the lower node ID wins and the other side answers it.

### WireGuard

- Userspace `wireguard-go`, with a custom `conn.Bind` that sends each peer's packets over its ICE path. Kernel WireGuard can't share ICE's sockets.
- Each peer's allowed IPs are its IPv4 `/32` and IPv6 `/128`.
- MTU 1280, which leaves room for TURN framing.

## Threat model

| Attacker | Can | Can't | Defenses |
|---|---|---|---|
| Any PDS | Withhold, delay, delete or replay records; plant fake declarations | Forge signed records; get a fake declaration approved | Record signatures, `seq` and `updatedAt`, remembered revocations, handshake check, expiry, fingerprint check |
| Relay | See who signals whom and traffic volumes; drop traffic | Read signals or WireGuard traffic; impersonate nodes | Sealed signals, several home relays |
| Stolen device | Use the network until revoked | Stay in after revocation | Revocation, handshake check, expiry |
| Stolen renewal agent | Extend existing leases; delete or withhold network records through its login | Admit new nodes; revoke nodes; outlast revocations | The root replaces the renew key; end the agent's PDS session |
| Stolen full admin key | Grant nodes in that admin's range; revoke any node | Change the admin list | The root replaces the key, which invalidates its grants |
| Stolen root key | Rewrite the config | Survive a DID document update | Replace `#neuros_root` in the DID document |
| Whoever controls the network's domain | Replace the root key | | This is the trust anchor, so protect the domain, DNS and hosting |

## Lexicons

| NSID | Type | Lives in |
|---|---|---|
| `is.neuros.descale.defs` | Shared types | |
| `is.neuros.descale.network.config` | Record | Network repo |
| `is.neuros.descale.network.approval` | Record | Network repo |
| `is.neuros.descale.network.revocation` | Record | Network repo |
| `is.neuros.descale.node.declaration` | Record | Owner repo |
| `is.neuros.descale.relay.getIceServers` | Query | Relay |
| `is.neuros.descale.relay.sendSignal` | Procedure | Relay |
| `is.neuros.descale.relay.subscribeSignals` | Subscription | Relay |
| `is.neuros.descale.relay.checkReachability` | Procedure | Relay |
| `is.neuros.descale.signal.defs` | Sealed signal payloads | Node to node |

### Publishing

The NSIDs live under `neuros.is`. Lexicon resolution looks up a `_lexicon` DNS TXT record for each NSID group and doesn't fall back to parent domains, so publishing needs one record per group:

| Group | TXT record |
|---|---|
| `is.neuros.descale` (shared defs) | `_lexicon.descale.neuros.is` |
| `is.neuros.descale.network` | `_lexicon.network.descale.neuros.is` |
| `is.neuros.descale.node` | `_lexicon.node.descale.neuros.is` |
| `is.neuros.descale.relay` | `_lexicon.relay.descale.neuros.is` |
| `is.neuros.descale.signal` | `_lexicon.signal.descale.neuros.is` |

Each record's value is `did=<DID>`, naming the account whose repo holds the schemas as `com.atproto.lexicon.schema` records keyed by NSID.
