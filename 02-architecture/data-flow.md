# Data flow

**Status: current for the v1 core and validator runtime.** Canonical validation,
execution, bounded mempool admission, signed finality, gossip and durable local
head tracking are connected through the running daemon.

## Transaction path

```text
canonical bytes
  -> shape/size/type decode
  -> chain and expiry checks
  -> exact account nonce
  -> resource caps and worst-case balance bound
  -> signature verification
  -> staged execution
  -> actual base-fee burn and full tip distribution
  -> receipt and state commit
```

The order is consensus-visible. Pre-validation failures change nothing.
Execution success, REVERT and trap are included outcomes: all consume the nonce,
actual resource fee and full tip. REVERT/trap restore staged writes, events and
value transfer before charging.

Transfer and PoTB-native bodies execute directly. Deploy validates and stores an
immutable ALVM container. Calls enter the requested function with calldata and a
transaction-backed host interface. Nested calls are synchronous; a child REVERT
returns status/data after restoring its subcall root, while a trap aborts the
transaction. The active-address stack rejects reentrancy.

## Block path

```text
canonical block
  -> header, parent and base-price transition checks
  -> transaction-root verification
  -> transactions applied in committed order
  -> block resource-limit check
  -> state, receipt and usage commitment checks
  -> atomic state commit
```

Any mismatch restores the entry snapshot. Genesis supplies chain ID, initial
root, limits, resource targets and prices, VM cost schedule, storage collateral
rate and PoTB parameters. A cross-toolchain fixture executes the same block and
publishes one canonical digest in CI.

## State path

The top depth-256 sparse Merkle tree maps tagged address hashes to account
leaves. Each contract owns a second depth-256 tree for tagged storage keys.
Staged updates replace only the working root; commit publishes it and rollback
discards reachability of newly written content-addressed nodes. Storage growth
locks contract balance at the genesis rate and deletion refunds it.

PoTB-native transactions write their complete canonical body below the reserved
system account. Bytecode cannot address this namespace.

## Node ingress path

```text
canonical transaction bytes
  -> canonical decode and signature verification
  -> chain, next-height price and expiry policy
  -> contiguous per-sender nonce and cumulative balance reservation
  -> owned bounded mempool storage

canonical block bytes
  -> decode into caller-sized scratch storage
  -> atomic core block execution
  -> local head publication
  -> stale/expired mempool pruning and byte-buffer compaction

local proposal
  -> FIFO selection bounded by declared transaction resources
  -> parent, height, base-price and transaction-root derivation
  -> isolated validation/execution pass
  -> state, receipt and resource commitment derivation
  -> canonical encoding and immediate checkpoint restore
  -> sync signing decision, sign and gossip proposal
  -> PREVOTE/PRECOMMIT quorum
  -> repeat execution only after finality

durable block commit
  -> append and sync finality certificate
  -> sync content-addressed node/value logs
  -> append checksummed canonical block record
  -> sync chain log and publish durable head
```

On restart, the node verifies the genesis-bound manifest, rebuilds hash/height
indexes, removes a torn final record and opens state at the last complete block
root. A validator also restores the signing journal and refuses a conflicting
proposal or vote for a previously signed height/round/phase. Mempool policy is
deliberately local and does not affect block validity.
There
is no replacement policy yet: duplicate hashes and conflicting sender/nonce
pairs are rejected. Receipt/event memory remains valid until the next block
attempt resets the node's execution arena.

## Deferred paths

Peer discovery, transport encryption, rate-limiting policy and fee-based
mempool ordering remain node/network work. The P2P transport now carries
handshakes, gossip and finalized range sync (see `src/net`). Committee-authorised
proposals, voting and finality are implemented in `src/consensus/finality.c`
and connected by the daemon. The PoTB arithmetic and native
schemas exist, but correlation-group discovery, trusted ASN observations and
the epoch producer pipeline remain research work rather than invented core
logic.
