# Network architecture
### The C / C++ split

A companion to [../01-consensus/potb.md](../01-consensus/potb.md) — this document fixes the general engineering architecture of the project on top of the consensus model.

---

## 1. The principle behind the C / C++ split

**The rule:** if code runs on every node on every block (hot path, millions of calls) it is C. If code runs rarely, locally on a developer's machine, or in a place that is not time-critical, it may be C++.

### In plain C

| Component | Why C specifically |
|---|---|
| Consensus core (PoTB: VRF, BFT voting, TBS/TGW scoring, committee selection) | The most frequent and most time-critical code in the system — it runs on every block on every validating node |
| Networking / P2P layer (gossip, transport) | Constant, high-frequency traffic; minimal overhead matters |
| Cryptography (signatures, hashes, VDF) | Hot path — a signature check on every inbound transaction; a thin wrapper without C++ overhead over proven primitives (libsodium or equivalent) |
| Storage engine (disk I/O, chain state) | Direct control of memory and I/O, with predictable performance and no hidden allocations |
| VM (smart contract execution) | The hottest path of all — it runs on every contract call on every node |

### In C++

| Component | Why C++ is acceptable |
|---|---|
| CLI / SDK for developers | Not on the hot path; expressiveness and development speed matter more |
| RPC / API layer (requests from wallets and dApps) | Orders of magnitude rarer than internal consensus traffic |
| Indexer / explorer backend | An asynchronous, non-time-critical job |
| Contract language compiler | Runs locally on a developer's machine, not on a node in consensus — the rich abstractions of C++ for AST work and parsing are justified |

---

## 2. How the two halves meet

They meet at exactly one place: the public C ABI in `include/astrolune/`, which is compiled by both languages. That is a real constraint on what those headers may contain, and it is spelled out in [c-cpp-boundary.md](c-cpp-boundary.md).

The build enforces the rule rather than relying on convention:

- `astrolune_add_core_library()` sets `LINKER_LANGUAGE C` and the C standard, so a stray `.cpp` file in `core/` fails the build instead of being discovered months later on an embedded target.
- `astrolune_add_tool_library()` is the only helper that adds `cpp/include` to the include path, so a core library cannot even see the tooling headers.
- Dependency direction is expressed as link edges (`al_base` → `al_crypto` → `al_potb`) so a layering violation is a link error, not a code review finding.

Both helpers live in `cmake/AstroluneTargets.cmake`; the reasoning is in [../08-implementation/build-and-test.md](../08-implementation/build-and-test.md).

---

## 3. Current state of the tree

| Directory | Language | Status |
|---|---|---|
| `include/astrolune/` | C ABI (compiled by both) | complete v1 core headers |
| `src/base/`, `src/crypto/`, `src/consensus/` | C23 | implemented |
| `src/vm/`, `src/state/`, `src/tx/`, `src/chain/` | C23 | implemented |
| `src/node/` | C23 | mempool, block ingress and durable state/chain storage implemented |
| `src/net/` | C23 | TCP transport, framed wire protocol, peer manager with gossip and range sync |
| `src/rpc/` | C23 | JSON codec and JSON-RPC server over HTTP/1.1 |
| `src/daemon/` | C23 | long-running node service gluing storage, P2P and RPC |
| `src/apps/alnode/` | C23 | CLI: keygen, genesis authoring, offline chain tools, `run` daemon |
| `abi/` | C++23 | boundary enforcement; Trocto/SDK not yet written |
| `tests/c/` | C23 | 16 behavioral/header suites plus determinism fixture |

The top-level `CMakeLists.txt` reports a missing component rather than skipping it silently, so a partial checkout is visible at configure time. Per-module detail is in [../08-implementation/implementation-status.md](../08-implementation/implementation-status.md).

---

## 4. Deferred

- **Storage layer** (holding large volumes of data off-chain) — not a first-order task; we will return to it later.
- **Proxy layer** (traffic anonymisation / anti-censorship) — not a first-order task; we will return to it later.

Both have skeleton documents in [../06-services/](../06-services/) recording their open questions so that the deferral is a decision on record rather than a gap.
