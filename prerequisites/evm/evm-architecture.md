# Understanding EVM Architecture: Stack, Memory, Storage & The Road to zkEVM

**Date**: 28 January 2026  
**Tags**: #EVM #Ethereum #zkEVM #Architecture

---

## Introduction

The Ethereum Virtual Machine (EVM) is the execution environment for smart contracts on Ethereum. While most developers work with high-level languages like Solidity, understanding how the EVM operates under the hood is essential for three main reasons:

1. **Gas optimization** – Making informed decisions about data structures and operations
2. **Security** – Understanding where bugs can hide and how attacks work
3. **zkEVM comprehension** – Zero-knowledge EVMs prove correct execution, which requires deep knowledge of what the EVM actually does

This article explores EVM architecture fundamentals with practical examples, security considerations, and insights into why these concepts matter for zkEVM development.

---

## 1. EVM Architecture Overview

The EVM operates as a stack-based virtual machine with several data locations. Each serves a specific purpose and has different characteristics regarding persistence, cost, and use cases.

### 1.1 The Core Components

**Stack**
- Last-In-First-Out (LIFO) structure with maximum 1024 items
- Each item is 256 bits (32 bytes)
- Fastest and cheapest operations
- Used for most computational operations
- Direct access limited to top 16 items (via DUP1-DUP16, SWAP1-SWAP16)

**Memory**
- Volatile, byte-addressable storage
- Expandable during transaction execution
- Cleared after transaction completes
- Used for temporary data like return values
- Opcodes: MLOAD, MSTORE

**Storage**
- Persistent key-value store (survives between transactions)
- Each slot holds 32 bytes
- Most expensive component (2,100-20,000 gas depending on access pattern per [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929))
- Used for contract state variables
- Opcodes: SLOAD, SSTORE

**Transient Storage** ([EIP-1153](https://eips.ethereum.org/EIPS/eip-1153))
- Introduced in Dencun upgrade (March 2024)
- Behaves like storage but cleared after transaction
- Cheaper than persistent storage (TLOAD: 100 gas, TSTORE: 100 gas)
- Useful for reentrancy locks and temporary cross-contract communication
- Opcodes: TLOAD, TSTORE

**Calldata**
- Read-only data containing function arguments
- Cheaper than memory for external function parameters
- Available only during transaction execution

**Code**
- Immutable bytecode of the smart contract
- Loaded once at contract creation
- Accessed via opcodes like CODECOPY

### 1.2 How Components Interact

Consider this simple transfer function:
```solidity
function transfer(address to, uint amount) public {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    balances[msg.sender] -= amount;
    balances[to] += amount;
}
```

During execution:
- **Calldata**: Contains `to` address and `amount` (read-only input)
- **Stack**: Performs arithmetic operations (subtraction, addition, comparisons)
- **Memory**: Temporarily stores the error message string if the condition fails
- **Storage**: Reads and writes `balances` mapping (persistent state)

### 1.3 Security Perspective

Each component has different security implications:

**Stack**
- Limited to 1024 total items, preventing infinite recursion
- Direct access limited to top 16 items (DUP16, SWAP16 are the deepest operations)
- "Stack too deep" errors occur when local variables exceed this 16-item accessible limit during compilation
- Difficult to exploit directly

**Memory**
- Volatile nature means bugs only affect current transaction
- Assembly misuse can cause memory corruption
- Improper pointer management leads to reading/writing wrong locations

**Storage**
- Persistent state means bugs cause permanent damage
- Most critical for security auditing
- Common vulnerabilities: reentrancy, storage collisions, uninitialized pointers

**Transient Storage**
- Transaction-scoped persistence (middle ground between memory and storage)
- Useful for reentrancy guards (cheaper than storage-based locks)
- Still requires proper access control (not automatically secure)
- Cleared after transaction, so no cross-transaction persistence bugs

**Calldata**
- User-controlled data – never trust without validation
- Source of most input validation bugs
- Must sanitize before using in critical operations

### 1.4 zkEVM Perspective

Zero-knowledge EVMs must prove that execution happened correctly. This means proving the behavior of each component:

**Core challenge**: How do you prove a computation is correct without re-executing it?

For each EVM component:
- **Stack**: Prove each PUSH, POP, DUP, SWAP operation executed correctly
- **Memory**: Prove read/write consistency throughout execution
- **Storage**: Prove state transitions are valid (most complex - requires Merkle proofs)
- **Transient Storage**: Similar to storage but simpler (no Merkle proofs, transaction-scoped)
- **Calldata**: Prove input data matches transaction data

Different zkEVM types make different trade-offs (see [Vitalik's zkEVM types classification](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)):
- **Type 1** (e.g., Taiko): Proves everything exactly as EVM does (slowest, most compatible)
- **Type 2-4** (e.g., Scroll, Polygon zkEVM): Optimize proof generation while maintaining EVM equivalence

Understanding these components is essential because you cannot prove what you do not understand.

---

## 2. Execution Model

### 2.1 Transaction Lifecycle

When a transaction executes on the EVM:
```
External Account → Transaction → EVM
                                 ├─ Load contract bytecode
                                 ├─ Initialize stack and memory
                                 ├─ Execute opcodes sequentially
                                 ├─ Update storage (if needed)
                                 └─ Return result or revert
```

Each step is deterministic – given the same initial state and transaction, the EVM always produces the same result. This determinism is fundamental to blockchain consensus and critical for zero-knowledge proofs.

### 2.2 Gas Metering

Every operation has a gas cost. Initial costs were defined in the Ethereum Yellow Paper, but they have evolved through various EIPs. For current gas costs, refer to [EVM.codes](https://www.evm.codes/). The EVM tracks gas consumption during execution:

- Arithmetic operations: 3-5 gas (ADD, MUL, etc.)
- Memory operations: 3 gas + expansion cost
- Storage cold access: 2,100 gas ([EIP-2929](https://eips.ethereum.org/EIPS/eip-2929))
- Storage warm access: 100 gas ([EIP-2929](https://eips.ethereum.org/EIPS/eip-2929))
- Storage initialization (zero to non-zero): 20,000 gas
- Transient storage: TLOAD 100 gas, TSTORE 100 gas ([EIP-1153](https://eips.ethereum.org/EIPS/eip-1153))
- External calls: 100-25,000 gas depending on context

**Security consideration**: Out-of-gas conditions create denial-of-service vectors. Example:
```solidity
// Vulnerable to DoS
function processAll(uint[] memory items) public {
    for(uint i = 0; i < items.length; i++) {
        // Process each item
    }
}
```

If `items` is large enough, the transaction will always run out of gas. Attackers can grief the system by submitting transactions that consume all gas without completing.

**Mitigation**: Bound your loops with a maximum iteration limit when necessary.

### 2.3 Call Stack Depth

The EVM allows maximum 1024 nested external calls. Each CALL, DELEGATECALL, or STATICCALL increases the call depth by one.

Historically, this limit enabled reentrancy attacks where attackers would recursively call victim contracts before state updates completed.

**Mitigation**: Always follow the Checks-Effects-Interactions (CEI) pattern.

### 2.4 Deterministic State Machine

The EVM is deterministic: identical input always produces identical output. There is no native source of secure randomness.

Historically, developers misused `block.timestamp`, `blockhash`, and `block.difficulty` (now PREVRANDAO per [EIP-4399](https://eips.ethereum.org/EIPS/eip-4399)) as randomness sources. These are all manipulable by miners/validators and should never be used for security-critical randomness.

**Mitigation**: For secure random values, always use off-chain oracles like [Chainlink VRF](https://docs.chain.link/vrf).

**Why this matters for zkEVM**: Determinism allows zkEVM systems to replay execution and generate proofs. If the EVM were non-deterministic, generating consistent proofs would be impossible.

---

## 3. Data Locations: A Comprehensive Comparison

Understanding when to use each data location is essential for optimization, security, and zkEVM development.

### Comparison Table

| Location | Persistent | Cost | Primary Use Case | Security Risk | zkEVM Proof Complexity |
|----------|------------|------|------------------|---------------|------------------------|
| **Stack** | No | Lowest (~3 gas) | Calculations, opcodes | Low (bounded size) | Low (simple arithmetic) |
| **Memory** | No | Medium (quadratic expansion) | Temporary arrays, structs | Medium (corruption) | Medium (consistency checks) |
| **Storage** | Yes | Highest (2.1k-20k gas) | State variables | High (permanent damage) | High (Merkle proofs required) |
| **Transient Storage** | Transaction-scoped | Low (100 gas) | Temp locks, cross-call data | Medium (improper access) | Medium (simpler than storage) |
| **Calldata** | No | Lowest | Function inputs | High (user-controlled) | Low (read-only verification) |

### 3.1 Stack

The stack is used for most EVM operations:
```solidity
// This simple addition uses the stack
uint result = a + b;

// Compiles to:
// PUSH a
// PUSH b  
// ADD     → result on stack
```

Operations: PUSH (add to stack), POP (remove), DUP (duplicate), SWAP (exchange positions)

Gas cost: ~3 gas per operation, making it the cheapest data location.

**Stack depth limitation**: While the stack can hold up to 1024 items total, the EVM can only directly access the top 16 items via DUP1-DUP16 and SWAP1-SWAP16 opcodes.

**"Stack too deep" error**: This compilation error occurs when a function requires more than 16 stack slots for its local variables and intermediate values. The Solidity compiler must keep all active variables within the top 16 stack items (accessible via DUP1-DUP16 and SWAP1-SWAP16). When this limit is exceeded, compilation fails with "Stack too deep".

The exact number of local variables that triggers this error varies depending on function complexity, parameters, return values, and intermediate calculations.
Example that causes stack too deep:
```solidity
function tooManyVars() public {
    uint a; uint b; uint c; uint d;
    uint e; uint f; uint g; uint h;
    uint i; uint j; uint k; uint l;
    uint m; uint n; uint o; uint p;
    uint q; // This may cause "Stack too deep"
    // ...
}
```

**Solutions**:
- Use fewer local variables
- Use structs to pack variables
- Split function into smaller functions
- Use `via-ir` compiler pipeline (better stack management)

### 3.2 Memory

Memory is linear and byte-addressable, starting at address 0x00.

**Layout**:
- `0x00-0x3f`: Scratch space (temporary calculations)
- `0x40-0x5f`: Free memory pointer (points to next available space)
- `0x60-0x7f`: Zero slot (always contains zero)
- `0x80+`: Your allocations

Example:
```solidity
function createArray() public pure returns (uint[] memory) {
    uint[] memory arr = new uint[](3);
    arr[0] = 10;
    arr[1] = 20;
    arr[2] = 30;
    return arr;
}
```

Solidity automatically manages memory allocation by updating the free memory pointer at `0x40`.

**Security consideration**: When using assembly, you must manually update the free memory pointer:
```solidity
assembly {
    let ptr := mload(0x40)        // Get current free memory pointer
    mstore(ptr, value)            // Store value
    mstore(0x40, add(ptr, 0x20))  // Update pointer (CRITICAL)
}
```

Forgetting to update the pointer causes the next allocation to overwrite your data.

### 3.3 Storage

Storage is the most critical component for understanding smart contract behavior and security.

**Characteristics**:
- Persistent across transactions
- 2^256 slots of 32 bytes each
- Variables allocated sequentially (slot 0, 1, 2, ...)

**Gas costs** ([EIP-2929](https://eips.ethereum.org/EIPS/eip-2929), [EIP-3529](https://eips.ethereum.org/EIPS/eip-3529)):
- Cold access (first time): 2,100 gas
- Warm access (subsequent): 100 gas
- Write zero → non-zero: 20,000 gas
- Write non-zero → non-zero: 2,900 gas
- Write non-zero → zero: 2,900 gas (refund removed per EIP-3529)

Example:
```solidity
contract Example {
    uint256 a;  // slot 0
    uint256 b;  // slot 1
    address c;  // slot 2
}
```

**Security implications**:

*Storage collisions* occur in proxy patterns when implementation and proxy use the same slots:
```solidity
// Proxy contract
contract Proxy {
    address implementation;  // slot 0
    // Uses delegatecall to implementation
}

// Logic contract  
contract Logic {
    address owner;  // ALSO slot 0!
}
```

**Mitigation**: Use [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) reserved storage slots (e.g., `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc` for implementation address).

**zkEVM complexity**: Storage is the bottleneck for zkEVM proof generation.

Why? Ethereum stores state in a Merkle Patricia Trie. Every SLOAD and SSTORE requires:
1. Proving the value at a specific slot
2. Generating a Merkle proof (32-40 hash operations)
3. Verifying the proof in the circuit

Keccak hashing in zero-knowledge circuits is expensive (thousands of constraints per hash). This is why:
- Polygon zkEVM uses Poseidon hash (ZK-friendly) instead of Keccak
- Scroll implements state diff compression
- Type 1 zkEVMs (Taiko) accept slower proof generation for exact EVM equivalence
- Vitalik Buterin has proposed [transitioning Ethereum L1 itself to a zkEVM](https://vitalik.eth.limo/general/2024/10/23/futures4.html) (The Verge), which would require replacing Keccak with ZK-friendly primitives

### 3.4 Transient Storage

Transient storage, introduced in [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153) (Dencun upgrade, March 2024), provides transaction-scoped persistence.

**Characteristics**:
- Behaves like storage but cleared after transaction completes
- Uses dedicated opcodes: TLOAD (0x5c), TSTORE (0x5d)
- Much cheaper than persistent storage (100 gas for both operations)
- Shared across all calls in a transaction (including internal calls)

**Use cases**:

*Reentrancy locks* (cheaper than storage-based):
```solidity
contract ReentrancyGuard {
    // Transient storage slot for lock
    bytes32 private constant LOCK_SLOT = bytes32(uint256(keccak256("lock")) - 1);
    
    modifier nonReentrant() {
        assembly {
            if tload(LOCK_SLOT) { revert(0, 0) }
            tstore(LOCK_SLOT, 1)
        }
        _;
        assembly {
            tstore(LOCK_SLOT, 0)
        }
    }
}
```

*Cross-contract temporary data*:
```solidity
// Contract A stores data in transient storage
// Contract B (called by A) reads it
// Data automatically cleared after transaction
```

**Security consideration**: Transient storage is shared across all calls in a transaction. Proper access control is still required – it's not automatically secure just because it's temporary.

**zkEVM perspective**: Transient storage is simpler to prove than persistent storage (no Merkle proofs needed, only transaction-scoped consistency checks).

### 3.5 Calldata

Calldata contains function arguments and is read-only.

**Key advantage**: Cheaper than memory for external function parameters:
```solidity
// More expensive - copies calldata to memory
function process(uint[] memory data) public {
    // data is now in memory
}

// Cheaper - reads directly from calldata  
function process(uint[] calldata data) external {
    // data remains in calldata
}
```

**Security consideration**: Calldata is user-controlled. Always validate before using in critical operations:
```solidity
function withdraw(uint amount) external {
    require(amount <= balances[msg.sender], "Insufficient balance");
    require(amount > 0, "Invalid amount");
    // ... rest of logic
}
```

---

## 4. Why This Matters

### For Developers

Understanding EVM internals enables:
- **Gas optimization**: Choose the right data location (calldata vs memory, storage vs transient storage)
- **Debugging**: Understand what Solidity compiles to
- **Advanced patterns**: Use assembly when necessary, leverage transient storage for cross-call data

### For Security Auditors

EVM knowledge reveals:
- **Attack surfaces**: Where can attackers inject malicious data?
- **Critical paths**: Which operations modify persistent state?
- **Bug patterns**: Storage collisions, memory corruption, reentrancy

Questions to ask during audits:
- Does this proxy pattern use EIP-1967 slots?
- Is memory pointer properly updated in assembly?
- Are storage writes protected against reentrancy?
- Could transient storage be exploited for cross-contract attacks?

### For zkEVM Engineers

You cannot prove what you do not understand. zkEVM development requires:
- Deep knowledge of what each opcode does
- Understanding which operations are expensive to prove
- Recognizing trade-offs between EVM equivalence and proof efficiency

Different components have vastly different proof complexity:
- Stack operations (ADD, MUL): Easy to prove (arithmetic circuits)
- Memory operations: Medium complexity (consistency checks)
- Transient storage: Medium complexity (transaction-scoped consistency)
- Storage operations: Hard to prove (Merkle proofs required)

This series documents the journey from Solidity development to zkEVM engineering. Understanding the EVM deeply is the essential first step.

---

## Conclusion

The EVM consists of several core components:
- **Stack**: Fast, bounded (1024 items total, 16 directly accessible), used for calculations
- **Memory**: Volatile, expandable, used for temporary data
- **Storage**: Persistent, expensive, used for state
- **Transient Storage**: Transaction-scoped, cheaper than storage, useful for locks and temporary cross-contract data
- **Calldata**: Read-only input, user-controlled
- **Code**: Immutable bytecode

Each has different characteristics regarding cost, persistence, security implications, and zkEVM proof complexity.

**Next in this series**: Storage deep dive – layouts, packing, mappings, security vulnerabilities, and why Merkle proofs make storage the hardest component for zkEVM to prove.

---

## References

- [EVM.codes](https://www.evm.codes) - Interactive opcode reference
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) - Formal EVM specification
- [The different types of ZK-EVMs](https://vitalik.eth.limo/general/2022/08/04/zkevm.html) - Vitalik Buterin's zk-EVM types classification
- [Possible futures of the Ethereum protocol, part 3: The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html) - Vitalik on zkEVM for L1
- [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) - Gas cost increases for state access opcodes
- [EIP-3529](https://eips.ethereum.org/EIPS/eip-3529) - Reduction in refunds
- [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153) - Transient storage opcodes
- [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) - Proxy storage slots
- [EIP-4399](https://eips.ethereum.org/EIPS/eip-4399) - Supplant DIFFICULTY opcode with PREVRANDAO

---

*This article is part of a learning journey from Solidity security auditing to zkEVM engineering. Follow along: [GitHub](https://github.com/Artyflex/technical-writings) • [Farcaster](https://warpcast.com/artyflex)*