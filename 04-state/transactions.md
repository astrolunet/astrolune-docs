# Transactions, receipts and blocks v1

**Status: implemented.** All integer fields are little-endian; all lengths are
minimal unsigned varints; trailing bytes are invalid.

## Signed envelope

| Field | Encoding |
|---|---|
| version | u16, exactly 1 |
| chain_id | u32 |
| expiry_height | u64 |
| sender | 32-byte public key |
| nonce | u64, exact next account nonce |
| resource_limit | four u64 values |
| max_base_price | four u64 values |
| tip | u64 |
| type | u8 |
| body | type-specific |
| signature | 64 bytes |

The four resource values are compute, memory, storage and bandwidth. Maximum
canonical transaction size is 1 MiB. Batching does not exist in v1.

| Type | Value | Body |
|---|---:|---|
| transfer | 0 | recipient:address, amount:u64 |
| deploy | 1 | value:u64, container:varbytes |
| call | 2 | contract:address, value:u64, entrypoint:u32, calldata:varbytes |
| PoTB native | 3 | operation:u8, target:pubkey, amount:u64, data:varbytes |

PoTB operations 0..9 are registration, attestation, challenge, challenge
response, offence evidence, bond deposit, bond withdrawal, seed commit, seed
reveal and committee vote. This layer commits their canonical evidence to the
reserved system namespace. The stored value contains the complete canonical
native body: operation, target, amount and evidence. Correlation grouping, ASN agreement and epoch
research are intentionally not invented by the transaction implementation.

## Hashes and signatures

The signing digest is tagged with AL_TAG_TX_SIGNING and covers every field
through the body, excluding the signature. The transaction identifier is tagged
with AL_TAG_TX and covers the same canonical fields plus the signature. These
domains are deliberately different.

## Validation and charging

Validation order is fixed:

1. version, encoding, size and type/body shape;
2. chain ID and expiry height;
3. exact nonce;
4. bandwidth, resource-price caps and worst-case balance debit;
5. signature.

Failures in this phase charge nothing. Once execution begins, al_tx_apply
returns AL_OK for an included transaction and records execution success,
REVERT or trap in the receipt. Included failures consume nonce, actual base fee
and full tip while restoring writes and value transfer.

## Events and receipts

An event contains emitting contract, tagged topic hash and borrowed data. Event
and receipt encoders have matching strict decoders; variable data remains a
borrowed view of the canonical input and receipt event arrays use a caller arena.
A receipt commits:

- transaction hash and execution status;
- actual four-dimensional resource use;
- burned base fee and paid tip;
- deployed contract address when applicable;
- return data;
- ordered events, whose hashes are folded into the receipt hash.

Receipt and event encodings have independent domain tags. A block commits the
ordered receipt Merkle root.

## Genesis

The canonical genesis format fixes version, chain ID, initial state root, block
limits, 50% targets, initial base prices, storage deposit rate, all opcode/host
costs, VM limits and the complete PoTB parameter set. Genesis validation rejects
zero prices/limits/costs and inconsistent PoTB parameters.

## Blocks

The fixed 338-byte header encodes version, chain ID, height, protocol day,
parent/state/transaction/receipt roots, actual resource usage, current base
prices, proposer key and the three PoTB tip-bucket addresses. The canonical body
is the header, transaction count, then length-prefixed canonical transactions.

Block execution verifies parent linkage, the price transition and transaction
root before execution. It applies transactions in body order, enforcing the
genesis block limit, then verifies state root, receipt root and exact aggregate
usage. Any mismatch restores the entry snapshot. Transaction ordering policy is
a proposer/mempool concern; validation executes the committed order.
