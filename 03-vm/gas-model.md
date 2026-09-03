# Resource and fee model v1

**Status: implemented; production numbers require benchmark calibration.**

Astrolune meters four independent unsigned 64-bit dimensions:

| Dimension | Definition |
|---|---|
| compute | sum of genesis opcode and host costs along the executed trace |
| memory | peak touched 4 KiB pages, including synchronous child calls |
| storage | canonical key plus old/new value bytes read or written |
| bandwidth | exact canonical signed transaction size |

Every transaction carries a limit vector and a maximum base-price vector.
Every block carries actual aggregate usage and the price vector used for that
block. Arithmetic overflow is an execution/consensus error; counters never
saturate.

## Base price

Each resource price evolves independently from parent usage. The genesis target
is normally 50% of the block limit. For price P, usage U and target T:

    delta = floor(P * abs(U - T) / T / 8)

Above target, the next price is P + max(delta, 1). Below target it is
max(P - delta, 1). At target it is unchanged. Thus a full block against a 50%
target moves at most 12.5%, and the minimum price is one.

The actual base fee is the checked dot product of actual usage and current
prices. It is removed from the sender and burned. The transaction tip is paid
in full once execution is included and is split into PoTB buckets 60/25/15;
integer rounding remainder goes to the flat bucket.

## Failure semantics

Encoding, size/type, chain/expiry, nonce, resource/balance bounds and signature
are checked in that order. A pre-validation failure changes nothing.

After execution starts:

- success commits writes/value, increments nonce, burns actual base fee and
  pays the full tip;
- REVERT or trap restores all execution writes and value transfer, but still
  increments nonce, burns actually used base fee and pays the full tip;
- an unused resource allowance is never charged;
- storage deposits are state collateral, not a fee.

## Storage deposit

Contract storage growth locks deposit_per_byte from that contract's balance.
Shrinking or deleting a value returns the exact corresponding deposit to the
same contract. The reserved PoTB system namespace is maintained by native
consensus transitions and is exempt from ordinary contract deposits.

Production block limits and prices are intentionally not hard-coded here. They
are genesis values and must be selected from benchmark data for the minimum
supported validator hardware.
