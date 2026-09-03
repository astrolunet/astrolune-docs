# `.lune` DNS zone

**Status: skeleton — deliberately deferred.** [../00-overview/vision.md](../00-overview/vision.md) §3 lists this as out of scope for the first phase. This document exists so the deferral is a decision on record with its open questions written down, rather than a gap someone rediscovers later.

Nothing is implemented. Nothing should be, until the VM, state and transaction layers exist — a naming service is a contract plus a resolver, and neither can be written without them.

---

## 1. The idea

A decentralised naming system in a `.lune` namespace: human-readable names resolving to Astrolune addresses, content hashes, or conventional network records, with ownership recorded on-chain rather than in a registrar's database.

The comparison points are ENS (`.eth`), Handshake, and Unstoppable Domains. All three have been running long enough that their problems are known, which is the main reason to defer rather than design now: the interesting questions here have been answered badly several times in public.

---

## 2. Why it is deferred

1. **It is a contract, not core infrastructure.** Registration, ownership, transfer and expiry are exactly what a smart contract is for. Building it into the protocol would be building something into consensus that does not need to be there.
2. **It depends on everything unwritten.** No VM, no state layer, no transaction types.
3. **The hard part is not on-chain.** Getting a name to resolve in an ordinary browser is a client and infrastructure problem — ENS's on-chain registry took far less effort than making `.eth` resolve anywhere.

---

## 3. Open questions

### 3.1. On-chain registry or protocol-level

Almost certainly a contract. Worth writing down so it is not revisited.

### 3.2. Name allocation

First-come-first-served invites squatting; auctions (Handshake, early ENS) price it but exclude; rent or renewal (current ENS) prevents permanent squatting on dead names but adds an expiry mechanism that must be timed off block height, not wall-clock time.

Related: reserved names. Whether existing trademarks, or names matching real DNS TLDs, are blocked. Every project that did not decide this early regretted it.

### 3.3. Record types

Address, content hash, text records, and possibly A/AAAA/TXT for interoperability with conventional DNS. A record set that is a flat key→value map is the flexible choice, and it makes validation the client's problem.

### 3.4. Resolution outside the chain

The genuinely hard part. Options: a browser extension, a local DNS proxy on 127.0.0.1, a DoH endpoint, or a gateway domain (`name.lune.example`) — the last of which reintroduces a central point and thereby defeats the purpose. **No decentralised naming system has solved this well.** Any design should state honestly which trade-off it is taking rather than claiming to have avoided it.

### 3.5. Subdomains and delegation

Whether a name owner can issue subdomains, and whether those are on-chain or delegated off-chain.

### 3.6. Interaction with the proxy service

If [proxy.md](proxy.md) is ever built, `.lune` resolution through it is the natural pairing — and the natural place for a metadata leak, since resolution reveals what a user is looking up. Worth noting now: **a naming service is a surveillance surface.** DNS-over-HTTPS exists because plaintext DNS was one.

---

## 4. What would need to exist first

`al_vm`, `al_state`, `al_tx`, a working Trocto compiler, and a settled answer to [../04-state/state-model.md](../04-state/state-model.md) §2.1 — because whether a name is an *account entry* or an *owned object* is the account-versus-resource question again, and a name is the textbook example of a thing you want to be an owned, non-duplicable object.

---

## 5. What "done" means for this document

It stays a skeleton until the first phase is complete. Reopening it earlier than that is a scope error, and this line is here to make that explicit.
