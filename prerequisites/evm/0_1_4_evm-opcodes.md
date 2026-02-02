# EVM Opcodes: Operations & zkEVM Proving Complexity

**Date**: 2 February 2026  
**Tags**: #EVM #Opcodes #zkEVM

---

## Introduction

The Ethereum Virtual Machine executes smart contracts through a set of low-level operations called opcodes. Each opcode performs a specific operation: arithmetic, storage access, control flow, or cryptographic computation.

Understanding opcodes is essential for security auditing and optimization. But opcodes also reveal a critical insight about zkEVMs: not all operations are equal when it comes to generating zero-knowledge proofs.

This article examines EVM opcodes by category, explains their behavior, and explores why some opcodes are trivial to prove while others create significant challenges for zkEVM implementations.

If you have read the previous articles in this series, you know that storage requires complex Merkle proofs and memory needs consistency checks. Opcodes introduce another dimension: computational complexity varies dramatically based on whether operations are algebraic-friendly or require special cryptographic circuits.

---

## 1. Arithmetic & Logic Operations

### 1.1 Basic Arithmetic

**Opcodes:** `ADD`, `SUB`, `MUL`, `DIV`, `MOD`, `ADDMOD`, `MULMOD`

These opcodes perform standard arithmetic on 256-bit integers.

```
ADD:     pop a, b → push (a + b) mod 2^256
MUL:     pop a, b → push (a * b) mod 2^256
DIV:     pop a, b → push (a / b) or 0 if b == 0
MOD:     pop a, b → push (a % b) or 0 if b == 0
ADDMOD:  pop a, b, N → push (a + b) mod N
MULMOD:  pop a, b, N → push (a * b) mod N
```

**Gas costs:** 3-8 gas (cheap)

**Security notes:**
- Division by zero returns 0 (no revert at EVM level)
- **Overflow behavior at EVM level:** All arithmetic operations wrap on overflow
    - Example: `2^256 - 1 + 1 = 0` (wraps to zero)
    - Example: `0 - 1 = 2^256 - 1` (underflow wraps to max)
    - This is **EVM opcode behavior**, not a Solidity feature
    - Solidity 0.8.0+ adds **compiler-level** overflow checks (extra opcodes before arithmetic)
    - Pre-0.8.0 Solidity or inline assembly: overflow/underflow happens silently
- ADDMOD and MULMOD are cheaper than separate operations

**zkEVM perspective:**
- **Easy to prove:** Addition and multiplication are native to field arithmetic
- Circuits handle these operations efficiently
- Proving cost ≈ Gas cost (well aligned)

### 1.2 Exponentiation

**Opcode:** `EXP`

```
EXP: pop a, b → push (a^b) mod 2^256
```

**Gas cost:** 10 + 50 per byte of exponent

**zkEVM perspective:**
- **Moderate complexity:** Exponentiation requires iterative multiplication in circuits
- Not as simple as ADD/MUL but manageable
- Some zkEVMs may optimize common cases (small exponents)

### 1.3 Comparison Operations

**Opcodes:** `LT`, `GT`, `SLT`, `SGT`, `EQ`, `ISZERO`

```
LT:      pop a, b → push 1 if a < b, else 0
EQ:      pop a, b → push 1 if a == b, else 0
ISZERO:  pop a → push 1 if a == 0, else 0
```

**Gas cost:** 3 gas each

**zkEVM perspective:**
- **Range checks required:** Circuits must prove values are within valid bounds
- Comparison operations need additional constraints
- Still relatively cheap to prove

### 1.4 Bitwise Operations

**Opcodes:** `AND`, `OR`, `XOR`, `NOT`, `BYTE`, `SHL`, `SHR`, `SAR`

```
AND:  pop a, b → push (a & b)
SHL:  pop shift, value → push (value << shift)
BYTE: pop i, x → push i-th byte of x (0 if i >= 32)
```

**Gas cost:** 3 gas each

**zkEVM perspective:**
- **Depends on circuit representation:**
    - If using binary representation: efficient
    - If using field elements: may require bit decomposition (costly)
- SAR (arithmetic right shift) requires sign handling

---

## 2. Stack Operations

**Opcodes:** `POP`, `PUSH1-PUSH32`, `DUP1-DUP16`, `SWAP1-SWAP16`

### 2.1 PUSH Operations

```
PUSH1 0x42: Push 1-byte value 0x42 onto stack
PUSH32:     Push 32-byte value onto stack
```

**Gas cost:** 3 gas

PUSH operations insert immediate values from bytecode onto the stack.

### 2.2 DUP and SWAP

```
DUP1:  Duplicate 1st stack item
DUP16: Duplicate 16th stack item
SWAP1: Swap top two stack items
```

**Gas cost:** 3 gas

**zkEVM perspective:**
- **Very easy to prove:** Stack operations are simple permutations
- No computation, just data movement
- Minimal proving cost

---

## 3. Memory & Storage Operations

### 3.1 Memory Access

**Opcodes:** `MLOAD`, `MSTORE`, `MSTORE8`, `MSIZE`

```
MLOAD:   pop offset → push value at memory[offset:offset+32]
MSTORE:  pop offset, value → store value at memory[offset:offset+32]
MSTORE8: pop offset, value → store (value mod 256) at memory[offset]
MSIZE:   push size of active memory in bytes
```

**Gas cost:** 3 gas + memory expansion cost

Memory expansion follows the quadratic formula covered in Article 3.

**zkEVM perspective:**
- Covered in Article 3: Memory & ABI Encoding
- Requires consistency proofs (read-write ordering)
- Significantly simpler than storage (no Merkle proofs)

### 3.2 Storage Access

**Opcodes:** `SLOAD`, `SSTORE`

```
SLOAD:  pop key → push storage[key]
SSTORE: pop key, value → storage[key] = value
```

**Gas cost:**
- SLOAD: 2100 gas (warm) or higher (cold)
- SSTORE: 2900-20000 gas depending on state change

**zkEVM perspective:**
- Covered in Article 2: EVM Storage
- **Most complex operations for zkEVM:** Require Merkle proof generation
- Must prove value exists at specific slot in global state tree
- Proving cost >> Gas cost
- This is why zkEVMs struggle most with storage-heavy contracts

---

## 4. Control Flow

**Opcodes:** `JUMP`, `JUMPI`, `PC`, `JUMPDEST`, `STOP`, `RETURN`, `REVERT`

### 4.1 Jumps

```
JUMP:     pop dest → set PC to dest
JUMPI:    pop dest, condition → if condition != 0, set PC to dest
JUMPDEST: Mark valid jump destination (no effect on execution)
```

**Gas cost:** 8 gas (JUMP), 10 gas (JUMPI), 1 gas (JUMPDEST)

**Validity rules:**
- JUMP/JUMPI destinations must point to JUMPDEST opcodes
- Invalid jumps cause transaction revert
- Static analysis can verify most jump targets at compile time

**Security note:**
- Dynamic jumps are source of complexity in static analysis
- Modern Solidity rarely uses dynamic jumps

**zkEVM perspective:**
- **Control flow complexity:** Circuits must prove valid jump targets
- Conditional branches (JUMPI) require both paths to be evaluated in some ZK systems
- Type 4 zkEVMs may simplify control flow model

### 4.2 Execution Termination

```
STOP:   Halt execution, return empty data, mark as success
RETURN: pop offset, length → halt with success, return memory[offset:offset+length]
REVERT: pop offset, length → halt with failure, return memory[offset:offset+length], revert state
```

**Gas cost:** 0 gas (STOP), dynamic (RETURN/REVERT based on memory access)

**Key differences:**

**STOP:**
- Ends execution immediately
- No return data
- Transaction succeeds (state changes persist)
- Remaining gas refunded to caller

**RETURN:**
- Ends execution with success
- Returns data from memory (used for function return values)
- State changes persist
- Remaining gas refunded

**REVERT:**
- Ends execution with failure
- Returns data from memory (used for error messages)
- **All state changes are reverted** (storage, balance transfers, etc.)
- Gas consumed up to REVERT is NOT refunded
- Introduced in Byzantium fork to allow revert with error data

**Example:**
```solidity
function example() public {
    require(condition, "Error message");  // Uses REVERT if false
    // ... operations ...
    return value;  // Uses RETURN
}
```

---

## 5. Environmental Information

### 5.1 Transaction Context

**Opcodes:** `ADDRESS`, `ORIGIN`, `CALLER`, `CALLVALUE`, `GASPRICE`, `BASEFEE`

```
ADDRESS:   Push address of current contract
CALLER:    Push address of caller (msg.sender)
CALLVALUE: Push value sent with transaction (msg.value)
ORIGIN:    Push original transaction sender (tx.origin)
```

**Gas cost:** 2 gas each

**Security note:**
- Avoid using `ORIGIN` for authentication (can be exploited via intermediary contracts)
- Use `CALLER` instead

**zkEVM perspective:**
- **Simple:** These are public inputs or constants
- No additional proving complexity
- Already committed in transaction data

### 5.2 Calldata Access

**Opcodes:** `CALLDATALOAD`, `CALLDATASIZE`, `CALLDATACOPY`

```
CALLDATALOAD: pop i → push calldata[i:i+32]
CALLDATASIZE: push size of calldata in bytes
CALLDATACOPY: pop destOffset, offset, length → copy calldata to memory
```

**Gas cost:** 3 gas + memory expansion for CALLDATACOPY

**zkEVM perspective:**
- Calldata is part of public input
- Already committed, no additional proofs needed
- Memory copy operations follow standard memory proving

### 5.3 Code Access

**Opcodes:** `CODESIZE`, `CODECOPY`, `EXTCODESIZE`, `EXTCODECOPY`, `EXTCODEHASH`

```
CODESIZE:     Push size of current contract's code
EXTCODESIZE:  pop addr → push size of code at addr
EXTCODEHASH:  pop addr → push keccak256(code at addr)
```

**Gas cost:** 2-2600 gas depending on opcode and state access

**zkEVM perspective:**
- `EXTCODESIZE` and `EXTCODEHASH` require state tree access
- Must prove code exists at given address
- Similar complexity to storage access (Merkle proofs)

### 5.4 Block Information

**Opcodes:** `BLOCKHASH`, `COINBASE`, `TIMESTAMP`, `NUMBER`, `PREVRANDAO`, `GASLIMIT`, `CHAINID`

```
BLOCKHASH: pop blockNumber → push hash of block (0 if > 256 blocks old)
TIMESTAMP: Push block timestamp
NUMBER:    Push block number
CHAINID:   Push chain ID
```

**Gas cost:** 2-20 gas

**zkEVM perspective:**
- Block metadata are public inputs
- **BLOCKHASH exception:** Requires access to recent block hashes
    - Must prove hash exists in historical data
    - Limited to 256 most recent blocks

---

## 6. Cryptographic Operations

### 6.1 KECCAK256

**Opcode:** `KECCAK256` (formerly SHA3)

```
KECCAK256: pop offset, length → push keccak256(memory[offset:offset+length])
```

**Gas cost:** 30 + 6 per word

**zkEVM perspective:**
- **MAJOR CHALLENGE for zkEVMs**
- Keccak-256 is not algebraic-friendly
- Requires enormous circuits (thousands of constraints per hash)
- Proving cost >>> Gas cost
- This single opcode is a primary bottleneck in zkEVM performance

**Why it matters:**
- Used everywhere: storage keys, signature verification, general hashing
- Cannot be avoided in most contracts
- zkEVMs invest heavily in optimizing KECCAK circuits

### 6.2 Precompiled Contracts

Precompiles are special addresses (0x01-0x09, 0x0a) that execute native functions.

**Address 0x02: SHA256**
```
Input: arbitrary data
Output: SHA-256 hash
```

**Address 0x03: RIPEMD160**
```
Input: arbitrary data
Output: RIPEMD-160 hash
```

**zkEVM perspective:**
- Like KECCAK256, these are non-algebraic hash functions
- Expensive to prove
- Less commonly used than KECCAK256

**Address 0x01: ECRECOVER**
```
Input: hash, v, r, s (signature components)
Output: recovered address
```

**zkEVM perspective:**
- Elliptic curve operations
- Moderately expensive but more algebraic-friendly than hash functions
- Used for signature verification

**Address 0x06-0x08: Elliptic Curve Operations (bn256)**
```
0x06: bn256Add (point addition)
0x07: bn256ScalarMul (scalar multiplication)
0x08: bn256Pairing (pairing check)
```

**zkEVM perspective:**
- **More ZK-friendly** than hash precompiles
- Algebraic operations on elliptic curves
- Can be proven efficiently in some circuit systems
- Critical for ZK-based protocols

**Address 0x09: BLAKE2F**
```
Compression function for BLAKE2b
```

**Address 0x0a: Point Evaluation (EIP-4844)**
```
KZG point evaluation for blob transactions
```

---

## 7. Call Operations

**Opcodes:** `CALL`, `CALLCODE`, `DELEGATECALL`, `STATICCALL`, `CREATE`, `CREATE2`

### 7.1 Inter-Contract Calls

```
CALL: pop gas, addr, value, argsOffset, argsLength, retOffset, retLength
  → call contract at addr with value, return success (1/0)

DELEGATECALL: pop gas, addr, argsOffset, argsLength, retOffset, retLength
  → execute code from addr in current contract's context

STATICCALL: Same as CALL but reverts on state modifications
```

**Gas cost:** Complex (depends on many factors, typically 100+ base + called contract execution)

**Understanding DELEGATECALL:**

DELEGATECALL is fundamentally different from regular CALL:

**Regular CALL (Contract A calls Contract B):**
```
Execution context in B:
- msg.sender = A (the caller)
- msg.value = value sent by A
- Storage accessed = B's storage
- Code executed = B's code
- address(this) = B
```

**DELEGATECALL (Contract A delegatecalls Contract B):**
```
Execution context in A (not B!):
- msg.sender = ORIGINAL sender (preserved from A's context)
- msg.value = ORIGINAL value (preserved from A's context)
- Storage accessed = A's storage (NOT B's!)
- Code executed = B's code
- address(this) = A (NOT B!)
```

**Key insight:** DELEGATECALL says "execute B's code, but pretend you're still in A's context."

**Use case - Proxy Pattern:**
```
Proxy (storage) --DELEGATECALL--> Implementation (logic)
Result: Implementation's code runs, but modifies Proxy's storage
```

**Security notes:**
- DELEGATECALL is extremely dangerous if called on untrusted contracts
    - External code can modify YOUR storage arbitrarily
    - External code runs with YOUR permissions
- Only use with trusted implementation contracts
- Critical for upgradeable proxy patterns
- STATICCALL guarantees no state changes (safer for view functions)

**zkEVM perspective:**
- **Complex context management:** Must save/restore execution state
- Stack depth tracking (max 1024)
- Each call creates new execution context that must be proven
- Proving cost scales with call depth

### 7.2 Contract Creation

```
CREATE:  pop value, offset, length → create contract with code from memory, push new address
CREATE2: pop value, offset, length, salt → create contract with deterministic address
```

**Gas cost:** 32000 + deployment cost

**CREATE2 address:** `keccak256(0xff, sender, salt, keccak256(init_code))`

**zkEVM perspective:**
- Must prove correct address calculation
- CREATE2 requires KECCAK256 (expensive)
- Deployment modifies state tree (Merkle proofs)

### 7.3 SELFDESTRUCT

```
SELFDESTRUCT: pop recipient → destroy current contract, send balance to recipient
```

**Gas cost:** 5000 gas + possible refund

**Security note:**
- Removes contract code from blockchain
- Balance transferred to recipient (can force-send ETH)
- Being deprecated in future Ethereum upgrades

**zkEVM perspective:**
- Modifies state tree (contract removal)
- Balance transfer requires state proofs
- Some zkEVMs may restrict or modify this opcode

---

## 8. Logging

**Opcodes:** `LOG0`, `LOG1`, `LOG2`, `LOG3`, `LOG4`

```
LOG0: pop offset, length → emit log with data from memory[offset:offset+length]
LOG1: pop offset, length, topic1 → emit log with 1 indexed topic
LOG4: pop offset, length, topic1, topic2, topic3, topic4 → emit log with 4 topics
```

**Gas cost:** 375 + 375 per topic + 8 per data byte

Logs are used for events. They are not accessible from contracts (only off-chain).

**zkEVM perspective:**
- **Relatively simple:** Logs are not part of state
- No Merkle proofs required
- Just prove logs were correctly generated from memory data
- Lower proving complexity than storage operations

---

## 9. System Operations

### 9.1 Gas Metering

**Opcodes:** `GAS`, `GASLIMIT`, `GASPRICE`

```
GAS:      Push remaining gas
GASLIMIT: Push block gas limit
GASPRICE: Push gas price of transaction
```

**Gas cost:** 2 gas

**zkEVM perspective:**
- **Gas metering must be proven:** Circuit verifies sufficient gas at each step
- Cumulative gas tracking throughout execution
- Some complexity but manageable

### 9.2 Return Data

**Opcodes:** `RETURNDATASIZE`, `RETURNDATACOPY`

```
RETURNDATASIZE: Push size of data returned from last external call
RETURNDATACOPY: pop destOffset, offset, length → copy return data to memory
```

**Gas cost:** 3 gas + memory expansion

Used to access return data from external calls without ABI decoding.

---

## 10. zkEVM Implications

### 10.1 Proving Cost vs Gas Cost Divergence

The EVM gas model reflects computation cost, but not proving cost.

**Well-aligned (Gas ≈ Proving cost):**
- Arithmetic operations (ADD, MUL)
- Stack operations (PUSH, DUP, SWAP)
- Simple memory access

**Misaligned (Proving cost >> Gas cost):**
- KECCAK256: 30 gas, thousands of constraints
- Storage operations: Merkle proof complexity
- Cryptographic precompiles (SHA256, RIPEMD)

**Consequence:** Some zkEVMs adjust gas schedules to better reflect proving costs.

### 10.2 Non-Algebraic Operations

**The core challenge:** Most ZK proof systems work with field arithmetic.

**Algebraic-friendly:**
- Addition, multiplication, modular arithmetic
- Elliptic curve operations (bn256 precompiles)

**Non-algebraic (expensive to prove):**
- Hash functions (KECCAK256, SHA256, RIPEMD)
- Bitwise operations (require bit decomposition)

**Why it matters:** A single KECCAK256 can dominate proving time for entire transaction.

### 10.3 Type Differences

Different zkEVM types handle opcodes differently based on their compatibility goals.

**Type 1 (Taiko):**
- Full EVM equivalence
- All opcodes work identically to mainnet
- Highest proving cost, maximum compatibility

**Type 2-3 (Scroll, Polygon zkEVM, Linea):**
- Minor modifications to expensive operations
- Most opcodes work the same
- Balance compatibility and efficiency
- May optimize gas costs to better reflect proving costs

**Type 4 (zkSync Era):**
- Different instruction set
- Some EVM opcodes replaced or modified
- May use ZK-friendly hash functions (like Poseidon) internally for certain operations while maintaining Solidity-level compatibility
- Significant optimizations but lower EVM equivalence

### 10.4 What This Means for Developers

When deploying to zkEVMs:

1. **Minimize KECCAK256 usage**
    - Each hash significantly increases proving time
    - Consider alternatives where possible

2. **Storage access patterns matter more**
    - SLOAD/SSTORE are even more expensive
    - Batch operations when possible

3. **Test on target zkEVM**
    - Gas costs may differ from mainnet
    - Some operations disproportionately expensive
    - Performance characteristics vary

4. **Understand your zkEVM type**
    - Type 1: Deploy as-is, expect slower proving
    - Type 4: May need contract modifications

---

## Conclusion

**Opcode Categories:**
- Arithmetic & Logic: Efficient to prove, algebraic-friendly
- Stack & Memory: Simple permutations and consistency checks
- Storage: Most complex (Merkle proofs required)
- Control Flow: Moderate complexity (branch handling)
- Cryptographic: Major challenge (non-algebraic operations)
- Calls: Context management complexity
- Logging: Simple (not part of state)

**zkEVM Key Insights:**
- Not all opcodes are equal in proving complexity
- KECCAK256 and storage operations dominate proving costs
- Algebraic operations (arithmetic) prove efficiently
- Non-algebraic operations (hashing) require massive circuits
- Gas costs do not reflect proving costs

**Design Principle:**
zkEVM implementations balance EVM compatibility with proving efficiency. Understanding which opcodes are expensive to prove guides both zkEVM design and smart contract optimization for ZK environments.

### Series Progress

**1 - [EVM Architecture: Stack, Memory, Storage & zkEVM Foundations](https://github.com/Artyflex/technical-writings/blob/main/prerequisites/evm/0_1_1_evm-architecture.md)**  
**2 - [EVM Storage: Layouts, Packing & zkEVM Challenges](https://github.com/Artyflex/technical-writings/blob/main/prerequisites/evm/0_1_2_evm_storage.md)**  
**3 - [EVM Memory & ABI Encoding: Optimizations, Security & zkEVM Insights](https://github.com/Artyflex/technical-writings/blob/main/prerequisites/evm/0_1_3_evm-memory-abi-encoding.md)**  
**4 - EVM Opcodes: Operations & zkEVM Proving Complexity**

These fundamentals provide the foundation for understanding how zkEVMs work and why certain design decisions are made. Understanding opcodes reveals why some operations are trivial to prove while others create significant bottlenecks in zero-knowledge proof systems.

---

## References

- [EVM.codes](https://www.evm.codes) - Interactive opcode reference with gas costs
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) - Formal EVM specification
- [The different types of ZK-EVMs](https://vitalik.eth.limo/general/2022/08/04/zkevm.html) - Vitalik Buterin's zkEVM types classification
- [Scroll zkEVM Documentation](https://docs.scroll.io/) - Type 2 zkEVM implementation
- [zkSync Era Documentation](https://docs.zksync.io/) - Type 4 zkEVM implementation

---

*This article is part of a learning journey from Solidity security auditing to zkEVM engineering. Follow along: [GitHub](https://github.com/Artyflex/technical-writings) • [Farcaster](https://warpcast.com/artyflex)*
