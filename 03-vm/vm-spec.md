# ALVM v1

**Status: implemented and consensus-visible.**

ALVM is a deterministic 64-bit stack machine. Values are unsigned u64.
Arithmetic overflow, division by zero, invalid memory access, bad stack shape,
unknown opcodes and unknown host calls trap. There is no floating point, clock,
threading, filesystem, network, JIT or ambient process state.

## Container

A deployed program is one canonical container:

| Field | Encoding |
|---|---|
| magic | four bytes ALVM |
| container version | u16-le, exactly 1 |
| ISA version | u16-le, exactly 1 |
| flags | u32-le, zero in v1 |
| function count | minimal varint, 1..1024 |
| functions | count descriptors |
| code length | minimal varint, 1..1 MiB |
| code | instruction bytes, no trailing data |

A descriptor is offset:u32, parameters:u16, results:u16, max_stack:u16,
reserved:u16. Reserved is zero. Function zero is the default entrypoint and has
no parameters. A transaction may select any zero-parameter descriptor as its
entrypoint. ABI names and source-language metadata are outside the consensus
container.

## Deployment validation

al_vm_validate and al_vm_program_load use caller-owned arena scratch and do not
allocate through malloc/free. Validation predecodes the complete code section,
then checks each function independently:

- every opcode and numbered host is known;
- immediates and function boundaries are complete and ordered;
- jump targets stay inside the current function and land on an instruction;
- every reachable merge has one identical stack height;
- every path stays within descriptor and genesis stack limits;
- internal call signatures match their declared pop/push effects;
- function zero terminates with STOP, RETURN or REVERT;
- functions reached through CALL follow the frame protocol and terminate
  with RET;
- a non-zero function that no CALL targets may instead be a pure
  entrypoint: zero parameters, terminating with RETURN or REVERT so the
  transaction layer receives its return data. Mixing both protocols in one
  function would let RETURN tear down a caller's frames and is rejected;
- every function's final instruction is a terminator.

Unreachable bytes are still decoded and must be valid. This prevents hiding a
second interpretation behind control flow.

## Execution

Execution uses a caller-provided arena for stack, linear memory, call frames,
active-address tracking and return data. Linear memory has a genesis limit and
is zero-initialized. LOAD64 and STORE64 are explicitly little-endian.

Internal calls are synchronous and limited to 64 frames by default. Contract
calls are also synchronous. A child REVERT restores its staged state root and
events, then returns status/data to the caller. Any other child trap terminates
the transaction. The active-address stack rejects a call to any currently
active contract with AL_ERR_REENTRANCY.

## Host ABI

HOST carries a u16 host number. Arguments are popped from the VM stack and
results are pushed in declaration order.

| ID | Operation | Stack shape |
|---:|---|---|
| 0 | write sender address to memory | (offset) -> () |
| 1 | write current address | (offset) -> () |
| 2 | block height | () -> (height) |
| 3 | protocol day | () -> (day) |
| 4 | account balance | (address_offset) -> (balance) |
| 5 | transfer from current contract | (address_offset, amount) -> () |
| 6 | storage get | (key_off,key_len,out_off,out_cap) -> (len) |
| 7 | storage set | (key_off,key_len,val_off,val_len) -> () |
| 8 | storage delete | (key_off,key_len) -> () |
| 9 | emit event | (topic_off,topic_len,data_off,data_len) -> () |
| 10 | tagged hash | (data_off,data_len,out_off,tag_id) -> () |
| 11 | verify signature | (hash_off,pubkey_off,signature_off) -> (valid) |
| 12 | contract call | (addr_off,value,in_off,in_len,out_off,out_cap) -> (status,len) |

The tagged-hash host accepts only the stable `al_vm_hash_domain` IDs: contract
data, address, storage key/value, event, PoTB record, transaction and block.
Unknown IDs trap with `AL_ERR_OUT_OF_RANGE`; bytecode cannot supply an arbitrary
domain string.

The PoTB system account is not reachable through storage hosts. Only native
consensus transitions use the state system bridge.

## Resources

Opcode and host compute costs come from al_vm_resource_schedule in genesis.
Memory is the peak number of touched 4 KiB pages. Storage I/O is reported by
the staged state transaction. Bandwidth is charged by the transaction layer.
Every addition and limit comparison uses checked integer arithmetic.
