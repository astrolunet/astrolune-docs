# State model v1

**Status: implemented with in-memory and durable node backends. The disk
backend preserves this interface and remains outside consensus.**

## Accounts

The top-level state tree maps tagged 256-bit address hashes to fixed account
values:

| Field | Meaning |
|---|---|
| address | full 32-byte account address |
| balance | spendable token amount |
| nonce | next sequential transaction nonce |
| code_hash | zero for users; SHA-256 of immutable ALVM container for contracts |
| storage_root | root of this contract's independent storage SMT |
| storage_bytes | current stored value bytes |
| storage_deposit | collateral locked for those bytes |

Contract code is immutable and content-addressed. Upgrade means deploying a new
address and explicitly migrating state; no opcode or state API replaces code.

## Two-level sparse Merkle tree

Both levels are compressed depth-256 binary sparse Merkle trees. Account keys
use AL_TAG_ACCOUNT_KEY(address). Contract storage keys use
AL_TAG_STORAGE_KEY(key). Leaves and branches use distinct SMT domain tags, and
account/storage values use distinct value tags. Empty hashes are deterministic
at every depth.

An SMT proof contains the full key, existence bit, value hash, a 256-bit sibling
bitmap and only non-default siblings in root-to-leaf order. The decoder rejects
non-canonical counts, impossible absence values, malformed bitmaps, truncation
and trailing bytes. The same format proves membership and absence.

## Store contract

The consensus layer receives an al_state_store with caller-owned context and
four callbacks: immutable node get/put and immutable value get/put. A put under
an existing hash must contain identical content. The core owns no database and
does not retain caller buffers.

al_state_memory_store is a deterministic linear backend for tests, fuzzing and
small simulations. The node's append-only backend indexes the same immutable
objects on disk and commits canonical block bytes only after state files are
synced. Unreachable content-addressed nodes may remain physically in either
backend; only nodes reachable from a committed root are logical state.

## Staging and snapshots

al_state_txn_begin copies the committed root. Reads and writes operate against
that staged root. Commit publishes it atomically; rollback discards it.
Snapshots are (height, root) pairs and restore reorg state in constant time.

A failed contract call restores its subcall root. A failed transaction restores
its execution root before committing only nonce and fees. A failed block
restores the block-entry snapshot.

## Storage and system state

Ordinary storage keys are 1..256 bytes and values are at most 64 KiB. Net growth
locks genesis deposit_per_byte from the contract; shrinkage refunds it.
Storage I/O bytes are added to the staged transaction resource vector.

PoTB records live in the storage tree of a deterministic reserved system
address. Ordinary storage methods and bytecode hosts reject that address.
Only the explicitly named native system-storage bridge can mutate it.
