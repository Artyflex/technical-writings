# EVM Memory & ABI Encoding: Optimizations, Security & zkEVM Insights

**Date**: 30 January 2026  
**Tags**: #EVM #Memory #ABI #Security #zkEVM

---
## Introduction

Memory in the EVM is volatile storage that exists only during transaction execution. Unlike storage which persists on-chain and costs significant gas, memory is cheaper but comes with its own subtleties and optimization requirements.

The Application Binary Interface (ABI) defines how smart contracts encode and decode data when communicating with each other or with external applications. Understanding ABI internals is critical for both optimization and security.

This article explores memory management at a low level, examines ABI encoding mechanisms, identifies common security pitfalls, and provides insights into how zkEVMs handle these components differently from traditional EVMs.

If you have read the previous articles in this series, you know that storage is the most complex component for zkEVMs due to Merkle proof requirements. Memory, being volatile, presents different challenges that we will explore here.

---

## 1. Memory Layout

### 1.1 Memory Structure

The EVM organizes memory as a byte array with specific reserved regions:

```
Memory Layout:
┌─────────────────────────────────────────────┐
│ 0x00 - 0x1f  (32 bytes) : Scratch space    │
│ 0x20 - 0x3f  (32 bytes) : Scratch space    │
│ 0x40 - 0x5f  (32 bytes) : Free memory ptr  │
│ 0x60 - 0x7f  (32 bytes) : Zero slot        │
│ 0x80+                   : Your allocations │
└─────────────────────────────────────────────┘
```

**Scratch space (0x00 - 0x3f):** Reserved for short-term use by Solidity. Used for hashing operations and certain internal calculations. Can be overwritten at any time.

**Free memory pointer (0x40 - 0x5f):** Contains the address of the next available memory slot. This is the most important location for memory management.

**Zero slot (0x60 - 0x7f):** Always contains zero. Used as a default value for dynamic memory arrays.

**User allocations (0x80+):** Your data starts here. All Solidity memory allocations begin at 0x80.

### 1.2 The Free Memory Pointer

The free memory pointer at 0x40 is fundamental to memory management:

```solidity
function testMemoryPointer() public pure returns (uint ptr) {
    assembly {
        ptr := mload(0x40)  // Read free memory pointer
    }
    // Returns 0x80 if no prior allocations
}
```

**Key behaviors:**
- Initialized to 0x80 at the start of execution
- Solidity automatically updates it after each allocation
- In Yul/assembly, you must manage it manually
- Never decreases (memory is never "freed" during execution)

### 1.3 Why This Layout Matters

The reserved regions serve specific purposes:

1. **Scratch space allows optimization:** Solidity can use 0x00-0x3f for temporary calculations without allocation overhead
2. **Free pointer enables dynamic allocation:** Contracts can allocate variable-sized data structures
3. **Zero slot provides efficiency:** Default values cost less gas when reading from 0x60

Understanding this layout is essential when writing assembly code or debugging low-level memory issues.

---

## 2. Memory Allocation

### 2.1 Automatic Allocation (Solidity)

When you create a memory variable in Solidity, the compiler handles allocation automatically:

```solidity
function allocateMemory() public pure returns (bytes memory) {
    bytes memory data = new bytes(64);
    return data;
}
```

**What Solidity does internally:**
1. Reads the free memory pointer: `ptr = mload(0x40)` (e.g., 0x80)
2. Writes the length: `mstore(ptr, 64)`
3. Returns pointer to the data location
4. Updates free memory pointer: `mstore(0x40, add(ptr, 0x60))` (0x20 for length + 0x40 for data rounded to 32-byte word)

**Memory layout after allocation:**
```
0x40: 0xe0              (updated free pointer)
0x80: 0x40              (length = 64 bytes)
0xa0: [your 64 bytes]   (actual data)
0xe0: [next allocation] (new free memory)
```

### 2.2 Manual Allocation (Yul)

In assembly, you must manage the free memory pointer yourself:

```solidity
function manualAllocation(uint256 value) public pure returns (uint256 result) {
    assembly {
        let ptr := mload(0x40)         // 1. Get current free pointer
        mstore(ptr, value)             // 2. Write data at pointer
        result := ptr                  // 3. Return pointer to data
        mstore(0x40, add(ptr, 0x20))   // 4. Update free pointer
    }
}
```

### 2.3 Common Allocation Pattern

A safe pattern for custom memory structures:

```solidity
function safeAllocation(uint256 size) internal pure returns (uint256 ptr) {
    assembly {
        ptr := mload(0x40)
        // Round size up to nearest 32-byte word
        let newPtr := add(ptr, and(add(size, 0x1f), not(0x1f)))
        mstore(0x40, newPtr)
    }
}
```

This pattern ensures proper alignment and prevents memory overlap.

---

## 3. Memory Expansion Costs

### 3.1 The Quadratic Cost Formula

Memory expansion is not free. The gas cost grows quadratically:

```
memory_gas = (words²) / 512 + 3 × words

where words = ceil(memory_size_bytes / 32)
```

This formula means that memory becomes increasingly expensive as you allocate more.

### 3.2 Cost Examples

Let's see how costs scale:

```
Size (bytes) | Words | Gas Cost |
-------------|-------|----------|
       64    |    2  |      ~6  |
      320    |   10  |     ~30  |
    3,200    |  100  |    ~320  |
   32,000    | 1000  |  ~5,000  |
  320,000    |10000  |~225,000  |
```

### 3.3 Optimization Implications

**Key takeaways:**
1. **Small allocations are cheap:** First few KB cost almost nothing
2. **Large allocations hurt:** Beyond 10KB, costs accelerate rapidly
3. **Prefer calldata when possible:** For read-only data, avoid memory entirely

**Practical example:**

```solidity
// More expensive: copies calldata to memory
function processExpensive(uint[] memory data) public pure returns (uint sum) {
    for (uint i = 0; i < data.length; i++) {
        sum += data[i];
    }
}

// Cheaper: reads directly from calldata
function processCheap(uint[] calldata data) external pure returns (uint sum) {
    for (uint i = 0; i < data.length; i++) {
        sum += data[i];
    }
}
```

**When to use each:**
- `calldata`: External functions, read-only data, large arrays
- `memory`: Internal functions, data modification needed, small to medium arrays

### 3.4 Memory Expansion in Loops

A common pitfall is repeated memory expansion inside loops:

```solidity
// BAD: Expands memory on each iteration
function badLoop(uint n) public pure {
    for (uint i = 0; i < n; i++) {
        bytes memory temp = new bytes(1000);
        // Process temp...
    }
}

// BETTER: Allocate once, reuse
function goodLoop(uint n) public pure {
    bytes memory temp = new bytes(1000);
    for (uint i = 0; i < n; i++) {
        // Reuse temp...
    }
}
```

The quadratic cost formula means the first approach can become extremely expensive with large n.

---

## 4. ABI Encoding Fundamentals

### 4.1 Static Types

Static types have fixed sizes and are encoded directly:

```solidity
abi.encode(uint256(42), address(0x1234567890123456789012345678901234567890))
```

**Memory layout:**
```
Offset | Value
-------|-------------------------------------------------------
0x00   | 0x000000000000000000000000000000000000000000000000000000000000002a
0x20   | 0x0000000000000000000000001234567890123456789012345678901234567890
```

Each value is padded to 32 bytes. Simple and predictable.

### 4.2 Dynamic Types

Dynamic types (strings, bytes, arrays) use a two-part structure:

```solidity
abi.encode(uint256(1), bytes("hello"))
```

**Memory layout:**
```
Offset | Value                                      | Description
-------|--------------------------------------------|-----------------
0x00   | 0x0000...0001                              | uint256(1)
0x20   | 0x0000...0040                              | offset to bytes data
0x40   | 0x0000...0005                              | length of bytes
0x60   | 0x68656c6c6f000000000000000000000000...    | "hello" + padding
```

This allows variable-sized data while maintaining a predictable structure.

### 4.3 Complex Example: Array of Structs

```solidity
struct Item {
    uint256 id;
    string name;
}

abi.encode(Item[](item1, item2))
```

**Layout (simplified):**
```
0x00: offset to array data
0x20: array length (2)
0x40: item1.id
0x60: offset to item1.name
0x80: item2.id
0xa0: offset to item2.name
0xc0: length of item1.name
0xe0: item1.name data
...
```

The encoding preserves the structure while allowing dynamic components.

### 4.4 abi.encode vs abi.encodePacked

Two encoding methods with different properties:

**abi.encode (standard ABI encoding):**
```solidity
abi.encode(uint8(1), uint8(2))
// Result: 64 bytes
// [0x000...001][0x000...002]
```

**abi.encodePacked (tight packing):**
```solidity
abi.encodePacked(uint8(1), uint8(2))
// Result: 2 bytes
// [0x01][0x02]
```

**Key differences:**

| Feature           | abi.encode          | abi.encodePacked    |
|-------------------|---------------------|---------------------|
| Padding           | Yes (32 bytes)      | No padding          |
| Length prefixes   | Yes (for dynamic)   | No                  |
| Gas cost          | Higher              | Lower               |
| Decodable         | Yes                 | No                  |
| Collision-safe    | Yes                 | No (see below)      |

**When to use each:**
- `abi.encode`: Contract calls, signatures, when decoding needed
- `abi.encodePacked`: Hashing, when size matters, no decoding needed

---

## 5. ABI Security Vulnerabilities

### 5.1 Hash Collision with abi.encodePacked

The most critical security issue with ABI encoding:

```solidity
// VULNERABLE
function verifySignature(string memory a, string memory b) public pure returns (bytes32) {
    return keccak256(abi.encodePacked(a, b));
}
```

**The problem:**
```solidity
keccak256(abi.encodePacked("aa", "b"))   // = hash of "aab"
keccak256(abi.encodePacked("a", "ab"))   // = hash of "aab"  ← COLLISION!
```

Without length separators, different inputs can produce the same encoding.

**Real-world impact:**
- **Signature schemes:** Attacker can forge valid signatures
- **Merkle trees:** Different leaves could hash to same value
- **Access control:** Bypass authentication checks

**Example vulnerable pattern:**

```solidity
contract VulnerableAuth {
    mapping(bytes32 => bool) public authorized;
    
    function authorize(string memory role, string memory user) external {
        bytes32 key = keccak256(abi.encodePacked(role, user));
        authorized[key] = true;
    }
    
    function checkAuth(string memory role, string memory user) external view returns (bool) {
        bytes32 key = keccak256(abi.encodePacked(role, user));
        return authorized[key];
    }
}

// Attacker can exploit:
// authorize("admin", "user") 
// then access with ("admi", "nuser") - same hash!
```

**Fix: Use abi.encode**

```solidity
bytes32 key = keccak256(abi.encode(role, user));
// "admin" + "user" → [...0020]["admin"]["user"]
// "admi" + "nuser" → [...0019]["admi"]["nuser"]  ← Different!
```

The length prefixes prevent collisions.

### 5.2 ABI Decoding of Untrusted Data

When decoding data from external sources, assumptions can be dangerous:

```solidity
contract VulnerableDecoder {
    function process(bytes memory data) public {
        (address token, uint256 amount) = abi.decode(data, (address, uint256));
        // Direct use without validation
        IERC20(token).transfer(msg.sender, amount);
    }
}
```

**Problems:**
1. No validation that `data` is correctly encoded
2. No bounds checking on decoded values
3. Malformed data can cause unexpected behavior

**Attack vector:**
- Attacker provides crafted `data` with invalid encoding
- `abi.decode` may return unexpected values
- Could lead to unauthorized transfers or contract state corruption

**Better approach:**

```solidity
contract SafeDecoder {
    function process(bytes memory data) public {
        require(data.length == 64, "Invalid data length");
        
        (address token, uint256 amount) = abi.decode(data, (address, uint256));
        
        // Validate decoded values
        require(token != address(0), "Invalid token");
        require(amount > 0 && amount <= MAX_AMOUNT, "Invalid amount");
        require(isWhitelistedToken(token), "Token not allowed");
        
        IERC20(token).transfer(msg.sender, amount);
    }
}
```

Always validate:
- Input data length before decoding
- Decoded values against expected ranges
- Addresses against whitelists when applicable

## 6. Memory & ABI in zkEVMs

### 6.1 Why Memory is Simpler Than Storage for zkEVMs

In our previous article on storage, we saw that zkEVMs must prove Merkle tree operations, making storage the hardest component to handle. Memory is fundamentally different:

**Storage (complex for zkEVM):**
- Persistent state requires Merkle proofs
- Must prove: "this value exists at this storage slot in the global state tree"
- Requires cryptographic commitments to entire state

**Memory (simpler for zkEVM):**
- Volatile, exists only during execution
- No Merkle trees needed
- Must only prove: "this memory operation is consistent with previous operations in this execution"

This is a significant simplification, but zkEVMs still face challenges with memory.

### 6.2 What zkEVMs Must Prove for Memory

For every memory operation, the zkEVM circuit must prove correctness:

**MLOAD(addr) → value:**
```
Circuit must prove:
1. Address 'addr' is valid (within accessed range)
2. If 'addr' was previously written, return that value
3. If 'addr' was never written, return 0
4. The operation is consistent with the execution trace
```

**MSTORE(addr, value):**
```
Circuit must prove:
1. Address 'addr' is valid
2. Write operation is recorded correctly
3. Subsequent MLOAD(addr) will return 'value'
4. Memory expansion costs are calculated correctly
```

**Key difference from regular EVM:**
- Regular EVM: Just execute the operation
- zkEVM: Generate a proof that the operation was executed correctly

### 6.3 Memory Consistency Proofs

zkEVMs must prove that memory operations are consistent throughout execution. The core challenge is: how do you prove that a MLOAD returns the correct value without storing the entire memory state in the proof?

**The fundamental problem:**
```
Step 1: MSTORE(0x80, 0x42)
Step 2: MSTORE(0xa0, 0x100)
Step 3: MLOAD(0x80) → must return 0x42
Step 4: MLOAD(0xc0) → must return 0x00 (never written)
```

The circuit must prove that step 3 returns the value from step 1, not some arbitrary value.

**General proving approach:**

All zkEVMs use some form of **execution trace** that records memory operations. The circuit then proves:

1. **Read-write consistency:** Every MLOAD returns the value from the most recent MSTORE to that address
2. **Uninitialized reads:** If an address was never written, MLOAD returns 0
3. **Ordering:** Memory operations happened in the correct sequence

**Implementation techniques vary:**

zkEVM implementations use different cryptographic primitives to achieve this, such as:
- Lookup arguments (proving a value exists in a table)
- Permutation checks (proving two lists contain the same elements in different orders)
- Polynomial commitments (representing memory state as polynomials)

The specific technique depends on the proving system used (STARKs, SNARKs, Halo2, etc.), but the security guarantee is the same: memory operations are provably consistent.

**Why this matters:**

Unlike storage (which requires Merkle proofs), memory consistency can be proven more efficiently because:
- Memory is local to one execution (no global state)
- The proving system only needs to verify consistency within the current transaction
- No need for cryptographic commitments to persistent state

This is why memory is significantly cheaper to prove than storage operations in zkEVMs.

### 6.4 ABI Encoding in zkEVM Context

ABI encoding operations (abi.encode, abi.decode) are just memory operations, but with interesting zkEVM implications:

**abi.encode complexity:**
```solidity
abi.encode(uint256(a), bytes(b), uint256(c))
```

**What the circuit must prove:**
1. Calculate offsets for dynamic data (division operations)
2. Write length prefixes to correct memory locations
3. Copy data from calldata/storage to memory
4. Update free memory pointer correctly

Each step requires constraints in the circuit.

### 6.5 Learning Perspective

Understanding memory in zkEVM context reinforces key concepts:

**Memory is simpler than storage because:**
- No global state tree to maintain
- No Merkle proofs needed
- Only local consistency within execution

**But zkEVMs still must prove:**
- Every read returns correct value
- Every write is recorded correctly
- Memory expansion costs are calculated properly
- No out-of-bounds access occurs

**This teaches us:**
- zkEVM proving is about guaranteeing correctness at every step
- Even "simple" operations require careful proof construction
- Different zkEVM types make different tradeoffs
- Gas costs reflect proving complexity, not just computation

zkEVM implementations balance EVM compatibility with proving efficiency. This tradeoff drives most design decisions.

---

## Conclusion

**Memory Management:**
- Memory layout starts at 0x80, with free pointer at 0x40
- Memory expansion has quadratic cost: manageable for small allocations, expensive for large ones
- Prefer calldata for read-only data to avoid memory copies
- Manual memory management requires careful pointer updates

**ABI Encoding:**
- `abi.encode` uses standard encoding with padding and length prefixes
- `abi.encodePacked` removes padding but creates collision risks
- Always use `abi.encode` for hashing in security-critical contexts

**zkEVM Insights:**
- Memory is significantly simpler than storage for zkEVMs (no Merkle proofs)
- Circuits must prove consistency of all memory operations
- Proving costs do not always align with EVM gas costs
- Different zkEVM types make different tradeoffs between compatibility and efficiency

### Series Progress

**1 - [EVM Architecture: Stack, Memory, Storage & zkEVM Foundations](https://github.com/Artyflex/technical-writings/blob/main/prerequisites/evm/0_1_1_evm-architecture.md)**  
**2 - [EVM Storage: Layouts, Packing & zkEVM Challenges](https://github.com/Artyflex/technical-writings/blob/main/prerequisites/evm/0_1_2_evm_storage.md)**  
**3 - EVM Memory & ABI Encoding: Optimizations, Security & zkEVM Insights**  

These fundamentals provide the foundation for understanding how zkEVMs work and why certain design decisions are made. The EVM's architecture directly influences zkEVM circuit design, and understanding these low-level details is essential for building or auditing zero-knowledge proof systems.

---

## References

- [EVM.codes](https://www.evm.codes) - Interactive opcode reference
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) - Formal EVM specification, Section 9.2 (Memory Expansion)
- [Solidity ABI Specification](https://docs.soliditylang.org/en/latest/abi-spec.html) - Official ABI encoding documentation
- [EIP-712: Typed structured data hashing and signing](https://eips.ethereum.org/EIPS/eip-712) - Standard for cross-chain message signing
- [The different types of ZK-EVMs](https://vitalik.eth.limo/general/2022/08/04/zkevm.html) - Vitalik Buterin's zkEVM types classification
- [Solidity Memory Layout](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html) - Official memory layout documentation
- [OpenZeppelin SafeERC20](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20) - Safe token interaction patterns

---

*This article is part of a learning journey from Solidity security auditing to zkEVM engineering. Follow along: [GitHub](https://github.com/Artyflex/technical-writings) • [Farcaster](https://warpcast.com/artyflex)*