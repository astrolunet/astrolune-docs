# Astrolune — Documentation for Engineers and Validators

This set is reorganized from the project's original technical documentation
(`astrolune-docs`) into a format oriented not around the repository's
folder structure, but around **how the network actually works, how it
computes things, and what you need to know to operate it or build on it.**
Nothing here is invented: every formula, table, and warning is carried over
in substance from the original design and spec documents; only the
presentation has been reworked — a continuous explanation instead of files
scattered across repository directories.

Astrolune is a decentralized network built on a custom consensus model
called **PoTB** (Proof of Trusted Behavior), with its own virtual machine
(ALVM), a two-tier smart contract language system (Trocto/Regol), and, in
the future, a set of infrastructure services (`.lune` DNS, storage, proxy).

**Русская версия этой документации доступна здесь:** [`RU/README.md`](RU/README.md)

## How to read this

| If you need to... | Read |
|---|---|
| understand the project's overall idea | [00-overview.md](00-overview.md) |
| understand how node weight is computed and who finalizes blocks | [01-consensus-potb.md](01-consensus-potb.md) |
| understand the architecture and the C/C++ boundary | [02-architecture.md](02-architecture.md) |
| understand contract execution and the gas model | [03-vm-and-gas.md](03-vm-and-gas.md) |
| understand the state, transaction, and block model | [04-state-and-transactions.md](04-state-and-transactions.md) |
| write contracts (Trocto/Regol) | [05-contract-languages.md](05-contract-languages.md) |
| learn about deferred services (.lune, storage, proxy) | [06-deferred-services.md](06-deferred-services.md) |
| **learn validator requirements and rights** | [07-validator-requirements.md](07-validator-requirements.md) |
| understand what's actually implemented vs. an open risk | [08-implementation-status.md](08-implementation-status.md) |

## The most important thing to understand before anything else

1. **This documentation is honest about its own incompleteness.** In
   several places (the PoTB section, the cryptography section), it states
   plainly: "this is not proven," "this is a heuristic, not a guarantee."
   This is not a drafting oversight — it is a deliberate stance by the
   protocol's authors: open problems are recorded as open, not presented
   as solved. If something in the text reads like an admission of
   weakness, it isn't a forgotten draft note — it's information the reader
   needs.

2. **Where the C library header (`include/astrolune/*.h`) and a document
   disagree — the header wins.** That's what nodes actually execute; a
   discrepancy is treated as a documentation bug, not a code bug. Known
   discrepancies are tracked separately (see
   [08-implementation-status.md](08-implementation-status.md)).

3. **Cryptography is not production-ready right now.** The default
   signing backend and all current VRF/VDF implementations are
   deliberately insecure dev stand-ins. Before running anything that needs
   to be secure, read the warning in
   [07-validator-requirements.md](07-validator-requirements.md) and the
   cryptography section in [02-architecture.md](02-architecture.md).

## Status legend

Carried over unchanged from the source documentation:

- **stable** — designed and agreed upon; rarely changes.
- **draft** — work in progress; details still being worked out.
- **skeleton** — a frame with open questions fixed in writing;
  substantive specification is deliberately deferred. Such a file records
  what's already constrained by existing code and what's genuinely
  unresolved — it doesn't invent a spec for the sake of looking finished.
- **current** — describes the code tree as it is now and goes stale as the
  code changes.

## License

The original documentation is distributed under the MIT license
(Astrolune contributors, 2026).