# Roadmap

## Completed baseline

### Step 1: deterministic foundation

- C23/C++23 build system, strict warnings and sanitizers;
- arenas, canonical byte codecs and deterministic fixed-point arithmetic;
- SHA-256/HMAC/HKDF/Merkle and explicit development crypto;
- PoTB scoring, committee selection, epoch seed and rewards.

### Step 2: Astrolune v1 core

- two-level depth-256 content-addressed sparse Merkle state;
- immutable contract code, staged roots, snapshots and storage deposits;
- canonical ALVM v1 container, CFG validation and 64-bit interpreter;
- synchronous host/contract calls, rollback and no-reentry enforcement;
- typed/versioned transactions, resource caps, expiry and receipts/events;
- vector base fee, burned base charge and PoTB 60/25/15 tip;
- canonical genesis/block body and atomic commitment-checking executor;
- PoTB native schemas and reserved system-state bridge;
- ABI manifest/layout enforcement, multi-toolchain CI, fuzz targets and a
  deterministic cross-toolchain block fixture.

## Next engineering milestones

1. Complete the production cryptography migration. The optional libsodium
   Ed25519 signature path and RFC 8032 vectors are implemented; production
   packaging, ECVRF and the VDF-or-no-VDF decision remain.
2. Extend durable storage. The content-addressed state backend, canonical block
   log and crash-tail recovery are implemented; pruning, snapshot import/export
   and canonical-chain rewind remain.
3. Benchmark opcode, host, storage and block execution on the minimum validator
   hardware, then publish production genesis limits/prices.
4. Harden the network layer. The transport now carries handshakes bound to the
   genesis hash, transaction/block gossip with dedup, and range-based catch-up
   sync; peer discovery, transport encryption, rate-limiting policy and
   committee-authorised proposal/finality remain.
5. Complete the evidence validation behind PoTB-native operations without
   guessing unresolved correlation/ASN policy.
6. Build Trocto/Regol tooling and contract SDK against the frozen ALVM v1
   container. The v0.2 compiler is in place (both tiers, constructors,
   expanded maps, string literals, assert, import validation); function
   linking for imports, generics and the module system remain.
7. Add applications and operational tooling only after the node and production
   crypto boundaries are complete.

## Continuous gates

Every protocol change must keep:

- standalone public-header C++ compilation;
- C/C++ layout contract and AL_PUBLIC manifest equality;
- GCC, Clang and MSVC strict builds;
- Clang ASan/UBSan and MSVC ASan;
- all behavioral and corpus CTests;
- short decoder fuzz smoke runs;
- one identical determinism digest across all three toolchains.

Long-running fuzz campaigns and performance calibration are release activities,
not substitutes for the short gates above.
