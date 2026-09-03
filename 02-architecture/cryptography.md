# Cryptography

Astrolune's cryptography is in two halves with very different maturity. This document states plainly which is which, why, and what has to happen before a public network.

| Layer | File | Status |
|---|---|---|
| SHA-256 | `src/crypto/sha256.c` | **Real.** Checked against the NIST vectors. |
| HMAC-SHA256, HKDF | `src/crypto/hmac.c` | **Real.** Checked against RFC 4231 / RFC 5869. |
| Merkle trees | `src/crypto/merkle.c` | **Real.** |
| Key derivation, addresses | `src/crypto/keys.c` | Address derivation real; keypair derivation depends on the backend. |
| Signatures | `src/crypto/dev_backend.c` or `sodium_backend.c` | Development stub by default; real Ed25519 with explicit sodium configuration. |
| VRF and VDF | `src/crypto/dev_vrf_vdf.c` | **Insecure development stubs.** Must be replaced or removed. |

---

## 1. One hash function: SHA-256

Astrolune uses SHA-256 everywhere. One primitive:

- keeps the consensus-critical surface small,
- is implementable in ~200 lines of auditable C,
- is hardware-accelerated on every current CPU,
- has decades of analysis behind it.

The cost is that it is slower than BLAKE3 on long inputs. That trade is accepted: the protocol hashes many small structures (transactions, headers, tree nodes) rather than few large ones, which is the regime where SHA-256's per-block cost matters least and where BLAKE3's tree parallelism buys least.

`al_sha256d` (SHA-256 applied twice) exists for the cases where a length-extension property would be exploitable and the input length is not itself authenticated.

### 1.1. Domain separation is mandatory

Every structural hash in the protocol is tagged. Hashing a transaction and hashing a block header must never be able to produce the same digest from the same bytes, or an attacker can present one object where another is expected.

`al_hash_tagged` absorbs `SHA256(tag) || data` rather than `tag || data`. The indirection matters: with a raw variable-length prefix, a longer tag and a shorter message can produce the same byte stream as a shorter tag and a longer message. Hashing the tag first fixes its contribution at 32 bytes.

The `AL_TAG_*` list in `astrolune/hash.h` enumerates every domain the protocol defines. **Adding a hashed structure means adding a tag. Reusing an existing tag for a new structure is a consensus bug.**

Two entries on that list are worth calling out because their separation is load-bearing rather than tidiness:

```
AL_TAG_EPOCH_SEED    "astrolune.epoch.seed.v1"
AL_TAG_EPOCH_COMMIT  "astrolune.epoch.commit.v1"
```

The commit-reveal scheme for the epoch seed publishes a commitment in round one and the committed value in round two. If both were derived under one tag, the published commitment would *equal* the value it later contributes, the entire seed would be computable from the commit round alone, and the reveal round would protect nothing.

### 1.2. Addresses are the full digest

`al_address_from_pubkey` uses all 32 bytes of the tagged SHA-256, not a truncation. A 20-byte address gives 80-bit collision resistance, which is uncomfortably close to feasible for a chain intended to be long-lived, and the 12 bytes saved per account do not justify designing that ceiling in.

Contract addresses are `H(tag || deployer || nonce || code_hash)` — deterministic, so a deployer can compute the address before the transaction lands, and collision-free across deployers.

---

## 2. The development primitives — read this before using them for anything

The default backend and `src/crypto/dev_vrf_vdf.c` implement the `al_sign_*`,
`al_vrf_*` and `al_vdf_*` interfaces with hash constructions. The optional
`ASTROLUNE_CRYPTO_BACKEND=sodium` configuration replaces only key derivation and
signatures with libsodium Ed25519. The remaining development primitives mean
`al_crypto_is_secure()` correctly returns false in both configurations.

The development implementation is:

- **deterministic and self-consistent**, so the node, the VM, the state machine and the whole test suite exercise the real code paths;
- **not cryptographically secure.** It does not implement Ed25519.

### 2.1. What it actually does

A "signature" is two halves:

```
sig[0:32]   H(tag_sig  || pk        || message)   - checked by al_verify
sig[32:64]  H(tag_bind || sk_scalar || message)   - carried, never checked
```

The verified half is computable from public data. **Anyone can forge a signature for any key**, and anyone who reads the file can see how.

The unverified half is computable only by the key holder. That looks pointless and is not: signing must genuinely require the secret key, or every layer above would compile and pass its tests while never actually using a key — and the day a real backend lands, all of those call sites would break at once. Carrying the binding half keeps the API honest about who can sign, keeps the 64-byte wire format identical to Ed25519's, and confines the eventual swap to this one file.

The secret key layout is stable across backends: `sk[0:32]` is backend secret
material (the original seed for libsodium) and `sk[32:64]` is the cached public
key. Neither the ABI nor stored key width changes when the backend does.

### 2.2. The VRF stub

PoTB selects each block's committee with a VRF. The three properties consensus depends on are: the output is unpredictable before evaluation, unique per `(key, input)`, and publicly verifiable afterwards — so nobody can grind their way into a committee and nobody can deny a result once produced.

The stub is deterministic and verifiable but **not unpredictable to the holder of the secret key**. Uniqueness and verifiability hold; unpredictability does not.

### 2.3. The VDF stub

The seed for committee selection is agreed by commit-reveal. Commit-reveal alone lets the last revealer see everyone else's contribution and choose whether to reveal, biasing the result. A VDF closes that: the seed passes through a function that provably takes wall-clock time to evaluate but is fast to verify, so by the time anyone could compute the outcome, the window to act on it has shut.

The stand-in is an iterated hash. It is genuinely sequential, but it has **no succinct proof**, so `al_vdf_verify` recomputes the whole chain and verification costs exactly as much as evaluation. Fast verification is the entire point of a VDF, and this one does not have it.

A real deployment needs a proper construction — Wesolowski or Pietrzak over a class group of unknown order. Whether the VDF branch is taken at all is the open question in [../01-consensus/potb.md](../01-consensus/potb.md) section 5.

### 2.4. Why a hand-rolled Ed25519 was not attempted

Ed25519 is not hard to write incorrectly. Field arithmetic that is constant-time on paper but not after the optimiser has seen it, incomplete point validation, small-subgroup and cofactor handling, malleable signature encodings, and the RFC 8032 canonicality rules are all real, documented sources of breakage in real implementations. Writing one as part of a wider project, without a dedicated review and without side-channel measurement, would produce something that looks finished and passes its own tests — the worst possible failure mode for a signature scheme, because it is indistinguishable from a working one until it is attacked.

A stub that is *obviously* and *loudly* broken is safer than an implementation that is subtly broken. That is the whole design decision here.

### 2.5. Guard rails

- Every stub function is marked `AL_CRYPTO_INSECURE` in its documentation.
- `al_crypto_backend()` reports either the development signature backend or the
  libsodium Ed25519 backend; the name also exposes the remaining dev VRF/VDF.
- `al_crypto_is_secure()` returns `AL_FALSE`. Node startup and key generation check it and warn loudly.
- The backend's internal tags all carry a `dev` segment (`astrolune.dev.sig.v1`) so they can never collide with a protocol tag.

These make shipping the stub to a public network hard to do by accident. They do not make it impossible, and no build-system trick can substitute for the migration below actually being done.

---

## 3. Migration checklist: dev backend → real Ed25519

Nothing outside `src/crypto/` depends on the construction, so this is a contained job. In order:

1. **Done: choose the source.** The production signature path uses libsodium's
   `crypto_sign_ed25519` implementation rather than project-local curve code.
2. **Done for development builds: add the dependency without breaking `tiny`.**
   Sodium is an explicit configuration, so dependency-free `dev` and `tiny`
   builds continue to work. A production distribution still needs a pinned
   static dependency artifact.
3. **Done: implement the key and signature API** against libsodium while keeping
   the existing 32-byte-public/64-byte-secret ABI.
4. **Done: reject non-canonical signatures explicitly.** The adapter checks
   `S < L` before invoking libsodium, and the test suite pins that rule.
5. **Done for this implementation: decide the cofactor question.** Astrolune
   adopts libsodium's strict Ed25519 verification semantics, including rejection
   of non-canonical points and small-order components. Alternative node
   implementations must match those acceptance rules.
6. **Replace the VRF** with a real construction (RFC 9381 ECVRF-EDWARDS25519-SHA512-TAI is the obvious candidate, since it reuses the same curve). Keep `AL_VRF_PROOF_SIZE` accurate — 80 bytes is the ECVRF proof size, which is why it was chosen.
7. **Replace or drop the VDF.** If the VRF-only branch is taken (see PoTB §5), `al_vdf_*` should be removed rather than left as a stub that looks usable.
8. **Set `al_crypto_is_secure()` to `AL_TRUE`** last, and only once every step
   above is done. The sodium build already reports
   `AL_CRYPTO_BACKEND_ED25519`, which describes its signature implementation,
   while the deployment gate remains false.
9. **Done for Ed25519: add RFC 8032 test vectors** to
   `tests/c/test_crypto.c`. Development goldens remain for the default backend.
10. **Run the suite under `asan`** — the sanitizer preset exists because both ASan and UBSan findings turn into non-deterministic execution, which for a blockchain means a chain split rather than a crash.

Until step 8, `al_crypto_is_secure()` returning `AL_FALSE` is correct and must not be edited to silence a warning.

---

## 4. Constant-time discipline

Where a comparison result is not already public, the comparison must not leak through timing:

- `al_bytes_eq_ct` for secrets, MACs and signatures. Plain `al_bytes_eq` short-circuits on the first differing byte and therefore leaks the length of the common prefix.
- `al_potb_epoch_seed_check` is constant-time, because a reveal is checked against a value an adversary chose and the result is not public until the round closes.
- `al_secure_zero` overwrites key material in a way the optimiser may not remove. Use it before key material goes out of scope.

Note the limit of what the current code can promise: the *stub* backend's operations are hash-based and their timing does not depend on secret data, so they are incidentally constant-time. A real backend must provide that property deliberately, and verifying it is part of step 1 above — not something the surrounding code can enforce.
