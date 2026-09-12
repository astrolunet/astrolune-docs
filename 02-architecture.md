# 2. Network Architecture

## 2.1. The C / C++ separation principle

**Rule:** if code executes on every node on every block (hot path,
millions of calls), it's in C. If code executes rarely, locally on a
developer's machine, or somewhere not time-critical, it can be in C++.

### Pure C

| Component | Why C specifically |
|---|---|
| Consensus core (PoTB: VRF, BFT voting, TBS/TGW scoring, committee selection) | The most frequent and most time-critical code in the system — executes on every block on every validating node |
| Networking/P2P layer (gossip, transport) | Constant high-frequency traffic; minimal overhead is critical |
| Cryptography (signatures, hashes, VDF) | Hot path — signature verification on every incoming transaction; a thin wrapper with no C++ overhead over proven primitives (libsodium or equivalent) |
| Storage engine (disk I/O, chain state) | Direct memory and I/O control, with predictable performance and no hidden allocations |
| VM (smart contract execution) | The hottest path of all — executes on every contract call on every node |

### C++

| Component | Why C++ is acceptable |
|---|---|
| Developer CLI / SDK | Not on the hot path; expressiveness and dev speed matter more |
| RPC/API layer (requests from wallets and dApps) | Orders of magnitude less frequent than internal consensus traffic |
| Indexer/explorer backend | Asynchronous, not time-critical |
| Contract language compiler | Runs locally on a developer's machine, not on a node in consensus — C++'s rich AST/parsing abstractions are justified |

## 2.2. Where the two halves meet

They meet in exactly one place: the public C ABI in `include/astrolune/`,
compiled by both languages. This is a real constraint on what these
headers may contain.

The build enforces the rule mechanically, not by convention:
- `astrolune_add_core_library()` sets `LINKER_LANGUAGE C` and the C
  standard, so a stray `.cpp` file in `core/` breaks the build
  immediately, rather than being discovered months later on an embedded
  target;
- `astrolune_add_tool_library()` is the only helper that adds
  `cpp/include` to the include path, so the core library doesn't even
  see the tooling headers;
- The direction of dependencies is expressed as link edges
  (`al_base` → `al_crypto` → `al_potb`), so a layering violation is a
  link error, not a code review finding.

### C/C++ boundary rules — summary

The full list of rules and their rationale is engineering detail
important to core contributors, but not to validator operations. Key
points:

- No hand-written `extern "C"` — only through the
  `AL_EXTERN_C_BEGIN`/`AL_EXTERN_C_END` macros.
- No C-only keywords (`restrict`, `_Atomic`) in declarations — only
  through portable macros.
- No variable-length arrays (VLAs) — they're not C++, and they're a
  stack-overflow vector at attacker-controlled length.
- Every enum has an explicit sentinel fixing its width to at least 32
  bits — because enums cross the ABI in `al_status` values and in struct
  fields.
- Fixed-width types (`al_u8`…`al_i64`) for anything serialized or hashed
  — a consensus field whose width depends on the data model is a
  ready-made chain split on a platform change.
- **No floating point in a consensus-visible position** — not just a
  language-compatibility rule, but the single most important rule on the
  list. Float results depend on FPU mode, instruction selection, whether
  the compiler folds a multiply-add into FMA, and the platform's libm.
  Two honest validators could compute different weights from identical
  inputs. Every non-integer quantity in the protocol is `al_fixed` (Q32.32
  in `int64_t`).
- Hashes and addresses are structs, not arrays (`al_hash256`,
  `al_address`) — so an address can't accidentally be passed where a
  public key is expected, even though both are 32-byte arrays.

### Error handling across the boundary

The core reports errors through a return value — never through an
exception, `errno`, or a global variable. A function producing a value
takes an output parameter and returns `al_status`. This isn't a style
preference: an exception thrown across an `extern "C"` boundary is
undefined behavior, and the C++ tooling links C archives directly.

### Memory ownership across the boundary

- `al_bytes` / `al_bytes_mut` are non-owning views. The core never takes
  ownership of the caller's memory and never returns memory the caller
  must free.
- Anything with a lifetime comes from an `al_arena` supplied by the
  caller, and stays valid until that arena is reset.
- Nothing in the core stores a pointer to an arena across block
  boundaries.

## 2.3. Current state of the code tree

| Directory | Language | Status |
|---|---|---|
| `include/astrolune/` | C ABI (compiled by both) | complete v1 core headers |
| `src/base/`, `src/crypto/`, `src/consensus/` | C23 | implemented |
| `src/vm/`, `src/state/`, `src/tx/`, `src/chain/` | C23 | implemented |
| `src/node/` | C23 | mempool, block ingestion, durable state/chain storage implemented |
| `src/net/` | C23 | TCP transport, framed wire protocol, peer manager with gossip and range sync |
| `src/rpc/` | C23 | JSON codec and JSON-RPC server over HTTP/1.1 |
| `src/daemon/` | C23 | long-lived node service gluing storage, P2P, and RPC together |
| `src/apps/alnode/` | C23 | CLI: keygen, genesis authoring, offline chain tools, `run` daemon |
| `abi/` | C++23 | boundary enforcement; Trocto/SDK not yet written |
| `tests/c/` | C23 | 16 behavioral/header suites plus a determinism fixture |

## 2.4. Deferred components

- **The storage layer** (large off-chain data storage) — not a
  first-order task; to be revisited later.
- **The proxy layer** (traffic anonymization/anti-censorship) — not a
  first-order task; to be revisited later.

Both have skeleton documents (see
[06-deferred-services.md](06-deferred-services.md)) recording open
questions in writing, so deferral is a recorded decision, not a gap.

## 2.5. Component map

### Build graph

```
al_base
  +-- al_crypto
  |     +-- al_potb
  |     +-- al_state
  +-- al_vm
  +-- al_tx      (al_crypto + al_state + al_vm)
  +-- al_block   (al_crypto + al_potb + al_state + al_vm + al_tx)
        +-- al_node
              +-- al_daemon  (al_net + al_rpc) -> alnode
  +-- al_net    (al_base + al_crypto)
```

`Astrolune::core` is an interface target exposing all seven core static
libraries. The separation enforces dependency direction: VM has no
dependency on state/crypto, state has no dependency on VM. `al_node` is
kept separate because mempool and block-ingestion behavior are local
policy, not consensus ABI. `al_net` moves opaque canonical bytes and never
links consensus; the daemon is the only component that sees node, net,
and rpc together.

| Target | Responsibility |
|---|---|
| al_base | statuses, portability, arena, bytes, fixed-point, resource fees |
| al_crypto | hashes, Merkle, keys, and a selectable signing backend |
| al_potb | pure PoTB scoring, committee, epoch seed, and rewards |
| al_state | immutable node store, two-tier SMT, and state staging |
| al_vm | ALVM decoder, CFG validator, and interpreter |
| al_tx | transaction envelope, host integration, fees, events, and receipts |
| al_block | genesis v2, canonical body, atomic production, and execution |
| al_node | mempool, block ingestion, durable state objects, and the canonical chain log |
| al_net | sockets, framed wire protocol, peer manager with gossip and sync |
| al_rpc | JSON codec and JSON-RPC over HTTP/1.1 |
| al_daemon | event loop gluing storage, P2P, and RPC together; timed block production |
| alnode | CLI: keygen, genesis authoring, offline chain tools, and `run` |

### Source tree

```
include/astrolune/   installable C ABI
src/base/            dependency-free foundation
src/crypto/          hash and crypto backends
src/consensus/       PoTB arithmetic
src/state/           SMT and state staging
src/vm/              ALVM implementation
src/tx/              transactions and the execution host
src/chain/           genesis and the block pipeline
src/node/            local policy, block ingestion, and durable storage
src/net/             P2P transport (sockets, wire, peers)
src/rpc/             JSON-RPC server
src/daemon/          node service event loop
src/apps/alnode/     command-line entry point
tools/               Trocto/Regol compiler (C++23) and the trocto CLI
abi/                 C/C++ boundary and manifest checks
tests/c/             behavioral suites and the determinism fixture
fuzz/                libFuzzer/corpus-runner targets and seed corpus
cmake/                standards, warnings, sanitizers, and target helpers
astrolune-docs/      protocol and engineering specification (separate repository)
```

Peer discovery, transport encryption, and the Trocto compiler remain
future components. Durable storage reaches consensus state only through
the existing `al_state_store` callbacks.

### Verification map

| Area | Verification |
|---|---|
| base/crypto/PoTB | focused CTest suites and published vectors where possible |
| state | roots, order independence, proofs, deposits, staging, and snapshots |
| VM | container/CFG, opcodes, hosts, traps, entry points, and resources |
| transactions | canonical decode, signatures, validation order, fees, and rollback |
| blocks | genesis/body round trip, roots, price transition, and atomicity mismatch |
| node | admission, nonce reservation, payload ownership, pruning, and head rollback |
| decoders | four corpus runners; the same sources become libFuzzer targets |
| determinism | one canonical block digest compared across GCC, Clang, and MSVC |
| contracts | compiled sources execute on the real VM with a mock host (test_lang) |
| ABI | layout, standalone headers, C-linkage symbols, and the manifest of AL_PUBLIC |

## 2.6. Cryptography

Astrolune's cryptography consists of two halves with very different
maturity.

| Layer | File | Status |
|---|---|---|
| SHA-256 | `src/crypto/sha256.c` | **Real.** Verified against NIST vectors. |
| HMAC-SHA256, HKDF | `src/crypto/hmac.c` | **Real.** Verified against RFC 4231 / RFC 5869. |
| Merkle trees | `src/crypto/merkle.c` | **Real.** |
| Key derivation, addresses | `src/crypto/keys.c` | Address derivation is real; keypair derivation depends on the backend. |
| Signatures | `src/crypto/dev_backend.c` or `sodium_backend.c` | Dev stub by default; real Ed25519 when sodium is explicitly configured. |
| VRF and VDF | *(removed)* | **Removed from the codebase.** `src/crypto/dev_vrf_vdf.c` no longer exists. Only ABI-compatible struct stubs (`al_vrf_proof`, `al_vdf_output`) remain in `crypto.h` for layout stability; no `al_vrf_*`/`al_vdf_*` functions ship. The seed committee uses a hash-chain instead (the "remove" branch of the migration checklist below). |

### One hash function: SHA-256

Astrolune uses SHA-256 throughout. One primitive: keeps the
consensus-critical surface small, implementable in ~200 lines of
auditable C, hardware-accelerated on every modern CPU, backed by decades
of analysis. The cost is that it's slower than BLAKE3 on long inputs;
this is accepted because the protocol hashes many small structures
(transactions, headers, tree nodes) rather than a few large ones — a
regime where per-block SHA-256 cost matters least and BLAKE3's tree
parallelism gains the least.

**Domain separation is mandatory.** Every structural hash in the protocol
is tagged. Hashing a transaction and hashing a block header must never
produce the same digest from the same bytes — otherwise an attacker could
present one object where another is expected. `al_hash_tagged` absorbs
`SHA256(tag) || data`, not `tag || data` — the tag is hashed first, to fix
its contribution at exactly 32 bytes.

Two examples of domain separation whose isolation is critical, not just
tidy:

```
AL_TAG_EPOCH_SEED    "astrolune.epoch.seed.v1"
AL_TAG_EPOCH_COMMIT  "astrolune.epoch.commit.v1"
```

The commit-reveal scheme for the epoch seed publishes a commitment in the
first round and the committed value in the second. If both were derived
under the same tag, the published commitment would **equal** the value it
later reveals, the entire seed would be computable from the commit round
alone, and the reveal round would protect nothing.

**Addresses use the full digest.** `al_address_from_pubkey` uses all 32
bytes of the tagged SHA-256, no truncation. A 20-byte address would give
80-bit collision resistance, uncomfortably close to achievable for a
long-lived chain.

### Dev primitives — read before use

The default (dev) backend implements the `al_sign_*` interface via a hash
construction. The optional `ASTROLUNE_CRYPTO_BACKEND=sodium` configuration
replaces key derivation and signing with libsodium Ed25519, and
`al_crypto_is_secure()` returns `AL_TRUE` on that backend — signatures are
real. It still returns `AL_FALSE` on the default dev backend, which remains
forgeable and gated behind `allow_insecure_crypto` /
`--allow-insecure-crypto`.

VRF and VDF are **not implemented and not used by consensus.** No
`al_vrf_*`/`al_vdf_*` functions exist in either backend; `al_crypto_is_secure()`
describes the active *signature* backend only and says nothing about VRF/VDF,
because there is no VRF/VDF code path to describe. The epoch seed committee
uses a hash-chain commit-reveal scheme instead (see the migration checklist
below).

The dev signing implementation is: **deterministic and internally
consistent**, so the node, VM, state machine, and the full test suite
exercise real code paths; **not cryptographically secure** — it does not
implement Ed25519.

What it actually does — a "signature" is two halves:

```
sig[0:32]   H(tag_sig  || pk        || message)   - verified by al_verify
sig[32:64]  H(tag_bind || sk_scalar || message)    - carried, never verified
```

The verified half is computable from public data. **Anyone can forge a
signature for any key**, and anyone reading the source can see how, as long
as the dev backend is in use.

**Why Ed25519 wasn't written from scratch:** Ed25519 isn't hard to
implement incorrectly. A stub that's *obviously* and *loudly* broken is
safer than an implementation that's subtly broken — that's the worst
possible failure mode for a signature scheme, because it's
indistinguishable from a working implementation until it's attacked.

**Guardrails:** the dev signing stub is marked `AL_CRYPTO_INSECURE` in
documentation; `al_crypto_backend()` reports the backend in use;
`al_crypto_is_secure()` reflects the signature backend actually configured
(`AL_FALSE` on dev, `AL_TRUE` on sodium); internal backend tags carry a
`dev` segment, so they never collide with a protocol tag. This makes it
harder to accidentally ship the dev backend to a public network. It doesn't
make it impossible, and the daemon's `allow_insecure_crypto` gate is the
actual enforcement point, not this reporting function alone.

### Migration checklist: dev backend → real Ed25519 (complete for signing; VRF/VDF removed)

Nothing outside `src/crypto/` depends on the construction, so this was
isolated work. Status, in order:

1. **Done:** source selection — the production signing path uses
   libsodium's `crypto_sign_ed25519` implementation.
2. **Done:** dependency added without breaking `tiny` — sodium is an
   explicit configuration, `dev` and `tiny` without the dependency continue
   to work. The production distribution still needs a pinned static build
   of the dependency.
3. **Done:** the key and signing API is implemented against libsodium,
   preserving the existing ABI width (32-byte public, 64-byte secret key).
4. **Done:** explicit rejection of non-canonical signatures — the adapter
   checks `S < L` before calling libsodium.
5. **Done:** the cofactor question is resolved — Astrolune accepts
   libsodium's strict Ed25519 verification semantics, including rejection
   of non-canonical points and small-order components. Alternative node
   implementations must match this acceptance rule.
6. **Resolved by removal, not replacement.** VRF is not part of the
   deployment path. No `al_vrf_*` function ships; `al_vrf_proof` survives
   only as an ABI-layout stub in `crypto.h`.
7. **Done — VDF removed.** `al_vdf_*` has been deleted, not left as a stub
   that looks usable; `al_vdf_output` survives only as an ABI-layout stub.
   The seed committee uses a hash-chain commit-reveal scheme instead.
8. **Done:** `al_crypto_is_secure()` returns `AL_TRUE` on the sodium
   backend. Since VRF/VDF are removed rather than pending, the function
   describes the signature backend only — there is no VRF/VDF status left
   for it to gate on.
9. **Done:** RFC 8032 test vectors added.
10. **Outstanding:** run the full test suite under the `asan` preset on the
    exact release revision in CI — the sanitizer preset exists because
    ASan and UBSan findings turn into non-deterministic execution, which
    for a blockchain means a chain split, not a crash. The `sanitizers` CI
    job is configured; confirming it green on a specific release tag is
    still a release-gate item, not a code gap.

`al_crypto_is_secure()` now correctly distinguishes the two shipped
backends: `AL_FALSE` on the default dev backend (forgeable, gated behind
`allow_insecure_crypto`), `AL_TRUE` on the sodium backend (real Ed25519).
Treat this function as the final deployment gate for signatures; it makes
no claim about VRF/VDF because none remain in the tree.

### Constant-time discipline

Where the outcome of a comparison isn't public, the comparison must not
leak through timing:
- `al_bytes_eq_ct` for secrets, MACs, and signatures;
- `al_potb_epoch_seed_check` is constant-time, because the reveal is
  checked against an attacker-chosen value and the outcome isn't public
  until the round closes;
- `al_secure_zero` wipes key material so the optimizer can't remove it.

The limit of what the current code can promise: the *dev*-backend
operations are hash-based, and their timing doesn't depend on secret
data, so they're incidentally constant-time. A real backend must provide
this property deliberately, and verifying it is part of step 1 above,
not something the surrounding code can guarantee.

## 2.7. Data flow

**Status: current for the v1 core and the validator runtime.** Canonical
validation, execution, gated mempool admission, signed finality, gossip,
and durable local-head tracking are all wired through a working daemon.

### Transaction path

```
canonical bytes
  -> decode shape/size/type
  -> chain and expiry check
  -> exact account nonce
  -> resource limits and worst-case balance
  -> signature verification
  -> staged execution
  -> actual base-fee burn and full tip distribution
  -> receipt and state commit
```

The order is consensus-visible. Pre-validation failures change nothing.
Successful execution, REVERT, and traps are all included outcomes: all
consume the nonce, the actual resource base fee, and the full tip.
REVERT/trap restore staged writes, events, and value transfer before the
charge.

### Block path

```
canonical block
  -> header, parent, and base-price transition checks
  -> transaction root verification
  -> transactions applied in committed order
  -> block resource limit check
  -> state, receipt, and usage commit checks
  -> atomic state commit
```

Any mismatch restores the snapshot from block entry. Genesis supplies the
chain ID, initial root, limits, resource targets and prices, the VM cost
schedule, the storage deposit rate, and PoTB parameters.

### State path

The top-level depth-256 sparse Merkle tree maps tagged address hashes to
account leaves. Each contract owns a second depth-256 tree for tagged
storage keys. Staged updates replace only the working root; commit
publishes it, and rollback drops the reachability of newly created
content-addressed nodes.

### Node ingestion path

```
canonical transaction bytes
  -> canonical decode and signature verification
  -> chain policy, next-height price, and expiry check
  -> continuous per-sender nonce and cumulative balance reservation
  -> its own bounded mempool storage

canonical block bytes
  -> decode into caller-sized scratch storage
  -> atomic block execution in the core
  -> publish local head
  -> prune stale/expired mempool and compact the byte buffer

local proposal
  -> FIFO selection, bounded by declared transaction resources
  -> parent, height, base price, and transaction root derivation
  -> isolated validation/execution pass
  -> state commit, receipt, and resource derivation
  -> canonical encoding and immediate checkpoint restoration
  -> synchronous signing decision, signature, and proposal gossip
  -> PREVOTE/PRECOMMIT quorum
  -> re-execution only after finality

durable block commit
  -> append and sync the finality certificate
  -> sync content-addressed node/value logs
  -> append a checksummed canonical block record
  -> sync the chain log and publish the durable head
```

On restart, a node verifies the genesis-bound manifest, rebuilds the
hash/height indexes, drops a torn finality record, and opens state at the
root of the last complete block. A validator also restores the signing
journal and rejects a conflicting proposal or vote for a previously
signed height/round/phase. Mempool policy is deliberately local and does
not affect block validity. There's no replace-by-fee mechanism yet:
duplicate hashes and conflicting sender/nonce pairs are rejected.

### Deferred paths

Peer discovery, transport encryption, rate-limiting policy, and
fee-based mempool ordering remain node/network-layer work. The P2P
transport now carries handshakes, gossip, and finalized-range sync.
Committee-authorized proposals, voting, and finality are implemented in
`src/consensus/finality.c` and wired up by the daemon. PoTB arithmetic
and native schemas exist, but correlation-group detection, trusted ASN
observations, and the epoch producer pipeline remain research work, not
invented core logic.

## 2.8. Networking and P2P

**Status: validator transport implemented.** `src/net` provides a TCP
transport with framed messages, a genesis-bound handshake, transaction
and consensus gossip with hop-based deduplication, keepalive, mutual
connection dedup, and paginated finalized catch-up from durable storage.
The daemon (`src/daemon`) drives it single-threaded alongside RPC. Peer
discovery, transport encryption, rate-limiting policy, and large-committee
topology remain open.

### Requirements already imposed by decisions already made

**A 400 ms – 1 s block time sets the latency budget.** Within one block,
a proposal must reach the committee, and votes must return and reach
quorum. That's a minimum of two network round-trips among ~100 nodes
within a few hundred milliseconds. This is the hardest requirement in the
project, and it constrains everything else.

**Partial committee rotation exists to reduce network load.** Full
rotation of a 100-node committee on every block was rejected as
unrealistic network load; ~10% per block was chosen instead. So **the
network layer is already a first-class input to the consensus design** —
if later measurements show a different rotation fraction is needed, that
is a consensus parameter change (`rotation_fraction`).

**NDM requires observing peer ASNs.** `al_potb_ndm` consumes `asn` and
`asn_peer_count` per node. Something must observe and agree on these
values, and they're consensus-visible inputs — so ASN observation can't
be a purely local heuristic. **How the network reaches agreement on
another node's ASN is unspecified and a genuine open problem**, not a
detail. A node self-reporting its ASN can trivially lie; a node's peers
observing it disagree with each other. A mitigating fact: NDM is a
documented soft multiplier, bypassable with residential proxies — this is
an auxiliary layer, not a defense, and it shouldn't become load-bearing.

**External challenges are a network protocol.** TGW's reinforcement
depends on protocol-forced external challenges. Issuing a challenge to a
specific node and recording whether it responded is network work with
consensus consequences.

**Attestation and vote messages have reserved hash tags.** `AL_TAG_VOTE`
and `AL_TAG_ATTESTATION` already exist in `hash.h`, fixing their hashing
discipline before the wire format is finalized.

### Open questions

**Transport.** TCP, QUIC, or something custom over UDP. QUIC gives
multiplexing without head-of-line blocking and built-in encryption at the
cost of a dependency — and the core's stated rule (C with no hidden
allocations) rules out most QUIC libraries.

**Peer discovery.** A Kademlia-like DHT (Ethereum's discv5), a
gossip-based membership protocol, or seed nodes plus exchange. Discovery
is an attack surface: eclipse attacks work by controlling a node's peer
set, and an eclipsed validator can be forced to miss votes — which costs
it points under PoTB. **The slashing design therefore has a networking
prerequisite**: `al_potb_slash` forgives misses within 2× the network
median, which limits damage from a partial, but not from a targeted,
eclipse.

**Committee communication topology.** Broadcasting votes among 100 nodes
is O(n²) messages per round. Options: a full mesh (simple, 10,000
messages), gossip (fewer messages, higher latency — which the 400 ms
budget may not allow), or aggregation with signature aggregation at
intermediate nodes (fewest messages, requires an aggregatable signature
scheme — Ed25519 isn't aggregatable in the BLS sense). **This decision is
tied to the signature scheme decision.** If vote aggregation turns out to
be necessary for the latency budget, the signature scheme can't be plain
Ed25519, and that changes the crypto migration checklist.

**Transaction gossip and DoS resilience.** Rate limiting, peer scoring,
and what a node does under overload. Separate from consensus slashing:
dropping a flooding peer is a local decision, not a protocol penalty.

**Block propagation.** Full blocks, or compact blocks announcing
transaction identifiers a peer likely already has. The latter matters far
more with a short block time.

**Synchronization.** How a new node catches up with the network: full
replay from genesis, state snapshots, or checkpoints.

**Peer identity and wire encryption.** Whether the transport authenticates
peers by their consensus key or a separate network key. A separate key is
better hygiene (the network key is exposed to every peer, the consensus
key signs blocks), but adds a binding to manage.

### What "done" means for this layer

A chosen transport with a measured round-trip figure for a 100-node
committee, a message-format table, a stated vote topology with message
count arithmetic, and a defensible answer to the ASN question. Until the
latency budget is *measured*, not assumed, the block-time range in
section 0.4 is an estimate.