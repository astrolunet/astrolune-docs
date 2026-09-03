# Implementation status

Updated for the Astrolune v1 core baseline and the PoTB validator runtime.

## Implemented

| Area | Target | State |
|---|---|---|
| portability, arenas, canonical bytes, fixed arithmetic, resources | al_base | implemented and tested |
| SHA-256, HMAC/HKDF, Merkle, address derivation | al_crypto | implemented and tested |
| signatures | al_crypto | dev backend by default; optional libsodium Ed25519 with RFC 8032 tests |
| VRF and VDF | al_crypto | deterministic insecure dev backend |
| PoTB arithmetic, committee, seed and rewards | al_potb | implemented and tested |
| depth-256 account/storage SMT and staged transactions | al_state | implemented and tested |
| ALVM container, CFG validator, interpreter and host ABI | al_vm | implemented and tested |
| typed transactions, fees, receipts, events and PoTB schemas | al_tx | implemented and tested |
| genesis v2 with prefunded allocations, blocks and atomic execution | al_block | implemented and tested |
| local head, bounded mempool, block production and canonical ingress | al_node | implemented |
| append-only state/chain/finality storage, checksums, crash-tail recovery, genesis materialization | al_node | implemented |
| TCP transport, framed wire protocol, peer manager with gossip, dedup and finalized paged range sync | al_net | implemented; tested over loopback |
| JSON codec and JSON-RPC server (HTTP/1.1) | al_rpc | implemented; tested end to end |
| single-threaded node daemon: storage + P2P + RPC + timed block production | al_daemon | implemented |
| `alnode` CLI: keygen, genesis authoring, offline chain tools, `run` daemon | alnode | implemented |
| two-validator restart/contract/quorum scenario | scripts/smoke.ps1 | scripted; runs in CI (Windows) |
| four-validator 3/4 and 2/4 quorum scenario | scripts/consensus-smoke.ps1 | scripted; runs in CI (Windows) |
| C/C++ ABI layout, link and manifest checks | al_abi_boundary | enforced during build |
| GCC/Clang/MSVC strict CI and ASan/UBSan jobs | GitHub Actions | configured |
| decoder fuzz/corpus targets | fuzz/ | transaction, ALVM, block/genesis, SMT proof |
| contract-language toolchain: Trocto v0.2 + Regol assembler + CLI | trocto | implemented; end-to-end tested on the real VM |
| cross-toolchain deterministic block digest | determinism_fixture | configured in CI |

## Validator runtime

- repeatable validator public-key configuration and deterministic PoTB committee
  construction;
- signed proposals with parent, block and committee commitments;
- PREVOTE/PRECOMMIT vote sets with unique-membership and quorum validation;
- finality certificates persisted beside each canonical block;
- finalized block+certificate envelopes for gossip and range catch-up;
- restart recovery that checks one finality record for every committed block;
- synchronized anti-double-sign journal for proposals, prevotes and precommits;
- isolated proposal execution: RPC head/state and mempool remain canonical until
  finality;
- timed round changes with deterministic proposer rotation.

The suite currently contains 24 CTest entries: 20 behavioral/header suites and
four corpus runners. LibFuzzer builds use the fuzz preset and Clang
CI performs short smoke campaigns.

## Consensus behavior now fixed

- account balance/nonce, immutable code and per-contract storage roots;
- content-addressed two-level sparse Merkle commitments and compressed proofs;
- synchronous calls, depth limit 64 and active-address no-reentry rule;
- compute, memory, storage and bandwidth accounting;
- refundable storage collateral;
- versioned typed transactions with chain ID, nonce and expiry;
- burned vector base fee and full PoTB 60/25/15 tip;
- failed-execution charging with execution-state rollback;
- state, transaction, receipt, usage and price commitments in block headers;
- genesis-owned resource/PoTB parameters and VM schedule;
- genesis v2 allocation table: canonical, sorted, replayable by every node.

## Network behavior now fixed

- user-facing addresses are Bech32 with the `al1...` prefix (BIP-173);
  raw 32-byte hex remains accepted on every input surface;
- contract lifecycle over RPC and CLI: DEPLOY transactions from compiled
  Trocto containers, CALL dry-run reads between blocks;
- TCP P2P with framed messages: HELLO bound to the genesis hash, ping/pong
  keepalive, transaction/block gossip with per-hop dedup rings;
- automatic range sync: a peer behind the taller side pulls finalized entries at
  handshake; full pages trigger the next request;
- JSON-RPC over HTTP/1.1: `get_info`, `get_account`, `send_raw_transaction`,
  `transfer`, `get_mempool`, `get_peers`, `stop`;
- daemon event loop: timed block production when the mempool is non-empty,
  bootstrap re-dialing, bounded outboxes, idle/handshake timeouts.

## Deliberate non-production boundaries

1. The default signatures and all current VRF/VDF implementations are
   development-only. A sodium build provides real Ed25519 signatures but still
   reports `al_crypto_is_secure() == AL_FALSE` until VRF/VDF are resolved.
2. The durable node backend is append-only. It implements crash-tail recovery
   and genesis binding, but pruning, snapshot import/export and chain rewind
   are not implemented yet.
3. PoTB native transactions commit schemas/evidence to system state, but no code
   invents correlation groups, trusted ASN observations or unresolved epoch
   policy.
4. Default limits/prices are development values. Production genesis values need
   benchmark calibration on minimum validator hardware.
5. Peer discovery and transport encryption remain outside the implemented
   boundary; the validator runtime itself uses committee-backed
   proposal/voting/finality and finalized catch-up.

## Verification commands

On Windows use:

    scripts\build.bat dev test
    scripts\build.bat ci test
    set ASAN_OPTIONS=allocator_may_return_null=1
    scripts\build.bat asan test
    powershell -ExecutionPolicy Bypass -File scripts\smoke.ps1
    powershell -ExecutionPolicy Bypass -File scripts\consensus-smoke.ps1

On GCC/Clang-capable hosts use the equivalent CMake workflow presets. The fuzz
preset requires Clang and links libFuzzer with ASan/UBSan.
