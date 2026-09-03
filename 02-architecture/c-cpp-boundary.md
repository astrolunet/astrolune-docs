# The C / C++ boundary

The rule for *which* language a component is written in is in [system-architecture.md](system-architecture.md). This document covers the narrower question: what may appear in the headers that **both** languages compile, and why each restriction exists.

`include/astrolune/` is the only place the two halves of the project meet. Every file there is compiled by the C23 core and by the C++23 tooling. A construct that is legal C23 but not legal C++23 — or one whose *layout* differs between the two — breaks the build or, worse, silently produces two incompatible views of the same struct.

---

## 1. The rules

Each rule below is stated as a prohibition, followed by what it costs and what to use instead.

### 1.1. No `extern "C"` written by hand

Every public header wraps its contents in `AL_EXTERN_C_BEGIN` / `AL_EXTERN_C_END`. In C those expand to nothing; in C++ they expand to `extern "C" {` / `}`. Writing the C++ form directly would not compile as C.

### 1.2. No C-only keywords in declarations

`restrict` is a C keyword with no standard C++ spelling. Public prototypes use `AL_RESTRICT`, which expands to `restrict` in C, `__restrict` under MSVC C++ and `__restrict__` under GCC/Clang C++.

`_Atomic` does not exist in C++ at all, and `<stdatomic.h>` and `<atomic>` do not agree on layout. Nothing atomic appears in a public header; where the core needs atomics they stay in a `.c` file behind an opaque type.

A C cast is *legal* C++, so it is not a portability problem — it is a warning problem. The tooling builds with `-Wold-style-cast`, which is an error under the `ci` and `asan` presets, so a cast written into a macro in a shared header makes every C++ user of that macro fail to build. Casts in the public headers therefore route through **`AL_CAST(T, v)`** — `static_cast` in C++, character-for-character the hand-written cast in C, so the core is unaffected. `AL_TRUE`, `AL_FALSE`, `AL_FIXED_ONE`, `AL_ARENA_NEW` and `AL_ARENA_NEW_ARRAY` all use it. (`(void)` casts are exempt from the warning and are left alone.) This was found by writing the first C++ TU, not by review: all five macros predate it and none had ever been expanded by a C++ compiler.

### 1.3. No variable-length arrays

C99 VLAs are not C++ at all, and they are a stack-overflow vector on an attacker-influenced length besides. `-Wvla` is on in `cmake/AstroluneWarnings.cmake`, so a VLA anywhere in the project is a warning and an error under the `ci` preset. Sizes are either compile-time constants (`AL_POTB_MAX_COMMITTEE`) or arena allocations.

### 1.4. No anonymous unions inside structs whose layout C++ would change

C11 anonymous members and C++ anonymous unions have overlapping but not identical rules, particularly around non-trivial members. Named members only.

### 1.5. No `constexpr`, `nullptr` or `enum : type`

These are the practical C23 gap. MSVC reports `__STDC_VERSION__ == 202312L` but does not implement them — verified against MSVC 19.51. The core therefore targets a *portable subset* of C23 rather than C23 as specified.

Instead:
- constants are `#define` or `static const`,
- null is `NULL`,
- enum width is pinned by an explicit sentinel (below).

### 1.6. Every enum has an explicit sentinel

```c
typedef enum al_status {
    AL_OK = 0,
    /* ... */
    AL_STATUS_SENTINEL = 0x7fffffff  /* pins the enum's width */
} al_status;
```

Without a fixed underlying type (1.5) the compiler picks the smallest type that fits the enumerators — and it may pick differently for C and C++, or between two compilers. A `0x7fffffff` sentinel forces at least 32 bits everywhere. This matters because an enum crosses the ABI in `al_status` return values and in struct fields.

Present on `al_status`, `al_potb_level`, `al_potb_offence` and `al_crypto_backend_kind`.

### 1.7. Fixed-width types for anything serialised or hashed

`al_u8` … `al_i64` rather than `int`/`long`. A consensus-visible field whose width depends on the data model is a chain split waiting for a platform change.

`al_bool` is `al_u8`, not C23's `bool`: `bool` is one byte in practice but its size is not pinned by the standard, and in C++ it is a distinct type with its own conversion rules.

### 1.8. No floating point in a consensus-visible position

Not a language-compatibility rule but the most important one on the list, so it belongs here too. Float results depend on FPU mode, instruction selection, whether the compiler contracted a multiply-add into an FMA, and the platform's libm. Two honest validators would compute different weights from identical inputs. Every non-integer quantity in the protocol is an `al_fixed` (Q32.32 in an `int64_t`) from `astrolune/fixed.h`.

`-Wdouble-promotion` is on, so an accidental promotion is visible.

### 1.9. Hashes and addresses are structs, not arrays

```c
typedef struct al_hash256 { al_u8 bytes[AL_HASH_SIZE]; } al_hash256;
typedef struct al_address { al_u8 bytes[AL_ADDRESS_SIZE]; } al_address;
```

A bare `al_u8[32]` decays to a pointer, cannot be returned or assigned, and — the real reason — is the same type as every other 32-byte array, so passing an address where a public key is expected compiles cleanly. Wrapping them makes them values and makes them distinct. `AL_STATIC_ASSERT` pins the sizes.

### 1.10. Attributes go through macros

`AL_NODISCARD`, `AL_MAYBE_UNUSED`, `AL_NORETURN`, `AL_FORCEINLINE`, `AL_ALIGNAS`, `AL_LIKELY`, `AL_UNREACHABLE`. C23 attribute syntax is only partly available across the three supported compilers, and the core must build identically on all of them. `AL_STATIC_ASSERT` likewise routes to `static_assert` in C++ and `_Static_assert` in C.

---

## 2. Error handling across the boundary

The core reports failure by return value — never by exception, `errno` or a global. A function that produces a value takes an out-parameter and returns `al_status`. Anything that can fail is `AL_NODISCARD`.

This is not a stylistic preference. An exception thrown across an `extern "C"` frame is undefined behaviour, and the C++ tooling links C archives directly. A single convention that works in both directions removes the question.

`AL_TRY(expr)` is the early-return helper for the core's internals. It is deliberately *not* a `goto cleanup` macro: the core allocates from arenas that the caller resets, so a plain early return leaks nothing.

---

## 3. Ownership across the boundary

- `al_bytes` / `al_bytes_mut` are non-owning views. The core never takes ownership of caller memory and never hands back memory the caller must free.
- Anything with a lifetime comes from an `al_arena` supplied by the caller, and is valid until that arena is reset at or below the mark it was allocated after.
- Nothing in the core stores an arena pointer across block boundaries.

For the C++ side this means the natural wrapper is a non-owning span type or a `std::vector` the caller keeps alive — never a smart pointer holding core-allocated memory.

---

## 4. How the rules are enforced

| Rule | Enforcement |
|---|---|
| Core is C only | `astrolune_add_core_library()` sets `LINKER_LANGUAGE C`; a `.cpp` in `core/` fails |
| Tooling headers invisible to the core | only `astrolune_add_tool_library()` adds `cpp/include` to the include path |
| Layering | the seven core libraries are link edges; a violation is a link error |
| 1.1 `extern "C"` present | `al_abi_boundary` takes the address of all 221 public functions from C++ — a header missing the wrapper is an unresolved-symbol **link** error (§4.1) |
| 1.2–1.4, 1.10 syntax | `cpp/abi/header_*.cpp` compile each public header as C++23, one TU per header |
| 1.5 no `constexpr`/`nullptr`/`enum : type` | the same TUs, under MSVC 19.51 — the compiler that lacks them |
| 1.6 enum width | `contract_cxx.cpp` asserts `sizeof(std::underlying_type<E>::type) == 4` for all nine public enums |
| 1.7 `al_bool` is not `bool` | `!std::is_same<al_bool, bool>` plus `AL_CAST(al_bool, 2) == 2` — a real `bool` collapses 2 to 1 |
| 1.9 distinct wrapper types | `!std::is_same` over the hash/address/pubkey/seckey/sig pairs |
| Identical layout in both languages | `abi_contract.h` is compiled by `contract_c.c` **and** `contract_cxx.cpp` (§4.1) |
| No VLAs | `-Wvla` |
| No float promotion | `-Wdouble-promotion` |
| Prototypes match | `-Wstrict-prototypes`, `-Wmissing-prototypes`, `-Wbad-function-cast` (C only) |
| Lossy conversions | `-Wconversion`, `-Wsign-conversion`; MSVC `/w14242`, `/w14254`, `/w14826` |
| Enum switch coverage | `-Wswitch-enum` — a new status or opcode forces every switch to be revisited |
| Everything above, as errors | `ASTROLUNE_WERROR=ON`, set by the `ci` and `asan` presets |

The one warning silenced deliberately is MSVC `/wd4200` (zero-sized array in a struct), because flexible array members are used intentionally in the bytecode and state structures.

### 4.1 Why the check is four things and not one file

The obvious form of this check — one `.cpp` including every public header — is what the roadmap asked for and what `tests/c/test_cpp_headers.cpp` already was. It is not enough, for two reasons that are worth stating because both were verified by breaking the rule and watching the build.

**Compiling proves nothing about rule 1.1.** A header that forgets `AL_EXTERN_C_BEGIN` compiles perfectly as C++; its declarations simply acquire C++ linkage. The mismatch surfaces only when something tries to *resolve* one of those names against the C archive. So `al_abi_boundary` is an **executable**, not a library — a static library archives objects and never resolves a symbol — and `boundary_symbols.cpp` takes the address of every public function into a table. Commenting the wrapper out of `vm.h` leaves `test_cpp_headers` linking cleanly and fails `al_abi_boundary` with four unresolved C++-mangled symbols (`?al_vm_config_default@@YA?AUal_vm_config@@XZ`, …).

**One aggregate TU hides a header that forgets its own includes.** Included after its dependency, a non-self-sufficient header is invisible. Removing `#include "astrolune/state.h"` from `tx.h` compiles fine in both `header_all.cpp` and `test_cpp_headers.cpp` — alphabetical order puts `state.h` first — and fails only `header_tx.cpp`. Hence one TU per header, plus `header_all.cpp` (which includes all eleven **twice**, for macro collisions and include guards) as a separate concern. `UNITY_BUILD` is forced `OFF` on the target: the `dist` preset would otherwise concatenate the per-header TUs and report success while checking nothing.

The layout half cannot be asserted from one language at all — a comparison needs a shared literal — so `abi_contract.h` states every size, alignment, offset and public constant as a number and is compiled by both a C and a C++ TU. That is also what makes it the ABI record: changing a public struct now fails the build with the field named, rather than changing what a `.tc` contract sees. The numbers were measured, not assumed; MSVC 19.51's C and C++ layouts agree on all of them.

`al_abi_manifest_check` closes the former omission gap: it parses every
`AL_PUBLIC` declaration without external dependencies and compares the result
with `boundary_symbols.cpp`, rejecting missing and duplicate entries during a
normal build.
