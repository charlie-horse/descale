# Descale design

Status: draft · Last updated 2026-10-26

Descale is a WireGuard mesh VPN with a control plane built on ATProto. Nodes connect directly over UDP wherever possible and fall back to relays run by other nodes. There is no coordination server: every node works out the network's state itself from signed records.

## Goals

- WireGuard between every pair of nodes, direct wherever possible.
- Control plane on ATProto: identity, keys, membership and revocation.
- PDS-agnostic: any PDS can host any repo, and no PDS is trusted.
- One user first, a few users later.
- Any node can opt in to relaying for the others.

Not in v1:

- Access rules between nodes. Every approved node can reach every other.
- DNS names for nodes.
- Subnet routes and exit nodes.
- Large networks. The design assumes tens of nodes, not thousands.

## Overview

```
  network repo (did:web:vpn.example.com, on any PDS)
  ├── is.neuros.network.config        signed by the root key
  ├── is.neuros.network.approval/*    signed by admins, one per node, keyed by IPv4
  └── is.neuros.network.revocation/*  signed by admins

  owner repos (any DID, on any PDS)
  └── is.neuros.node.declaration/*    signed by the node itself

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
| Admin | Any ATProto account | One or more **admin keys**, one per admin device, listed in the config. Sign approvals and revocations. |
| Node | A device, owned by an account | **Signing key** (P-256 or K-256). Its `did:key` is the node's ID. Signs the node's declaration and relay requests. |
| | | **WireGuard key** (X25519). |
| | | **Signal key** (X25519). Seals signaling messages. Kept separate from the WireGuard key. |

- Signing keys use ATProto's curves and `did:key` encoding, so existing ATProto crypto libraries work. P-256 also works with hardware key stores such as the Secure Enclave and TPMs.
- The network's DID document also carries the usual `#atproto` key and `#atproto_pds` service for its repo. ATProto ignores `#neuros_root`.
- Members and admins can use any DID method. Only the network DID must be `did:web`.

## Trust model

Everything else follows from four rules:

1. **Authority comes from signatures, never from where a record is stored.** Every Descale record carries its own signature. PDSes, Jetstream, relays and other nodes are only transport.
2. **A PDS is storage.** It can withhold, delay, delete or replay records, but it can't forge them.
3. **Deletes carry no authority.** Deleting a record is housekeeping. Trust ends only through a signed revocation, expiry, or a new config.
4. **Newer wins, and nodes remember.** Configs carry a `seq` and declarations an `updatedAt`. Nodes reject anything older than what they've already seen, and keep revocations until those can no longer matter.

### Signing records

- **What is signed:** the record's DAG-CBOR encoding with the `signature` field removed. `$type` and `network` are part of it, so a signature can't be reused for another record type or another network.
- **Algorithm:** ECDSA over SHA-256, as a 64-byte compact low-S signature. This is the same scheme ATProto uses to sign commits.
- **Signer:** `signature.key` names the signing key as a `did:key`.

## Records

### Network repo

| Collection | Record key | Signed by | Holds |
|---|---|---|---|
| `is.neuros.network.config` | `self` | Root key | Network name, address prefixes, admins with their keys and IPv4 ranges, maximum approval lifetime. |
| `is.neuros.network.approval` | The node's IPv4 address | Admin key | One admitted node: owner, keys, address, relay permission and expiry. |
| `is.neuros.network.revocation` | TID | Admin key | A node signing key that is permanently revoked. |

Each admin logs in to the network account with their own session. A login only grants write access to storage. Authority still comes from the signatures.

### Owner repos

| Collection | Record key | Signed by | Holds |
|---|---|---|---|
| `is.neuros.node.declaration` | TID | The node's signing key | The node's keys, its home relays and, if it relays, its endpoint. Doubles as the join request. |

Declarations are public. The optional `name` field reveals what devices someone owns, so leave it out if that matters.

## Validity rules

A node accepts a peer only if all of these hold:

1. **Config:** signed by the root key in the network's current DID document, with a `seq` at least as high as any seen before.
2. **Approval:**
   - signed by a key that belongs to an admin in the config;
   - its record key equals its `ipv4`, which lies inside the network prefix and that admin's range;
   - `issuedAt` ≤ now < `expiresAt`, and the lifetime is no longer than `maxApprovalSeconds`;
   - no other approval names the same signing key. If two do, both are rejected.
3. **Not revoked:** no revocation signed by a current admin key names the node's signing key.
4. **Declaration** (only needed for relays and signaling): signed by the approved signing key, with matching keys and `network`, and an `updatedAt` at least as new as any seen before.

As a result, removing an admin from the config invalidates every approval that admin signed. Other admins re-approve the nodes that should stay.

## Lifecycles

### Creating a network

1. Generate the root key on an offline device.
2. Create an account for the network DID on a PDS that supports `did:web` accounts. Publish its DID document at `https://<domain>/.well-known/did.json`, containing the PDS's `#atproto` key, the `#atproto_pds` service and `#neuros_root`.
3. Sign the first config (`seq: 1`) with yourself as the only admin, and write it to the network repo.

### Adding a node

1. The device generates its three keys.
2. It writes a declaration to its owner's repo. This needs a login for the owner's account.
3. The owner sends the declaration's `at://` URI to an admin.
4. The admin picks a free address in their range and **creates** the approval at that record key. Creating fails if the address is already taken. Never use put for new approvals, because put overwrites.
5. The device sees its approval and comes up.

### Renewal

Approvals expire, by default after 30 days. An always-on node holding one of an admin's keys re-signs that admin's approvals once less than a third of their lifetime is left. Renewal keeps the same record key and writes a new `issuedAt` and `expiresAt`. Admins can only sign inside their own range, so every admin who approves nodes needs a renewal agent.

### Revocation

1. An admin writes a revocation naming the node's signing key.
2. Nodes pick it up from the network repo (see [Sync](#sync)). The handshake check catches any they missed.
3. Nodes keep a revocation until `createdAt + maxApprovalSeconds`. By then, every approval issued before it has expired.

Revocation is permanent. A device that rejoins must generate new keys. Its old address stays reserved until the revocation is dropped.

### Admin changes

The root signs a new config with a higher `seq`. Removing an admin invalidates their approvals, as described under [Validity rules](#validity-rules).

## Sync

Records carry their own signatures, so nodes can read them from anywhere.

- **Network repo (primary):** poll `com.atproto.sync.getLatestCommit`, for example every 60 seconds, and fetch changes with `com.atproto.repo.listRecords`. Subscribing through Jetstream, filtered to the network DID, gives faster updates.
- **Network DID document:** resolve it again periodically to catch a root key change.
- **Declarations:** fetch from owner repos when needed.
- **Handshake check:** every connection setup carries each side's `configSeq` and a digest of its revocation set. The digest is SHA-256 over the sorted, newline-joined `did:key`s of all revocations it currently holds. On a mismatch, the side that's behind syncs from the network repo before continuing. If the repo is still behind, for example because the PDS is withholding records, the peer sends the missing signed records directly (`is.neuros.signal.defs#records`).
- **Coming back online:** a node that was offline syncs from the network repo before accepting any peer.

## Addressing

- The config sets an IPv4 prefix, e.g. `100.80.0.0/16`, and optionally an IPv6 ULA `/96`. Pick prefixes that won't clash with local networks or other VPNs on the same machines. Tailscale uses `100.64.0.0/10`.
- Each admin gets their own IPv4 range inside the prefix. Ranges don't overlap.
- A node's IPv6 address is the `/96` prefix followed by its 32 IPv4 bits, the same mapping NAT64 uses. One allocation covers both addresses.
- An approval's record key is the node's IPv4 address, so a repo can never hold two approvals for the same address.

## Connectivity

### Relays

A relay is an approved node whose approval has `relay: true` and whose declaration has a `relay` endpoint. It runs:

- STUN (RFC 8489) and TURN (RFC 8656) over UDP. TURN over TCP or TLS is a last resort for strict firewalls.
- The `is.neuros.relay.*` XRPC methods over HTTPS at its endpoint:

| Method | Type | Purpose |
|---|---|---|
| `getIceServers` | query | STUN/TURN URLs, plus short-lived TURN credentials. |
| `sendSignal` | procedure | Deliver a sealed signal to a node listening on this relay. |
| `subscribeSignals` | subscription | Receive sealed signals addressed to the caller. |
| `checkReachability` | procedure | Dial-back test before a node starts relaying. |

Relays can see who signals whom and how much traffic they carry. They can't read signals or WireGuard traffic.

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
- The `payload` is a 24-byte nonce followed by a NaCl `box` from A's signal key to B's. The plaintext is the DAG-CBOR encoding of an `is.neuros.signal.defs` object. Only B can read it, and it proves the message came from A.
- Recipients drop messages more than 60 seconds old by `createdAt`, and messages for sessions they have already finished.

### Connection setup

1. A sends an `offer` with its ICE credentials, candidates and sync state.
2. B runs the handshake check (see [Sync](#sync)), checks A against the [validity rules](#validity-rules), and replies with an `answer`.
3. Both sides send further candidates as they find them (`candidate`). ICE (RFC 8445) tests host, server-reflexive (STUN) and relayed (TURN) candidates and picks the best working pair.
4. That path carries WireGuard. ICE consent checks (RFC 7675) keep it alive. After a network change, the node sends a new offer to restart ICE.

- The offerer takes the ICE controlling role. If both sides send offers at once, the offer from the lower node ID wins and the other side answers it.
- In v1, nodes connect to every peer up front. Connecting on demand can come later.

### WireGuard

- Userspace `wireguard-go`, with a custom `conn.Bind` that sends each peer's packets over its ICE path. Kernel WireGuard can't share ICE's sockets.
- Each peer's allowed IPs are its IPv4 `/32` and IPv6 `/128`.
- MTU 1280, which leaves room for TURN framing.

## Threat model

| Attacker | Can | Can't | Defenses |
|---|---|---|---|
| Any PDS | Withhold, delay, delete or replay records | Forge records | Record signatures, `seq` and `updatedAt`, remembered revocations, handshake check, expiry |
| Relay | See who signals whom and traffic volumes; drop traffic | Read signals or WireGuard traffic; impersonate nodes | Sealed signals, several home relays |
| Stolen device | Use the network until revoked | Stay in after revocation | Revocation, handshake check, expiry |
| Stolen admin key | Approve nodes in that admin's range; revoke any node | Change the admin list | The root removes the admin, which invalidates their approvals |
| Stolen root key | Rewrite the config | Survive a DID document update | Replace `#neuros_root` in the DID document |
| Whoever controls the network's domain | Replace the root key | | This is the trust anchor, so protect the domain, DNS and hosting |

## Open questions

1. **Namespace.** Keep `is.neuros.*`, or move to something like `is.neuros.descale.*` if Neuros will cover more than the VPN?
2. **TLS for relays without a domain.** Pin a certificate hash in the declaration, or require a domain name?
3. **Renew-only admin keys.** Give renewal agents a key that can only extend existing approvals, so a compromised agent can't approve new nodes.
4. **Revocations from removed admins.** These stop counting once that admin is removed. Keep honoring them, or have a remaining admin re-sign them?
5. **Joining.** Is sending an `at://` URI to an admin enough, or should admins discover pending declarations, for example through Jetstream?
6. **Later features.** Access rules, DNS names (handles are a natural fit), connecting on demand.

## Lexicons

| NSID | Type | Lives in |
|---|---|---|
| `is.neuros.defs` | Shared types | |
| `is.neuros.network.config` | Record | Network repo |
| `is.neuros.network.approval` | Record | Network repo |
| `is.neuros.network.revocation` | Record | Network repo |
| `is.neuros.node.declaration` | Record | Owner repo |
| `is.neuros.relay.getIceServers` | Query | Relay |
| `is.neuros.relay.sendSignal` | Procedure | Relay |
| `is.neuros.relay.subscribeSignals` | Subscription | Relay |
| `is.neuros.relay.checkReachability` | Procedure | Relay |
| `is.neuros.signal.defs` | Sealed signal payloads | Node to node |
