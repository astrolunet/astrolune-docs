# 3. Virtual Machine (ALVM) and the Resource Model

## 3.1. ALVM v1 — execution

**Status: implemented, consensus-visible.**

ALVM is a deterministic 64-bit stack machine. Values are unsigned u64.
Arithmetic overflow, division by zero, out-of-bounds memory access,
malformed stack shape, unknown opcodes, and unknown host calls all trap.
There is no floating point, clock, multithreading, filesystem, network,
JIT, or ambient process state.

### Container

A deployed program is one canonical container:

| Field | Encoding |
|---|---|
| magic | four bytes `ALVM` |
| container version | u16-le, exactly 1 |
| ISA version | u16-le, exactly 1 |
| flags | u32-le, zero in v1 |
| function count | minimal varint, 1..1024 |
| functions | descriptors, by count |
| code length | minimal varint, 1..1 MiB |
| code | instruction bytes, no trailing data |

Descriptor: `offset:u32, parameters:u16, results:u16, max_stack:u16,
reserved:u16`. Reserved is zero. Function zero is the default entry
point, with no parameters. A transaction may pick any descriptor with
zero parameters as its entry point. ABI names and source-language
metadata are outside the consensus container.

### Deploy-time validation

`al_vm_validate` and `al_vm_program_load` use caller-supplied arena
scratch memory and never allocate via malloc/free. Validation predecodes
the entire code section, then checks each function independently:

- every opcode and host number is known;
- immediates and function boundaries are complete and ordered;
- jump targets stay within the current function and land on an
  instruction;
- every reachable merge point has one identical stack height;
- every path stays within the descriptor's and genesis's stack limits;
- internal call signatures match declared pop/push effects;
- function zero terminates with STOP, RETURN, or REVERT;
- functions reached via CALL follow the frame protocol and terminate
  with RET;
- a non-zero function that no CALL points to may be a pure entry point:
  zero parameters, terminates with RETURN or REVERT, so the transaction
  layer gets return data. Mixing both protocols in one function is
  rejected — RETURN would unwind the caller's frames;
- the final instruction of every function is a terminator.

Unreachable bytes are still decoded and must be valid — this prevents
hiding a second interpretation behind control flow.

### Execution

Execution uses caller-supplied arena for the stack, linear memory, call
frames, active-address tracking, and return data. Linear memory has a
genesis limit and is zero-initialized. LOAD64 and STORE64 are explicitly
little-endian.

Internal calls are synchronous and bounded to 64 frames by default.
Contract calls are also synchronous. A child REVERT restores its own
staged state root and events, then returns status/data to the caller. Any
other child trap aborts the entire transaction. The active-address stack
rejects calling any currently active contract with `AL_ERR_REENTRANCY`.

### Host ABI

HOST carries a u16 host number. Arguments are popped from the VM stack,
results pushed in declaration order.

| ID | Operation | Stack shape |
|---:|---|---|
| 0 | write sender address to memory | (offset) -> () |
| 1 | write current address | (offset) -> () |
| 2 | block height | () -> (height) |
| 3 | protocol day | () -> (day) |
| 4 | account balance | (address_offset) -> (balance) |
| 5 | transfer from current contract | (address_offset, amount) -> () |
| 6 | storage get | (key_off,key_len,out_off,out_cap) -> (len) |
| 7 | storage put | (key_off,key_len,val_off,val_len) -> () |
| 8 | storage delete | (key_off,key_len) -> () |
| 9 | emit event | (topic_off,topic_len,data_off,data_len) -> () |
| 10 | tagged hash | (data_off,data_len,out_off,tag_id) -> () |
| 11 | signature verification | (hash_off,pubkey_off,signature_off) -> (valid) |
| 12 | call contract | (addr_off,value,in_off,in_len,out_off,out_cap) -> (status,len) |

The tagged-hash host accepts only stable `al_vm_hash_domain` IDs:
contract data, address, storage key/value, event, PoTB record,
transaction, and block. Unknown IDs trap `AL_ERR_OUT_OF_RANGE`; bytecode
cannot substitute an arbitrary domain string.

The PoTB system account is unreachable through storage hosts. Only native
consensus transitions use the system state bridge.

### Resources

Opcode and host compute cost is taken from `al_vm_resource_schedule` in
genesis. Memory is peak touched 4 KiB pages. Storage I/O is reported by
the staged state transaction. Bandwidth is charged at the transaction
level. Every addition and limit comparison uses checked integer
arithmetic.

## 3.2. ALVM v1 instruction set (ISA)

**Status: implemented.** Opcode values are permanent consensus
identifiers. All immediates are little-endian.

| Byte | Mnemonic | Immediate | Stack effect | Default cost |
|---:|---|---|---|---:|
| 00 | STOP | - | () -> () | 1 |
| 01 | PUSH64 | u64 | () -> (x) | 1 |
| 02 | ADD | - | (a,b) -> (a+b) | 3 |
| 03 | SUB | - | (a,b) -> (a-b) | 3 |
| 04 | MUL | - | (a,b) -> (a*b) | 3 |
| 05 | DIV | - | (a,b) -> (a/b) | 3 |
| 06 | EQ | - | (a,b) -> (a==b) | 3 |
| 07 | LT | - | (a,b) -> (a<b) | 3 |
| 08 | DUP | - | (a) -> (a,a) | 1 |
| 09 | DROP | - | (a) -> () | 1 |
| 0a | JUMP | u32 offset | () -> () | 2 |
| 0b | JUMPI | u32 offset | (condition) -> () | 2 |
| 0c | LOAD8 | - | (offset) -> (value) | 5 |
| 0d | STORE8 | - | (value,offset) -> () | 5 |
| 0e | RETURN | - | (offset,length) -> () | 1 |
| 0f | REVERT | - | (offset,length) -> () | 1 |
| 10 | MOD | - | (a,b) -> (a%b) | 3 |
| 11 | AND | - | (a,b) -> (a&b) | 3 |
| 12 | OR | - | (a,b) -> (a\|b) | 3 |
| 13 | XOR | - | (a,b) -> (a^b) | 3 |
| 14 | NOT | - | (a) -> (~a) | 3 |
| 15 | SHL | - | (value,count) -> (value<<count) | 3 |
| 16 | SHR | - | (value,count) -> (value>>count) | 3 |
| 17 | GT | - | (a,b) -> (a>b) | 3 |
| 18 | LE | - | (a,b) -> (a<=b) | 3 |
| 19 | GE | - | (a,b) -> (a>=b) | 3 |
| 1a | SWAP | - | (a,b) -> (b,a) | 1 |
| 1b | LOAD64 | - | (offset) -> (u64-le) | 5 |
| 1c | STORE64 | - | (value,offset) -> () | 5 |
| 1d | CALLDATA_SIZE | - | () -> (length) | 1 |
| 1e | CALLDATA_COPY | - | (source,destination,length) -> () | 5 |
| 1f | CALL | u16 function | parameters -> results | 2 |
| 20 | RET | - | results -> caller | 1 |
| 21 | HOST | u16 host ID | depends on host | 2 plus host |
| 22 | ISZERO | - | (a) -> (a==0) | 1 |
| 23 | BYTE | - | (position,value) -> (byte) | 3 |
| 24 | SIGNEXTEND | - | (bytes,value) -> (sign-extended) | 3 |
| 25 | SHA3 | - | (offset,length) -> (hash), writes a 32-byte hash to memory[offset] | 30 |
| 26 | MLOAD | - | (offset) -> (u64), 8 LE bytes from memory[offset] | 5 |
| 27 | MSTORE | - | (value,offset) -> (), 8 LE bytes to memory[offset] | 5 |
| 28 | SLOAD | - | (key_off,key_len) -> (val_len), reads from contract storage | 50 |
| 29 | SSTORE | - | (key_off,key_len,val_off,val_len) -> (), writes to contract storage | 200 |
| 2a | ADDRESS | - | (offset) -> (), writes the current contract address (32 bytes) to memory[offset] | 2 |
| 2b | CALLER | - | (offset) -> (), writes the transaction sender address (32 bytes) to memory[offset] | 2 |
| 2c | CALLVALUE | - | () -> (value), the transaction's call value | 2 |
| 2d | CODESIZE | - | () -> (size), length of the executing code | 2 |
| 2e | CODECOPY | - | (dest_off,src_off,len) -> (), copies len bytes from code[src_off] into memory[dest_off] | 5 |

### Arithmetic and control flow

ADD, SUB, and MUL trap on unsigned overflow. DIV and MOD trap on zero.
Shift counts of 64 or more yield zero. Memory accesses are bounds-checked
before any read/write. Jumps are static offsets within the current
function; there are no dynamic or cross-function jumps.

### Byte extraction and sign extension

BYTE extracts the byte at position `p` (0 = most significant) from a
64-bit value. For positions outside 0..7, the result is zero.

SIGNEXTEND treats the first argument as a byte count (0..7) for sign
extension. For `b < 8`, the sign bit is at position `b*8 + 7`. All higher
bits are set if the sign bit is set, cleared otherwise. For `b >= 8`, the
value passes through unchanged.

### Memory and storage

MLOAD and MSTORE are 8-byte (u64) loads and stores, little-endian, using
the same linear memory as LOAD8/STORE8. SLOAD and SSTORE access a
contract's paired key-value storage through the state transaction. Keys
and values are byte slices of arbitrary length, referenced by
(offset, length) pairs in memory.

### Execution context

ADDRESS writes the 32-byte current contract address to memory at the
given offset. CALLER writes the 32-byte transaction sender address.
CALLVALUE pushes the transaction's call value as a u64 onto the stack.
CODESIZE pushes the length of the executing code. CODECOPY copies a
range of executing code into memory.

### RETURN and REVERT

RETURN and REVERT unwind the entire machine, so they're only legal where
nothing above them needs resuming: in function zero, and in non-zero
functions that no CALL points to — these serve as transaction entry
points returning data (zero parameters). Everything reachable via CALL
terminates with RET. The validator statically enforces this split.

### Cost schedule

The table above is the development default schedule. Genesis carries the
cost of every opcode and host; validation rejects zero and `UINT64_MAX`
costs. Changing the schedule is a chain-identity change, not a local
setting.

## 3.3. Resource and fee model v1

**Status: implemented; production figures require benchmark
calibration.**

Astrolune measures four independent unsigned 64-bit dimensions:

| Dimension | Definition |
|---|---|
| compute | sum of genesis opcode/host costs along the executed trace |
| memory | peak touched 4 KiB pages, including synchronous child calls |
| storage | canonical key plus old/new value bytes read or written |
| bandwidth | exact size of the canonical signed transaction |

Every transaction carries a limit vector and a maximum base-price vector.
Every block carries actual aggregate usage and the price vector used for
that block. Arithmetic overflow is an execution/consensus error; counters
never saturate.

### Base price

Each resource price evolves independently of its parent's usage. The
genesis target is typically 50% of the block limit. For price P, usage U,
and target T:

```
delta = floor(P * abs(U - T) / T / 8)
```

Above target, the next price is `P + max(delta, 1)`. Below target, it's
`max(P - delta, 1)`. At target, no change. So a full block against a 50%
target moves the price by at most 12.5%, and the minimum price is one.

The actual base fee is a provable dot product of actual usage and current
prices. It's debited from the sender and burned. Transaction tips are
paid out in full after execution inclusion and split across the PoTB
60/25/15 buckets; the integer-rounding remainder goes to a flat bucket.

### Failure semantics

Encoding, size/type, chain/expiry, nonce, resource-balance bounds, and
signature are checked in that order. A pre-validation failure changes
nothing.

Once execution starts:
- success commits writes/value, increments the nonce, burns the actual
  base fee, and pays the full tip;
- REVERT or trap restores all execution writes and value transfer, but
  still increments the nonce, burns the actually-used base fee, and pays
  the full tip;
- unused resource limit is never charged;
- storage deposits are state collateral, not a fee.

### Storage deposit

Growth of a contract's storage locks `deposit_per_byte` from that
contract's balance. Shrinking or deleting a value returns the
corresponding deposit exactly, to the same contract. The reserved PoTB
system namespace is maintained by native consensus transitions and is
exempt from ordinary contract deposits.

Production block limits and prices are deliberately not hardcoded in
this specification. These are genesis values that must be chosen from
benchmark data for the minimum supported validator hardware — see
section 7 on validator requirements.
