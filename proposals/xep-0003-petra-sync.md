# XEP-3: XRPL EVM Multi-Hardfork Upgrade (Shanghai + Cancun + Pectra)

- **Author**: CapSign Inc.
- **Status**: Draft
- **Type**: Network Upgrade
- **Created**: 2025-07-17
- **Requires**: Network-wide coordination

## Abstract

This proposal outlines a comprehensive upgrade path for XRPL EVM to bridge the gap from its current hardfork level (London/Gray Glacier, ~2022) to the latest Ethereum features (Pectra, May 2025). XRPL EVM is currently **three major hardforks behind** Ethereum mainnet, missing critical execution layer improvements from Shanghai (2023), Cancun (2024), and Pectra (2025). This proposal focuses exclusively on PoA-compatible execution layer features while respecting XRPL EVM's Cosmos SDK architecture.

## Motivation

**XRPL EVM is significantly behind Ethereum's execution layer evolution:**

- **Current**: go-ethereum v1.10.26 via evmos/go-ethereum fork (~September 2022)
- **Missing**: 3 major hardforks with 20+ execution layer improvements
- **Impact**: Critical features like MCOPY (causing Diamond proxy failures), account abstraction, BLS cryptography, and modern gas optimizations are unavailable

This creates:

1. **Smart Contract Failures**: Modern contracts fail due to missing opcodes (MCOPY, TSTORE/TLOAD)
2. **Developer Friction**: Difficult to port modern Ethereum applications
3. **Security Gaps**: Missing important security improvements from 2023-2025
4. **Innovation Barriers**: Cannot access latest DeFi/Web3 patterns

## Technical Scope - Missing Hardforks Analysis

### ✅ **Currently Supported** (London/Gray Glacier Era)

- **London** (August 2021): EIP-1559 base fees, EIP-3529 gas refund changes
- **Berlin** (April 2021): Gas cost optimizations
- **Gray Glacier** (June 2022): Difficulty bomb delay
- **Paris/Merge** (September 2022): PoS transition (irrelevant for PoA)

### ❌ **Missing: Shanghai/Capella** (April 2023)

#### Compatible Execution Layer Features:

- **EIP-3855 (PUSH0)**: ✅ New efficient opcode for pushing zero
- **EIP-3651 (Warm COINBASE)**: ✅ Gas optimization for accessing coinbase
- **EIP-3860 (Limit Initcode)**: ✅ Security improvement limiting contract size
- **EIP-6049 (Deprecate SELFDESTRUCT)**: ✅ Prepare for future removal

#### Non-Compatible (Consensus Layer):

- **EIP-4895 (Withdrawals)**: ❌ Validator withdrawals (no validators in PoA)

### ❌ **Missing: Cancun/Deneb** (March 2024)

#### Compatible Execution Layer Features:

- **EIP-5656 (MCOPY)**: ✅ **CRITICAL** - Memory copy instruction (fixes Diamond proxies!)
- **EIP-1153 (Transient Storage)**: ✅ TSTORE/TLOAD opcodes
- **EIP-4788 (Beacon Root)**: ⚠️ Beacon chain access (may need Cosmos adaptation)
- **EIP-6780 (SELFDESTRUCT)**: ✅ Security improvement

#### Data Availability Features:

- **EIP-4844 (Blobs)**: ❓ Proto-Danksharding (depends on L2 strategy)
- **EIP-7516 (BLOBBASEFEE)**: ❓ Blob gas pricing (if blobs supported)

#### Non-Compatible (Consensus Layer):

- Various validator and consensus improvements

### ❌ **Missing: Pectra** (May 2025)

#### Compatible Execution Layer Features:

- **EIP-7702 (Account Abstraction)**: ✅ EOA smart contract delegation
- **EIP-2537 (BLS12-381)**: ✅ Advanced cryptography precompiles
- **EIP-2935 (Historical Hashes)**: ✅ Extended block hash access
- **EIP-7623 (Calldata Cost)**: ✅ Gas optimization for data-heavy transactions
- **EIP-7685 (EL Requests)**: ⚠️ Execution layer requests (needs Cosmos evaluation)

#### Data Availability:

- **EIP-7691 (Blob Throughput)**: ❓ If blobs are implemented

#### Non-Compatible (Consensus Layer):

- **EIP-7251, EIP-7002, EIP-6110, EIP-7549**: All PoS-specific features

## Implementation Strategy

### **Phase 1: Shanghai Compatibility** (Q4 2025)

**Target**: Bridge 2-year execution layer gap

- EIP-3855 (PUSH0): New opcode support
- EIP-3651 (Warm COINBASE): Gas optimizations
- EIP-3860 (Limit Initcode): Security improvements
- EIP-6049 (SELFDESTRUCT warning): Deprecation preparation

### **Phase 2: Cancun Compatibility** (Q1 2026)

**Target**: Modern smart contract support

- **EIP-5656 (MCOPY)**: **HIGH PRIORITY** - Fixes Diamond proxy failures
- EIP-1153 (Transient Storage): TSTORE/TLOAD opcodes
- EIP-6780 (SELFDESTRUCT fix): Security improvements
- EIP-4844/7516 (Blobs): _If data availability strategy requires_

### **Phase 3: Pectra Compatibility** (Q2 2026)

**Target**: Cutting-edge features

- EIP-7702 (Account Abstraction): Advanced wallet functionality
- EIP-2537 (BLS12-381): Zero-knowledge and advanced crypto
- EIP-2935 (Historical Hashes): Extended block access
- EIP-7623 (Calldata optimization): Gas efficiency improvements

## Compatibility Analysis

### **Cosmos SDK Integration**

- **Execution Layer**: evmos/go-ethereum handles EVM execution independently
- **Consensus Layer**: Cosmos SDK manages PoA consensus and networking
- **Isolation**: Execution layer hardforks should not impact Cosmos consensus
- **Risk**: EIP-7685 (EL Requests) may need Cosmos-specific adaptation

### **Go-Ethereum Version Path**

- **Current**: evmos/go-ethereum v1.10.26-evmos-rc4
- **Intermediate**: Update to include Shanghai + Cancun features (~v1.13.x equivalent)
- **Target**: Full Pectra execution layer support (~v1.15.x equivalent)
- **Strategy**: Either update evmos fork or create XRPL-specific fork

## Priority Assessment

### **Immediate Critical (Phase 1)**

1. **EIP-5656 (MCOPY)**: Fixes existing smart contract failures
2. **EIP-3855 (PUSH0)**: Basic opcode support
3. **EIP-3651/3860**: Security and gas improvements

### **High Priority (Phase 2)**

1. **EIP-1153 (Transient Storage)**: Enables modern DeFi patterns
2. **EIP-6780 (SELFDESTRUCT)**: Important security fix

### **Strategic (Phase 3)**

1. **EIP-7702 (Account Abstraction)**: Next-generation wallet UX
2. **EIP-2537 (BLS)**: Advanced cryptographic applications

## Benefits by Phase

### **Phase 1 Benefits**

- **Smart Contract Compatibility**: Fix immediate deployment failures
- **Security**: Latest security improvements from 2023
- **Gas Efficiency**: Updated gas cost models

### **Phase 2 Benefits**

- **Diamond Proxy Support**: MCOPY enables complex proxy patterns
- **Modern DeFi**: Transient storage enables advanced protocols
- **Developer Experience**: Modern Solidity features work properly

### **Phase 3 Benefits**

- **Account Abstraction**: Gasless transactions, social recovery
- **Advanced Crypto**: BLS signatures, ZK applications
- **Future-Proofing**: Alignment with Ethereum's cutting edge

## Implementation Requirements

### **Technical Updates**

1. **Fork Source Decision**: Update evmos/go-ethereum vs. create XRPL-specific fork
2. **Incremental Integration**: Test each hardfork phase independently
3. **Cosmos Compatibility**: Ensure no conflicts with Cosmos SDK
4. **Network Coordination**: All nodes must upgrade for each phase

### **Testing Strategy**

1. **Isolated Hardfork Testing**: Test Shanghai, Cancun, Pectra features separately
2. **Integration Testing**: Verify Cosmos SDK compatibility at each phase
3. **Smart Contract Testing**: Validate Diamond proxy and modern contract patterns
4. **Network Stress Testing**: Ensure performance under PoA consensus

## Conclusion

This comprehensive three-hardfork upgrade path will bring XRPL EVM from its 2022 feature set to 2025 cutting-edge capabilities. By focusing on execution layer features compatible with PoA consensus, XRPL EVM can:

- **Immediately**: Fix smart contract failures (Diamond proxies) with MCOPY
- **Medium-term**: Support modern DeFi and Web3 applications
- **Long-term**: Offer account abstraction and advanced cryptography

The phased approach respects XRPL EVM's unique architecture while maximizing compatibility with the broader Ethereum ecosystem. This is not just an upgrade—it's bringing XRPL EVM into the modern era of blockchain technology.
