# Smart contract languages
### Trocto (.tc) and Regol (.rg)

A companion to [../01-consensus/potb.md](../01-consensus/potb.md) and [../02-architecture/system-architecture.md](../02-architecture/system-architecture.md).

**Status: v0.2 implemented.** `tools/` contains the compiler (C++23, target
`trocto`) with both tiers wired end to end: Trocto lowers to Regol, Regol
assembles to an ALVM container, and every artifact is re-validated through
the same `al_vm_validate` deployment performs. `tests/cxx/test_lang.cpp` executes
compiled contracts on the real VM against a mock host; `examples/counter.tc`
and `examples/token.tc` are reference samples. The long-term language design
below is unchanged; what follows is the shipped subset.

---

## 1. The principle: two tiers of one system, not two independent languages

The model is the one Ethereum uses with Solidity/Yul, or Aptos/Sui with Move/Move-IR: the high-level language compiles into the low-level one rather than existing alongside it.

```
Developer writes .tc  →  Trocto compiler  →  Regol (.rg)  →  VM bytecode
                                                 ↑
                        an advanced developer may write .rg directly
```

---

## 2. Trocto (.tc) — high-level, safe by default

**Who it is for:** the bulk of developers — tokens, NFTs, game logic, standard protocols.

**Principles:**
- Safe by default — protection against the typical contract vulnerabilities is built into the compiler (integer overflow, reentrancy, unchecked calls), by analogy with how Solidity 0.8+ made overflow checks the standard rather than an option.
- Readable, declarative syntax — the goal is that a developer with Solidity/TypeScript experience can start writing Trocto without a long ramp-up.
- Deliberately limited access to low-level primitives, so that by default you cannot accidentally shoot yourself in the foot.

**Not for:** situations that need precise control over resource consumption (gas/execution) or access to VM capabilities that Trocto intentionally hides for safety.

---

## 3. Regol (.rg) — low-level, full control

**Who it is for:** advanced developers, infrastructure protocols, and library authors whose libraries then become the foundation for Trocto contracts.

**Principles:**
- Direct access to VM primitives without Trocto's protective wrappers.
- Precise control over resource consumption — important for high-load protocols (DEXes, on-chain games with frequent calls) where every unit of gas counts.
- Less protection out of the box — responsibility for safety lies with the developer, as in any language close to the bytecode level.

**Also:** this is the language Trocto compiles into — so Regol is not only directly available but is the intermediate representation (IR) of the whole compilation system.

---

## 3.5. Trocto syntax sample (Rust-like)

A draft example — a simple fungible token contract, to fix the feel of the syntax:

```rust
contract Token {
    state {
        name: string,
        total_supply: u64,
        balances: map<address, u64>,
    }

    // constructor — called once, at deployment
    init(name: string, initial_supply: u64) {
        self.name = name;
        self.total_supply = initial_supply;
        self.balances.insert(sender(), initial_supply);
    }

    // strict typing plus mandatory precondition checks (require)
    pub fn transfer(&mut self, to: address, amount: u64) -> Result<(), Error> {
        require(self.balances[sender()] >= amount, Error::InsufficientBalance);
        require(to != address::zero(), Error::InvalidAddress);

        self.balances[sender()] -= amount;   // overflow-checked by default
        self.balances[to] += amount;

        emit Transfer { from: sender(), to, amount };
        Ok(())
    }

    pub fn balance_of(&self, owner: address) -> u64 {
        self.balances.get(owner).unwrap_or(0)
    }
}
```

**What the example already shows, and what is worth fixing as syntax principles:**
- `contract` instead of Rust's `struct` + `impl` — a dedicated construct for smart contracts
- `state { }` — an explicit, separated block of blockchain-stored data (it is immediately visible what "lives" between calls and what is transient)
- `Result<(), Error>` and `require(...)` — borrowing from Rust the idea of explicit error handling instead of silent exceptions
- Integer operations (`+=`, `-=`) are checked by default (no `wrapping_*`/`unchecked` without asking for it explicitly — unlike bare Rust, where overflow wraps silently in a release build)
- `emit` for events — the familiar concept from Solidity, without breaking the Rust-likeness
- `pub fn` / `&mut self` / `&self` — direct borrowing of Rust's method and mutability syntax, so developers with Rust experience can read Trocto almost without learning it

---

## 4. Shipped subset: Trocto v0.2

The v0.2 tier adds constructors, string literals, expanded map types, and the
import system on top of the v0.1 integer foundation.

```rust
contract Token {
    state {
        total_supply: u64,
        balances: map<address,u64>,
        allowances: map<address,u64>,
    }

    init(initial_supply: u64) {              // constructor
        self.total_supply = initial_supply;
        balances[sender()] = initial_supply;
        emit Transfer(0, initial_supply);
    }

    pub fn transfer(to: address, amount: u64) -> u64 {
        let b = balances[sender()];
        require(b >= amount, 2);              // overflow traps from the VM
        balances[sender()] = b - amount;
        balances[to] += amount;
        emit Transfer(amount, 1);
        return 1;
    }

    pub fn balance_of(owner: address) -> u64 {
        return balances[owner];
    }
}
```

Grammar surface (v0.2 additions over v0.1):
- `init(...)` — constructor, compiled as the default entrypoint (function 0).
  Called once at deployment; receives calldata like a public function.
- `import "path";` — file-level or contract-level import. The imported file
  is parsed and validated; function linking is planned for v0.3.
- `"string"` — string literals, materialized in linear memory.
- `assert(cond);` — panics with revert code 0 (unrecoverable assertion).
- `map<u64,u64>`, `map<address,u64>`, `map<address,address>` — expanded map
  key/value type combinations.
- `address` type for parameters, locals, and map keys.

Fixed decisions the compiler owns:

- **State keys.** A field's storage key is
  `tagged_hash(contract_data domain, "field.<contract>.<name>")`, computed at
  compile time and embedded as constants. Absent slots read as zero.
- **Public ABI.** Each `pub fn` compiles to a zero-parameter ALVM function:
  a prologue validates calldata length (revert code 1 on mismatch) and
  decodes n×8 little-endian arguments into frame slots; results return
  through RETURN from reserved memory.
- **Frame protocol.** Plain `fn`s use VM CALL/RET with stack parameters;
  public functions are external-only in v0.1 (calling one internally is a
  compile error - RETURN would tear down caller frames).
- **Events.** `emit Name(…)` hashes the name in the event domain for its
  topic and serializes u64 args into linear memory.
- **Memory layout** per function: `[0,32)` scratch key/topic, `[32,64)`
  scratch data/revert code, `[64,72)` result slot, then locals and decoded
  parameters. All offsets static; no dynamic allocation.

Not yet (tracked in section 5): imports are validated but function linking is
not yet implemented (v0.3); generics, resource types, and the standard library.

## 4.5 Regol v0.1

Regol text is the IR made visible: mnemonics map 1:1 onto the ISA table, with
named functions, `.label` definitions, symbolic jump targets, and named hosts
(`host storage_get`). The first function is the default entrypoint and takes
no parameters, mirroring the container contract. `trocto --emit-regol` prints
the lowered form of any Trocto contract; that output reassembles to an
equivalent container.

## 5. Kreep — rejected

A third language (Kreep, `.kp`) for internal network tooling was considered initially. Decided against: it is not needed as a separate language. Network tooling (CLI, RPC, indexer) is developed in C++ (see [../02-architecture/system-architecture.md](../02-architecture/system-architecture.md)).

---

## 5. Open questions for further work

1. ~~The exact Trocto syntax~~ — fixed: Rust-like (see section 3.5). Still to finish: the full keyword list, the module/import system, generics
2. The type system — a resource model (like Move, for strict control of tokens/NFTs as objects that cannot be transferred by mistake) or a classical one
3. The gas/execution pricing model — how operation cost is computed in Trocto and in Regol
4. The standard library — which standards to build in from the start (the equivalent of ERC-20/ERC-721) for tokens and NFTs
5. Formal verification — whether a tool like Move Prover is needed for critical Regol contracts

Items 2 and 3 block the compiler: neither the type checker nor the code generator can be written without them. They are the reason the language specification sits at the end of the roadmap rather than the start. Tracked in [../07-roadmap/open-questions.md](../07-roadmap/open-questions.md).
