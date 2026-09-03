# Building and testing

CMake 3.25+ and Ninja are required. The C consensus core targets the portable
C23 subset supported by GCC, Clang and MSVC; boundary tooling uses C++23.

## Quick start

```sh
cmake --workflow --preset dev
```

On Windows, use the wrapper so compiler and sanitizer runtime paths are set:

```bat
scripts\build.bat dev test
```

Build output is written below `build/<preset>/`. In-source builds are rejected.

## Presets

| Preset | Purpose |
|---|---|
| `dev` | Debug build, warnings enabled, full tests and corpus runners |
| `release` | Optimized build with debug information |
| `dist` | Release, LTO/unity, no tests or fuzz targets |
| `tiny` | Core/node configuration without tools, tests or fuzz targets |
| `ci` | Strict warnings-as-errors build and test workflow |
| `asan` | ASan/UBSan on GCC/Clang; ASan on MSVC |
| `fuzz` | Clang libFuzzer with ASan/UBSan |

For sanitizer runs, set:

```sh
ASAN_OPTIONS=allocator_may_return_null=1
```

MSVC supplies AddressSanitizer but not UBSan. CI therefore runs Clang
ASan/UBSan on Linux and MSVC ASan on Windows.

## Build invariants

- The core is compiled and linked as C; public headers compile independently as
  both C23 and C++23.
- Link dependencies enforce module direction.
- `abi_contract.h` pins public sizes, alignment, offsets and enum values in both
  languages.
- `boundary_symbols.cpp` resolves every public C function from C++.
- The dependency-free manifest checker compares every `AL_PUBLIC` declaration
  against that link table and rejects omissions or duplicates. The current
  manifest contains 221 functions.
- Strict presets promote the curated GCC, Clang and MSVC warning sets to errors.

## Tests and fuzzing

The development configuration registers the full set of CTest entries shown by
the generated build (currently 24): behavioral/header suites plus transaction,
ALVM, block/genesis and SMT-proof corpus runners.

```sh
ctest --preset dev
```

Each fuzz target is also a normal executable that consumes the checked-in golden
corpus. Clang builds the same sources with libFuzzer:

```sh
cmake --preset fuzz
cmake --build --preset fuzz
./build/fuzz/bin/fuzz_tx -runs=100 fuzz/corpus/tx
```

GitHub Actions runs strict GCC, Clang and MSVC builds, Linux Clang ASan/UBSan,
Windows MSVC ASan, short libFuzzer campaigns and cross-toolchain comparison of
the deterministic block digest. Windows CI also runs the two-validator restart
scenario and the four-validator quorum-boundary scenario.

## Signature backend

The default `dev` backend has no external dependency and is intentionally
forgeable. To compile the real Ed25519 signature path, install libsodium 1.0.18
or newer and configure explicitly:

```sh
cmake --preset dev -DASTROLUNE_CRYPTO_BACKEND=sodium -DSodium_ROOT=/path/to/libsodium
cmake --build --preset dev
ctest --preset dev
```

`Sodium_ROOT` may be omitted for a system installation. This mode still returns
false from `al_crypto_is_secure()` because VRF/VDF remain development
implementations; the flag must not be overridden by deployment tooling.

## Node CLI

`alnode` currently provides the transport-independent node path. It can create
an empty canonical development genesis, produce the next local block and replay
canonical block files through the full decoder, parent checks, VM/state
execution and commitment checks:

```sh
./build/dev/bin/alnode init-genesis genesis.bin 42
./build/dev/bin/alnode produce-block genesis.bin block-0.bin
./build/dev/bin/alnode produce-block genesis.bin block-1.bin block-0.bin
./build/dev/bin/alnode verify-chain genesis.bin block-0.bin block-1.bin
./build/dev/bin/alnode init-node genesis.bin node-data
./build/dev/bin/alnode import-blocks genesis.bin node-data block-0.bin block-1.bin
./build/dev/bin/alnode node-head genesis.bin node-data
```

The producer currently starts with an empty process-local mempool; transport/RPC
ingress is the next node-layer boundary. Stateless commands still require an
empty genesis root; the durable commands can reopen roots already present in
their genesis-bound data directory. Importing an external non-empty genesis
snapshot is not implemented. All commands print the crypto safety warning rather
than allowing the development backend to look deployable.

## Optional components

Missing optional directories are reported and skipped. `alnode` is built when
node applications are enabled; `alc`, packaging, installation rules, coverage
and production benchmarks remain future work.
