# Core API reference

The installed C23/C++23-compatible ABI is include/astrolune. Headers are the
authority. Every public function is marked AL_PUBLIC and is checked against the
221-entry link manifest during every normal build.

## Shared rules

- fixed-width types are used for every consensus field;
- failures are al_status values, never exceptions, errno or globals;
- fallible return values use out-parameters and AL_NODISCARD;
- borrowed views never transfer ownership;
- variable-lifetime results use caller-owned arenas;
- canonical integers are little-endian and lengths are minimal varints;
- no floating point or ambient time enters consensus;
- checked arithmetic is used for resources, balances and fees.

## Header map

| Header | Public contract |
|---|---|
| base.h | scalar/domain types, statuses, resource vectors, fee transition |
| arena.h | block allocator, scoped marks and peak diagnostics |
| bytes.h | borrowed views, canonical reader/writer and hex |
| fixed.h | deterministic Q32.32 arithmetic |
| hash.h | SHA-256, HMAC/HKDF, domain tags and ordered Merkle trees |
| crypto.h | addresses and the explicitly insecure dev signature/VRF/VDF backend |
| potb.h | PoTB params, records, scoring, committees, seed and rewards |
| state.h | content-addressed SMT state, staging, snapshots and proofs |
| vm.h | ALVM container, validator, interpreter, hosts and resource schedule |
| tx.h | typed v1 envelope, events, receipts and transaction execution |
| block.h | genesis, block body/header, roots and atomic block execution |

## Ownership map

| Value | Lifetime |
|---|---|
| decoded transaction payload | borrows encoded transaction bytes |
| decoded block transactions | caller-supplied array; payloads borrow block bytes |
| decoded event/receipt data | borrows encoded bytes; receipt event array uses arena |
| VM program descriptors | arena |
| VM return data | execution arena |
| receipt events/data | execution arena |
| state storage read | copy in caller arena |
| state code read | borrowed immutable node-store value |
| SMT siblings | caller-supplied array |

## State

al_state contains only implementation context, committed height and root.
al_state_txn contains a staged root and resource counter. Begin/get/upsert/
remove/transfer/deploy/storage operations never publish a root until commit.
Rollback makes staged content unreachable. Snapshot restore is the reorg and
block-atomicity primitive.

The ordinary storage API rejects the reserved PoTB address. Native consensus
code uses the separately named system-storage functions.

## Cryptography

Signature implementation is selected at configure time. The default backend is
deterministic and forgeable; `ASTROLUNE_CRYPTO_BACKEND=sodium` provides RFC 8032
Ed25519 without changing public key, secret key or signature widths. Backend
kind describes the signature implementation only. Callers must use
`al_crypto_is_secure()` as the deployment gate, which remains false while the
VRF/VDF implementations are development-only.

## VM

al_vm_validate consumes scratch arena space only for the duration of validation.
al_vm_execute receives immutable container/calldata, genesis-derived config,
execution context, host callbacks and a result arena. Resource limits are four
dimensional. The schedule pointer may be null only to select development
defaults.

## Transactions

al_tx_decode borrows payload bytes. al_tx_apply distinguishes two classes:

- a returned error is pre-validation/inclusion failure and changes no state;
- AL_OK with receipt.status != AL_OK is an included failed execution, charged
  according to the v1 fee rules.

Callers must preserve the receipt arena until receipt/event/return data is no
longer needed.

## Blocks

al_genesis_validate must run before accepting a chain configuration.
al_block_decode borrows transaction payloads from the encoded body.
al_block_produce derives parent linkage, prices and every execution commitment
while applying a local proposal atomically; callers supply only proposal-owned
header fields and the ordered body.
al_block_execute takes the parent header (null only at height zero), genesis,
state, receipt storage and arena. It validates all commitments and restores the
entry state snapshot on any failure.

## ABI enforcement

cpp/abi performs four independent checks:

1. every installed header compiles alone and together as C++23;
2. abi_contract.h pins layout in both C23 and C++23;
3. boundary_symbols.cpp links every public C symbol from C++;
4. al_abi_manifest_check parses AL_PUBLIC declarations and rejects missing or
   duplicate manifest entries without external parser dependencies.
