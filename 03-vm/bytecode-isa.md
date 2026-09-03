# ALVM ISA v1

**Status: implemented.** Opcode values are permanent consensus identifiers.
All immediates are little-endian.

| Byte | Mnemonic | Immediate | Stack effect | Default compute |
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
| 12 | OR | - | (a,b) -> (a|b) | 3 |
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
| 1f | CALL | u16 function | params -> results | 2 |
| 20 | RET | - | results -> caller | 1 |
| 21 | HOST | u16 host ID | host-specific | 2 plus host |
| 22 | ISZERO | - | (a) -> (a==0) | 1 |
| 23 | BYTE | - | (position,value) -> (byte) | 3 |
| 24 | SIGNEXTEND | - | (bytes,value) -> (sign-extended) | 3 |
| 25 | SHA3 | - | (offset,length) -> (hash) writes 32-byte hash at memory[offset] | 30 |
| 26 | MLOAD | - | (offset) -> (u64) load 8 bytes LE from memory[offset] | 5 |
| 27 | MSTORE | - | (value,offset) -> () store 8 bytes LE to memory[offset] | 5 |
| 28 | SLOAD | - | (key_off,key_len) -> (val_len) read from contract storage | 50 |
| 29 | SSTORE | - | (key_off,key_len,val_off,val_len) -> () write to contract storage | 200 |
| 2a | ADDRESS | - | (offset) -> () write current contract address (32 bytes) to memory[offset] | 2 |
| 2b | CALLER | - | (offset) -> () write transaction sender address (32 bytes) to memory[offset] | 2 |
| 2c | CALLVALUE | - | () -> (value) push transaction call value | 2 |
| 2d | CODESIZE | - | () -> (size) push execution code length | 2 |
| 2e | CODECOPY | - | (dest_off,src_off,len) -> () copy len bytes from execution code[src_off] to memory[dest_off] | 5 |

## Arithmetic and control flow

ADD, SUB and MUL trap on unsigned overflow. DIV and MOD trap on zero. Shift
counts at least 64 produce zero. Memory accesses are bounds checked before any
read or write. Jumps are static offsets into the current function; dynamic
jumps and cross-function jumps do not exist.

## Byte extraction and sign extension

BYTE extracts the byte at position `p` (0 = most significant) from a 64-bit
value. For positions outside 0..7 the result is zero.

SIGNEXTEND treats the first argument as the number of bytes (0..7) to
sign-extend. For `b < 8` the sign bit is at position `b*8 + 7`. All upper
bits are set if the sign bit is set, cleared otherwise. For `b >= 8` the
value passes through unchanged.

## Memory and storage

MLOAD and MSTORE are 8-byte (u64) loads and stores, little-endian, using
the same linear memory as LOAD8/STORE8. SLOAD and SSTORE access the
per-contract key-value store through the state transaction. Keys and values
are arbitrary-length byte slices referenced by (offset, length) pairs in
memory.

## Execution context

ADDRESS writes the32-byte current contract address to memory at the given
offset. CALLER writes the32-byte transaction sender address. CALLVALUE
pushes the transaction call value as a u64 onto the stack. CODESIZE pushes
the length of the execution code. CODECOPY copies a range of the execution
code into memory.

## RETURN and REVERT

RETURN and REVERT unwind the whole machine, so they are legal only where
nothing above them needs resuming: in function zero, and in non-zero
functions that nothing CALLs - those serve as data-returning transaction
entrypoints (zero parameters). Everything reachable through CALL ends with
RET. The validator enforces this split statically.

## Schedule

The table is the default development schedule. Genesis carries every opcode and
host cost, and validation rejects zero or UINT64_MAX costs. A schedule change is
a chain-identity change, not a local tuning option.
