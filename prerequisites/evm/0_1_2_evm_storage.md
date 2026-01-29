# EVM Storage Deep Dive: Layouts, Packing & Security

**Date**: 29 January 2026  
**Tags**: #EVM #Storage #Security #zkEVM

---

## Introduction

Storage is the most critical EVM component for three reasons: it's persistent (bugs cause permanent damage), expensive (20,000 gas for cold writes), and complex to prove in zkEVM (requires Merkle proofs). This article explores how storage works, optimization techniques, real vulnerabilities, and why storage is the hardest part of zkEVM to prove.

This is Part 2 of the EVM Internals series. Read [Part 1: EVM Architecture](https://github.com/Artyflex/technical-writings/blob/main/prerequisites/evm/0_1_1_evm-architecture.md) if you haven't already.

---

## 1. Storage Fundamentals

### 1.1 Sequential Allocation

Storage organizes data in slots, each holding 32 bytes. Variables are allocated sequentially:
```solidity
contract Example {
    uint256 a;    // slot 0
    uint256 b;    // slot 1
    address c;    // slot 2 (20 bytes, but occupies full 32-byte slot)
}
```

Each slot is identified by a number from 0 to 2^256-1.

### 1.2 Gas Costs

Storage operations are the most expensive ([EIP-2200](https://eips.ethereum.org/EIPS/eip-2200), [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929), [EIP-3529](https://eips.ethereum.org/EIPS/eip-3529)):
- **Cold SLOAD** (first read): 2,100 gas
- **Warm SLOAD** (subsequent): 100 gas
- **SSTORE zero → non-zero**: 20,000 gas (+ 2,100 if cold = 22,100 total)
- **SSTORE non-zero → non-zero**: 2,900 gas (warm)
- **SSTORE non-zero → zero**: 2,900 gas (refund mechanism significantly reduced by EIP-3529)
More details [here](https://hackmd.io/@fvictorio/gas-costs-after-berlin).

**Why so expensive?** Storage persists across transactions. Every write must update the Ethereum state trie and be replicated across all nodes.

---

## 2. Storage Packing Optimization

### 2.1 Basic Packing

Variables smaller than 32 bytes can share a slot if declared consecutively:

**Unoptimized** (3 slots, 60,000 gas):
```solidity
contract Unoptimized {
    uint8 a;      // slot 0 (uses 1 byte, wastes 31)
    uint256 b;    // slot 1 (full 32 bytes)
    uint8 c;      // slot 2 (uses 1 byte, wastes 31)
}
```

**Optimized** (2 slots, 40,000 gas):
```solidity
contract Optimized {
    uint8 a;      // slot 0, byte 0
    uint8 c;      // slot 0, byte 1 (packed!)
    uint256 b;    // slot 1
}
// Gas saved: 20,000 (33% reduction)
```

### 2.2 Packing Rules

- Variables < 32 bytes are packed if declared **consecutively**
- Compatible types: `uint8`-`uint248`, `bool` (1 byte), `address` (20 bytes)
- Any variable ≥ 32 bytes breaks packing

**Complex example**:
```solidity
contract PackingExample {
    uint128 a;    // slot 0, bytes 0-15
    uint128 b;    // slot 0, bytes 16-31 (packed)
    address c;    // slot 1, bytes 0-19
    uint96 d;     // slot 1, bytes 20-31 (packed, 20+12=32)
    bool e;       // slot 2, byte 0
    uint248 f;    // slot 2, bytes 1-31 (packed)
}
// Total: 3 slots instead of 6
```

### 2.3 Trade-offs

**Pros**:
- Significant gas savings on SSTORE operations
- Fewer storage slots = cheaper contract deployment

**Cons**:
- Read-modify-write: updating one packed variable requires reading entire slot, modifying, then writing back
- Slightly more complex bytecode (masking/shifting operations)

**In practice**: Gas savings far outweigh the small CPU cost.

---

## 3. Mappings

### 3.1 Slot Calculation

Mappings do not use sequential slots. Instead, they use hashing:
```solidity
mapping(address => uint) balances;  // declared at slot 1
```

**Slot formula**:
```
slot = keccak256(abi.encode(key, mappingSlot))
```

**Example**: For `balances[0x742d35Cc...]`:
```solidity
slot = keccak256(abi.encode(0x742d35Cc..., 1))
     = 0x1a2b3c... (deterministic hash)
```

Each key maps to a unique slot via keccak256, avoiding collisions.

### 3.2 Nested Mappings

For nested mappings, apply the formula recursively:
```solidity
mapping(address => mapping(uint => bool)) approvals;  // slot 2

// approvals[user][tokenId]:
innerSlot = keccak256(abi.encode(user, 2))
finalSlot = keccak256(abi.encode(tokenId, innerSlot))
```

### 3.3 Security: Proxy Storage Collisions

**Vulnerability pattern**: Implementation and proxy contracts using the same storage slots.

```solidity
// Proxy.sol
contract Proxy {
    address implementation;  // slot 0
    
    fallback() external payable {
        address impl = implementation;
        assembly {
            delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
        }
    }
}

// Implementation.sol
contract Logic {
    address owner;  // ALSO slot 0!
    
    function updateOwner(address newOwner) external {
        owner = newOwner;  // Writes to slot 0
    }
}
```

**Attack**: When proxy delegates to Logic, calling `updateOwner()` overwrites `implementation` address in slot 0. Attacker can set their own malicious implementation.

**Mitigation**: Use [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) standardized slots:
```solidity
// EIP-1967: Implementation slot
bytes32 private constant IMPLEMENTATION_SLOT = 
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

These slots are far from slot 0 and unlikely to collide with normal storage layouts.

---

## 4. Dynamic Arrays

### 4.1 Storage Structure

Dynamic arrays use two locations:
- **Slot N**: Array length
- **Slot keccak256(N) + index**: Array elements
```solidity
uint[] data;  // slot 0

// Storage layout:
// slot 0: length (e.g., 3)
// slot keccak256(0): data[0]
// slot keccak256(0) + 1: data[1]
// slot keccak256(0) + 2: data[2]
```

### 4.2 Push/Pop Operations
```solidity
data.push(42);
// 1. SLOAD slot 0 (read length)
// 2. SSTORE at keccak256(0) + length (write value)
// 3. SSTORE slot 0 (increment length)
// Total: 3 storage operations
```
---

## 5. Structs and Nested Structures

### 5.1 Struct Storage Layout

Structs are laid out sequentially with packing applied:
```solidity
struct User {
    uint256 id;       // slot N
    address addr;     // slot N+1, bytes 0-19
    uint96 balance;   // slot N+1, bytes 20-31 (packed!)
}

User public user;  // starts at slot 0

// Layout:
// slot 0: user.id
// slot 1: user.addr + user.balance (packed)
```

### 5.2 Optimization

**Unoptimized struct** (3 slots):
```solidity
struct Unoptimized {
    uint8 a;      // slot 0
    uint256 b;    // slot 1
    uint8 c;      // slot 2
}
```

**Optimized struct** (2 slots):
```solidity
struct Optimized {
    uint8 a;      // slot 0, byte 0
    uint8 c;      // slot 0, byte 1 (packed)
    uint256 b;    // slot 1
}
```

**Best practice**: Group types by 32-byte alignment.

### 5.3 Structs in Arrays
```solidity
User[] public users;  // slot 0

// users[0]:
baseSlot = keccak256(0)
users[0].id        → slot baseSlot
users[0].addr      → slot baseSlot + 1 (bytes 0-19)
users[0].balance   → slot baseSlot + 1 (bytes 20-31)

// users[1]:
users[1].id        → slot baseSlot + 2
users[1].addr      → slot baseSlot + 3 (bytes 0-19)
users[1].balance   → slot baseSlot + 3 (bytes 20-31)
```
---

## 6. Storage and zkEVM

### 6.1 Why Storage is the Bottleneck

Ethereum stores account state in a Merkle Patricia Trie. Every storage operation requires:

**SLOAD process**:
1. Prove slot X contains value Y
2. Generate Merkle proof from root to leaf
3. Verify proof (32-40 Keccak hash operations)

**SSTORE process**:
1. Prove old root is correct
2. Update leaf (slot X → new value)
3. Recalculate path back to root (32-40 hashes)
4. Prove new root is correct

### 6.2 Proof Complexity

**Why Keccak is expensive in ZK**:
- Keccak256 was designed for CPU efficiency, not circuit efficiency
- One Keccak hash ≈ 150,000 constraints in a ZK circuit
- Storage operation with 40 hashes = 6,000,000+ constraints

**Example**: Simple ERC20 transfer:
```solidity
balances[sender] -= amount;  // SLOAD + SSTORE = 2 Merkle proofs
balances[receiver] += amount; // SLOAD + SSTORE = 2 Merkle proofs
// Total: 4 Merkle proofs ≈ 24,000,000 constraints
```

### 6.3 zkEVM Solutions

Different zkEVM types handle storage differently:

**Type 1 (Taiko)**:
- Proves everything exactly like EVM
- Uses full Keccak for Merkle proofs
- Slowest but most compatible
- Proof time: ~30-60 minutes per block

**Type 2-3 (Scroll, Polygon zkEVM)**:
- Replace Keccak with Poseidon hash (ZK-friendly)
- Poseidon: ~100 constraints vs 150,000 for Keccak
- Trade-off: Different state root format
- Proof time: ~2-5 minutes per block

**Type 4**:
- Major modifications to storage layout
- Can skip some Merkle proofs
- Fastest but least EVM-compatible

### 6.4 State Diff Compression

**Scroll's approach**: Only prove storage changes (state diff), not full state.

Example transaction:
- Changed: 3 storage slots
- Unchanged: 1,000,000 slots

**Standard approach**: Prove entire state (expensive)
**State diff**: Only prove 3 changed slots (99.9% reduction)

### 6.5 The Future: EVM → zkEVM Transition

Vitalik Buterin has proposed [transitioning Ethereum L1 itself to use ZK proofs](https://vitalik.eth.limo/general/2024/10/23/futures4.html) (The Verge). This would require:

1. **Replace Keccak with Poseidon** for state trie
2. **Redesign storage opcodes** for proof efficiency
3. **Implement STARKs or SNARKs** at protocol level

This demonstrates how critical storage optimization is for zkEVM viability.

---

## 7. Practical Implications for zkEVM Engineers

**Design considerations**:
- Storage operations dominate proof time
- Keccak in circuits = primary bottleneck
- State diff compression = major optimization
- Trade-offs: Proof time vs EVM equivalence

**Learning path**:
- Understand Merkle Patricia Tries deeply
- Study Poseidon vs Keccak trade-offs
- Explore state diff techniques (Scroll, Polygon)
- Analyze Type 1-4 zkEVM architectures

---

## Conclusion

Storage is the most complex and critical EVM component:

**For developers**: Most expensive (2,100-20,000 gas), requires careful optimization and security review.

**For auditors**: Primary source of vulnerabilities in proxies, upgradeable contracts, and assembly code.

**For zkEVM**: Biggest bottleneck due to Merkle proof requirements. Every SLOAD/SSTORE = 32-40 Keccak hashes = millions of constraints.

**Next in this series**: Memory & ABI Encoding – temporary data, ABI vulnerabilities, and why memory is simpler to prove than storage.

---

## References

- [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) - Gas cost increases for state access
- [EIP-3529](https://eips.ethereum.org/EIPS/eip-3529) - Reduction in refunds
- [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) - Proxy storage slots
- [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153) - Transient storage opcodes
- [Solidity Storage Layout](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html) - Official documentation
- [Possible futures: The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html) - Vitalik on L1 zkEVM transition
- [The different types of ZK-EVMs](https://vitalik.eth.limo/general/2022/08/04/zkevm.html) - zkEVM classification
- [Understanding gas costs after Berlin](https://hackmd.io/@fvictorio/gas-costs-after-berlin) - Franco Victorio
---

*This article is part of a learning journey from Solidity security auditing to zkEVM engineering. Follow along: [GitHub](https://github.com/Artyflex/technical-writings) • [Farcaster](https://warpcast.com/artyflex)*