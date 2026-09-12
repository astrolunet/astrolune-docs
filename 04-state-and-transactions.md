# 4. State, Transactions, and Blocks

## 4.1. State model v1

**Status: implemented, with in-memory and durable node backends. The
disk backend preserves this interface and stays outside consensus.**

### Accounts

The top-level state tree maps tagged 256-bit address hashes to fixed
account values:

| Field | Meaning |
|---|---|
| address | full 32-byte account address |
| balance | spendable token amount |
| nonce | next sequential transaction nonce |
| code_hash | zero for users; SHA-256 of the immutable ALVM container for contracts |
| storage_root | root of this contract's independent SMT storage |
| storage_bytes | current volume of stored value bytes |
| storage_deposit | collateral locked for those bytes |

Contract code is immutable and content-addressed. Upgrading means
deploying to a new address and explicit state migration; neither a VM
opcode nor the state API replaces code in place.

### Two-tier sparse Merkle tree

Both tiers are compressed depth-256 binary sparse Merkle trees. Account
keys use `AL_TAG_ACCOUNT_KEY(address)`. Contract storage keys use
`AL_TAG_STORAGE_KEY(key)`. Leaves and branches use distinct SMT domain
tags; account and storage values use distinct value tags. Empty hashes
are deterministic at every depth.

An SMT proof contains the full key, an existence bit, the value hash, a
256-bit neighbor bitmap, and only the non-default neighbors, in
root-to-leaf order. The decoder rejects non-canonical counts, impossible
absence values, malformed bitmaps, truncation, and trailing bytes. One
format proves both membership and absence.

### State store contract

The consensus layer receives an `al_state_store` with caller-supplied
context and four callbacks: immutable node get/put and immutable value
get/put. A write under an existing hash must contain identical content.
The core does not own a database and does not hold caller buffers.

`al_state_memory_store` is a deterministic linear backend for tests,
fuzzing, and small simulations. The node backend append-only indexes the
same immutable objects on disk and commits canonical block bytes only
after state files are synced. Unreachable content-addressed nodes may
physically remain in either backend; only nodes reachable from the
committed root are logical state.

### Staging and snapshots

`al_state_txn_begin` copies the committed root. Reads and writes operate
against this staged root. Commit publishes it atomically; rollback
discards it. Snapshots are (height, root) pairs and restore reorg state
in constant time.

A failed contract call restores its subroot. A failed transaction
restores its execution root before committing only the nonce and fees. A
failed block restores the snapshot from block entry.

### Storage and system state

Ordinary storage keys are 1..256 bytes, values up to 64 KiB. Net growth
locks `deposit_per_byte` from genesis at the contract; shrinkage returns
the deposit. Storage I/O bytes are added to the transaction's staged
resource vector.

PoTB records live in the storage tree of a deterministic reserved system
address. Ordinary storage methods and bytecode hosts reject this address.
Only an explicitly named native system-storage bridge may modify it.

## 4.2. Transactions, receipts, and blocks v1

**Status: implemented.** All integer fields are little-endian; all
lengths are minimal unsigned varints; trailing bytes are invalid.

### Signed envelope

| Field | Encoding |
|---|---|
| version | u16, exactly 1 |
| chain_id | u32 |
| expiry_height | u64 |
| sender | 32-byte public key |
| nonce | u64, the account's exact next nonce |
| resource_limit | four u64 values |
| max_base_price | four u64 values |
| tip | u64 |
| type | u8 |
| body | type-dependent |
| signature | 64 bytes |

The four resource values are compute, memory, storage, and bandwidth. The
maximum canonical transaction size is 1 MiB. There is no batching in v1.

| Type | Value | Body |
|---|---:|---|
| transfer | 0 | recipient:address, amount:u64 |
| deploy | 1 | value:u64, container:varbytes |
| call | 2 | contract:address, value:u64, entrypoint:u32, calldata:varbytes |
| PoTB native | 3 | operation:u8, target:pubkey, amount:u64, data:varbytes |

PoTB operations 0..9 are registration, attestation, challenge, challenge
response, violation proof, bond deposit, bond withdrawal, seed commit,
seed reveal, and committee vote. This layer commits their canonical proof
to the reserved system namespace. The stored value carries the full
canonical native body: operation, target, amount, and proof. Correlation
grouping, ASN reconciliation, and epoch research are deliberately not
invented by the transaction implementation.

### Hashes and signatures

The signable digest is tagged `AL_TAG_TX_SIGNING` and covers every field
through the body, excluding the signature. The transaction identifier is
tagged `AL_TAG_TX` and covers the same canonical fields plus the
signature. These domains are deliberately distinct.

### Validation and charging

Validation order is fixed:
1. version, encoding, size, and type/body shape;
2. chain ID and expiry height;
3. exact nonce;
4. bandwidth, resource price ceilings, and worst-case balance debit;
5. signature.

Failures at this stage charge nothing. Once execution begins, `al_tx_apply`
returns `AL_OK` for an included transaction and records in the receipt
whether execution succeeded, REVERTed, or trapped. Included failures
consume the nonce, the actual base fee, and the full tip, restoring
writes.

### Events and receipts

An event carries the emitting contract, a tagged topic hash, and
borrowed data. Event and receipt encoders have matching strict decoders;
variable data remains a borrowed view of the canonical input, and a
receipt's event array uses the caller's arena. A receipt commits:
- transaction hash and execution status;
- actual four-dimensional resource usage;
- burned base fee and paid tip;
- deployed contract address, when applicable;
- return data;
- ordered events, whose hashes fold into the receipt hash.

Receipt and event encodings have independent domain tags. A block commits
an ordered Merkle root of receipts.

### Genesis

The canonical genesis format fixes the version, chain ID, initial state
root, block limits, 50% targets, initial base prices, storage deposit
rate, every opcode/host cost, VM limits, and the full set of PoTB
parameters. Genesis validation rejects zero prices/limits/costs and
inconsistent PoTB parameters.

### Blocks

A fixed 338-byte header encodes the version, chain ID, height, protocol
day, parent/state/transaction/receipt roots, actual resource usage,
current base prices, the proposer key, and three PoTB tip-bucket
addresses. The canonical body is the header, a transaction count, then
length-prefixed canonical transactions.

Block execution verifies the parent link, price transition, and
transaction root before execution. It applies transactions in body order,
enforcing the genesis block limit, then verifies the state root, receipt
root, and exact aggregate usage. Any mismatch restores the snapshot from
block entry. Transaction ordering policy is a proposer/mempool concern;
validation executes the committed order.
