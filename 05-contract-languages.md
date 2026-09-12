# 5. Smart Contract Languages: Trocto (.tc) and Regol (.rg)

**Status: v0.2 implemented.** `tools/` contains the compiler (C++23,
target `trocto`) with both tiers wired through the whole pipeline: Trocto
lowers to Regol, Regol assembles into an ALVM container, every artifact
is re-validated through the same `al_vm_validate` used at deploy time.
`tests/cxx/test_lang.cpp` executes compiled contracts on the real VM
against a mock host; `examples/counter.tc` and `examples/token.tc` are
reference examples.

## 5.1. Principle: two tiers of one system, not two independent languages

The model is the same one Ethereum uses with Solidity/Yul, or
Aptos/Sui with Move/Move-IR: a high-level language compiles down to a
low-level one, rather than existing alongside it.

```
Developer writes .tc  →  Trocto compiler  →  Regol (.rg)  →  VM bytecode
                                                  ↑
                       an advanced developer may write .rg directly
```

## 5.2. Trocto (.tc) — high level, secure by default

**Audience:** the bulk of developers — tokens, NFTs, game logic,
standard protocols.

**Principles:**
- Secure by default — protection against common contract
  vulnerabilities is built into the compiler (integer overflow,
  reentrancy, unchecked calls), analogous to how Solidity 0.8+ made
  overflow checks the default rather than an option.
- Readable, declarative syntax — the goal is for a developer with
  Solidity/TypeScript experience to start writing Trocto without a long
  ramp-up.
- Deliberately restricted access to low-level primitives, so you can't
  easily shoot yourself in the foot by default.

**Not for:** situations requiring precise control over resource
consumption (gas/execution) or access to VM capabilities that Trocto
deliberately hides for safety.

## 5.3. Regol (.rg) — low level, full control

**Audience:** advanced developers, infrastructure protocols, and library
authors whose libraries then become the foundation for Trocto contracts.

**Principles:**
- Direct access to VM primitives without Trocto's protective wrappers.
- Precise control over resource consumption — matters for
  high-throughput protocols (DEXes, on-chain games with frequent calls)
  where every unit of gas counts.
- Less "out of the box" protection — responsibility for safety lies with
  the developer, as with any language close to the bytecode level.

**Also:** this is the language Trocto compiles into — so Regol is not
only directly accessible but also the intermediate representation (IR)
of the entire compilation system.

## 5.4. Trocto syntax — an example

A rough example — a simple fungible token contract, to capture the feel
of the syntax:

```rust
contract Token {
    state {
        name: string,
        total_supply: u64,
        balances: map<address, u64>,
    }

    // constructor — called once, at deploy time
    init(name: string, initial_supply: u64) {
        self.name = name;
        self.total_supply = initial_supply;
        self.balances.insert(sender(), initial_supply);
    }

    // strong typing plus mandatory precondition checks (require)
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

**What the example already shows, and what's worth establishing as
syntax principles:**
- `contract` instead of Rust's `struct` + `impl` combo — a dedicated
  construct for smart contracts;
- `state { }` — an explicit, separate block for data stored on-chain
  (immediately visible what "lives" between calls versus what's
  temporary);
- `Result<(), Error>` and `require(...)` — borrowing Rust's idea of
  explicit error handling instead of silent exceptions;
- integer operations (`+=`, `-=`) are checked by default (no
  `wrapping_*`/`unchecked` without an explicit request — unlike plain
  Rust, where overflow silently wraps in a release build);
- `emit` for events — a familiar concept from Solidity, without breaking
  the Rust resemblance;
- `pub fn` / `&mut self` / `&self` — a direct borrow of Rust's method and
  mutability syntax, so developers with Rust experience can read Trocto
  with almost no learning curve.

## 5.5. Implemented subset: Trocto v0.2

The v0.2 tier adds constructors, string literals, extended map types, and
an import system on top of the v0.1 integer foundation.

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
        require(b >= amount, 2);              // overflow — VM trap
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
- `init(...)` — a constructor, compiled as the default entry point
  (function 0). Called once at deploy time; receives calldata like a
  public function.
- `import "path";` — file-level or contract-level import. The imported
  file is parsed and validated; function linking is planned for v0.3.
- `"string"` — string literals, materialized in linear memory.
- `assert(cond);` — panics with revert code 0 (unrecoverable assertion).
- `map<u64,u64>`, `map<address,u64>`, `map<address,address>` — extended
  key/value type combinations for maps.
- The `address` type for parameters, locals, and map keys.

Fixed decisions the compiler is responsible for:

- **State keys.** A field's storage key is
  `tagged_hash(contract_data domain, "field.<contract>.<name>")`,
  computed at compile time and embedded as constants. Missing slots read
  as zero.
- **Public ABI.** Every `pub fn` compiles into a zero-parameter ALVM
  function: the prologue validates calldata length (revert code 1 on
  mismatch) and decodes n×8 little-endian arguments into frame slots;
  results return via RETURN from reserved memory.
- **Frame protocol.** Ordinary `fn` use VM CALL/RET with stack
  parameters; public functions in v0.1 are external-only (calling one
  from another is a compile error, since RETURN would unwind the
  caller's frames).
- **Events.** `emit Name(…)` hashes the name in the event domain for its
  topic and serializes u64 arguments into linear memory.
- **Per-function memory layout:** `[0,32)` — scratch key/topic,
  `[32,64)` — scratch data/revert code, `[64,72)` — result slot, then
  locals and decoded parameters. All offsets are static; there's no
  dynamic memory allocation.

Not yet implemented (tracked in open questions): imports are validated
but function linking isn't yet implemented (v0.3); generics, resource
types, and a standard library.

## 5.6. Regol v0.1

Regol text is the visible IR: mnemonics map 1:1 onto the ISA table, with
named functions, `.label` definitions, symbolic jump targets, and named
hosts (`host storage_get`). The first function is the default entry
point, with no parameters, mirroring the contract-container. `trocto
--emit-regol` prints the lowered form of any Trocto contract; this output
assembles back into an equivalent container.

## 5.7. Kreep — rejected

A third language (Kreep, `.kp`) for internal network tooling was
originally considered. Decision: against. It isn't needed as a separate
language. Network tooling (CLI, RPC, indexer) is developed in C++.

## 5.8. Open questions for further work

1. ~~Exact Trocto syntax~~ — fixed: Rust-like. Remaining: the full
   keyword list, the module/import system, generics.
2. Type system — a resource model (as in Move, for strict control of
   tokens/NFTs as objects that can't be accidentally transferred) or a
   classical one.
3. Gas/execution pricing model — how operation cost is computed in
   Trocto and in Regol.
4. Standard library — which standards to build in from the start
   (ERC-20/ERC-721 equivalents) for tokens and NFTs.
5. Formal verification — whether a tool like the Move Prover is needed
   for critical Regol contracts.

Items 2 and 3 block the compiler: neither the type checker nor the code
generator can be written without them. That's why the language spec sits
at the end of the roadmap, not the beginning.
