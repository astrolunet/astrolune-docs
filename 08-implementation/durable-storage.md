# Durable node storage v1

**Status: implemented.** This is node-local storage, not a consensus encoding.
It implements `al_state_store` without changing the state or block ABI.

## Files and ownership

A data directory is bound to one canonical genesis hash by `manifest.bin` and
locked against concurrent writers through `LOCK`. The remaining files are
append-only logs:

| File | Contents |
|---|---|
| `state-nodes.log` | fixed-size content-addressed SMT branch and leaf records |
| `state-values.log` | variable-size account, storage and contract-code values |
| `chain.log` | canonical block bytes in contiguous height order |
| `finality.log` | checksum-protected finality certificates paired by height |
| `signing.log` | validator proposal/vote decisions synced before signing |

Every record carries a format version and SHA-256 checksum. Node records also
recompute their consensus SMT hash while opening. In-memory open-addressed hash
indexes map state hashes to 64-bit file offsets; values are loaded lazily and
cached so the borrowed `al_state_store.value_get` lifetime remains stable until
the store closes.

The formats are deliberately node-local. They may be migrated without a hard
fork because canonical state roots and block bytes remain the authority.

## Commit protocol

State writes occur while the consensus executor stages a block. They are safe
to append early because content-addressed objects are immutable and unreachable
objects have no logical effect. Publishing a block uses this order:

1. verify the encoded header matches the current state height and root;
2. append and sync the matching finality certificate;
3. flush and sync both state logs;
4. append the checksummed canonical block record;
5. flush and sync the chain log;
6. publish the new durable head in memory.

Thus a durable chain record never points at state objects that were still only
in userspace buffers. A failure during commit requires closing and reopening
the runtime; the last complete chain record remains authoritative.

## Recovery

Open scans each log, verifies record boundaries and checksums, rebuilds the
indexes and truncates an incomplete or torn final record. Corruption before the
tail is reported as `AL_ERR_STATE_CORRUPT` rather than guessed around. Chain
records must start at height zero, remain contiguous, match their encoded block
headers and link to the prior header hash. In validator mode, every chain
record must have a matching finality record with the same height and block hash.

The recovered head supplies `(height, state_root)` to `al_state_open`, and
`al_node_open` verifies the reopened state against that head. The mempool is
local volatile policy and intentionally starts empty after a restart.

The daemon opens `signing.log` for a local committee member and binds every
record to chain ID and signer public key. A complete checksum failure blocks
startup. An incomplete tail is removed because the daemon never creates or
publishes the corresponding signature until the record has been synced. An
existing slot may be replayed only with the exact same signing digest.

## CLI

    alnode init-node <genesis> <data-dir>
    alnode import-blocks <genesis> <data-dir> <block ...>
    alnode node-head <genesis> <data-dir>
    alnode produce-node-block <genesis> <data-dir> <output>

The existing `verify-chain` and `produce-block` commands remain stateless tools.

## Deferred work

Compaction/pruning, snapshot export/import, canonical-chain rewind and an
alternative high-throughput database backend remain separate milestones. The
append-only v1 format keeps unreachable state safely but does not reclaim it.
