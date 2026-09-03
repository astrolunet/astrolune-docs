# Component map

## Build graph

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

Astrolune::core is an interface target that exposes all seven core static
libraries. The split enforces dependency direction: VM has no state/crypto
dependency and state has no VM dependency. al_node is separate because mempool
and ingress behavior are local policy, not consensus ABI. al_net moves opaque
canonical bytes and never links consensus; the daemon is the only component
that sees node, net and rpc together.

| Target | Responsibility |
|---|---|
| al_base | status, portability, arena, bytes, fixed point, resource fees |
| al_crypto | hashes, Merkle, keys and selectable signature backend |
| al_potb | pure PoTB scoring, committee, epoch seed and rewards |
| al_state | immutable node store, two-level SMT and staged state |
| al_vm | ALVM decoder, CFG validator and interpreter |
| al_tx | envelope, host integration, fees, events and receipts |
| al_block | genesis v2 allocations, canonical body, atomic production and execution |
| al_node | mempool, block ingress, durable state objects and canonical chain log |
| al_net | sockets, framed wire protocol, peer manager with gossip and sync |
| al_rpc | JSON codec and JSON-RPC over HTTP/1.1 |
| al_daemon | event loop gluing storage, P2P and RPC; timed block production |
| alnode | CLI: keygen, genesis authoring, offline chain tools and `run` |

## Source tree

    include/astrolune/   installed C ABI
    src/base/            dependency-free foundation
    src/crypto/          hash and crypto backends
    src/consensus/       PoTB arithmetic
    src/state/           SMT and state staging
    src/vm/              ALVM implementation
    src/tx/              transactions and execution host
    src/chain/           genesis and block pipeline
    src/node/            local policy, block ingress and durable storage
    src/net/             P2P transport (sockets, wire, peers)
    src/rpc/             JSON-RPC server
    src/daemon/          node service event loop
    src/apps/alnode/     command-line entry point
    tools/               Trocto/Regol compiler (C++23) and trocto CLI
    abi/                 C/C++ boundary and manifest checks
    tests/c/             behavioral suites and determinism fixture
    fuzz/                libFuzzer/corpus-runner targets and seed corpus
    cmake/               standards, warnings, sanitizer and target helpers
    docs/                protocol and engineering specification

Peer discovery, transport encryption and the Trocto compiler remain future
components. Durable storage reaches consensus state only through the existing
`al_state_store` callbacks.

## Public headers

| Header | Owner |
|---|---|
| base.h, arena.h, bytes.h, fixed.h | al_base |
| hash.h, crypto.h | al_crypto |
| potb.h | al_potb |
| state.h | al_state |
| vm.h | al_vm |
| tx.h | al_tx |
| block.h | al_block |

Every public function carries AL_PUBLIC. The build parses these declarations,
compares them with boundary_symbols.cpp, compiles each header alone in C++23 and
checks all ABI layouts from both C23 and C++23.

## Verification map

| Area | Verification |
|---|---|
| base/crypto/PoTB | focused CTest suites and published vectors where possible |
| state | roots, order independence, proofs, deposits, staging and snapshots |
| VM | container/CFG, opcodes, hosts, traps, entrypoints and resources |
| transactions | canonical decode, signatures, validation order, fees and rollback |
| blocks | genesis/body round trips, roots, price transition and atomic mismatch |
| node | admission, nonce reservations, payload ownership, pruning and head rollback |
| decoders | four corpus runners; the same sources become libFuzzer targets |
| determinism | one canonical block digest compared across GCC, Clang and MSVC |
| contracts | compiled sources execute on the real VM with a mock host (test_lang) |
| ABI | layout, standalone headers, C-link symbols and AL_PUBLIC manifest |
