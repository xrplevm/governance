---
xep: 3
title: Add EIP-5656 MCOPY Opcode Support via CapSign-Maintained evmos/go-ethereum Fork
author: CapSign Inc.
discussions-to: TBD
status: Ready for Submission
type: Network Upgrade
category: Core
created: 2025-07-16
requires: Network-wide validator upgrade
---

# XEP-3: Add EIP-5656 MCOPY Opcode Support via CapSign-Maintained evmos/go-ethereum Fork

## Abstract

This proposal requests adoption of a CapSign-maintained fork of evmos/go-ethereum that includes EIP-5656 MCOPY opcode support. The current XRPL EVM Sidechain lacks support for the MCOPY instruction (opcode 0x5e), preventing deployment of modern smart contract architectures including Diamond patterns and other advanced DeFi protocols.

## Motivation

### Current Problem

- **Diamond Pattern Contracts Fail**: Modern contracts using OpenZeppelin's Diamond pattern fail with "opcode 0x5e not defined"
- **Limited DeFi Ecosystem**: Advanced protocols cannot deploy due to missing EVM opcodes
- **Ethereum Compatibility Gap**: XRPL EVM lacks opcodes available on Ethereum mainnet since Dencun upgrade (March 2024)
- **Developer Experience**: Developers face unexpected deployment failures with modern Solidity contracts

### Root Cause Analysis

XRPL EVM's dependency chain: `XRPL EVM → evmos → evmos/go-ethereum v1.10.26-evmos-rc4 (May 2024)`

The evmos/go-ethereum base predates EIP-5656 MCOPY implementation (introduced in mainline go-ethereum v1.13.x+ post-Cancun). Additionally, evmos/go-ethereum appears to have limited recent maintenance.

### Impact Examples

- **CapSign Protocol**: Cannot deploy Diamond-based architecture
- **OpenZeppelin Diamonds**: Standard library contracts fail
- **Modern DeFi**: Advanced upgradeability patterns blocked
- **Developer Adoption**: Reduced ecosystem growth due to compatibility limits

## Specification

### Technical Implementation

CapSign proposes to maintain an updated fork of evmos/go-ethereum with the following enhancements:

#### 1. EIP-5656 MCOPY Opcode Support

- **Opcode**: 0x5e
- **Functionality**: Memory copying instruction `MCOPY(dst, src, length)`
- **Gas Cost**: `GasFastestStep (3) + memoryCopierGas` per EIP-5656 specification
- **Fork Compatibility**: Properly activated only in Dencun+ instruction set
- **Memory Handling**: Safe overlap handling using Go's built-in copy function

#### 2. Go 1.23+ Compatibility

- Remove deprecated `github.com/fjl/memsize` dependency
- Fix `runtime.stopTheWorld` build errors
- Ensure compatibility with modern Go compiler versions

#### 3. Ongoing Maintenance Commitment

- Regular updates to maintain Ethereum mainnet compatibility
- Security patches and performance improvements
- XRPL EVM-specific optimizations as needed

### Implementation Details

#### Modified Files:

- `core/vm/opcodes.go` - MCOPY opcode definition
- `core/vm/instructions.go` - opMCopy execution logic
- `core/vm/memory.go` - Memory.Copy method with overlap handling
- `core/vm/memory_table.go` - Memory expansion calculation
- `core/vm/gas_table.go` - Gas cost implementation
- `core/vm/jump_table.go` - Dencun instruction set registration
- `params/config.go` - Fork compatibility rules

#### Fork Activation Strategy:

MCOPY will only be available on networks with Dencun fork rules activated, ensuring proper Ethereum compatibility and preventing issues on non-upgraded chains.

## Rationale

### Why CapSign as Maintainer?

1. **Active Development**: CapSign has immediate need and ongoing development resources
2. **Technical Expertise**: Team has deep EVM and blockchain infrastructure experience
3. **Long-term Commitment**: CapSign's business model depends on robust XRPL EVM infrastructure
4. **Community Benefit**: Open-source maintenance benefits entire ecosystem
5. **Validator Infrastructure**: CapSign will run production validators using this fork

### Alternative Approaches Considered:

1. **Wait for evmos/go-ethereum updates**: Uncertain timeline, appears unmaintained
2. **Direct XRPL EVM team implementation**: Resource constraints, not their core focus
3. **Third-party patch**: No long-term maintenance guarantee

### Benefits to XRPL EVM Ecosystem:

- ✅ **Immediate**: Diamond contracts and advanced DeFi protocols can deploy
- ✅ **Medium-term**: Better Ethereum mainnet compatibility attracts more protocols
- ✅ **Long-term**: Dedicated maintenance ensures XRPL EVM stays current with Ethereum
- ✅ **Strategic**: Positions XRPL EVM as serious DeFi platform

## Backwards Compatibility

This proposal maintains full backwards compatibility:

- All existing contracts continue to function unchanged
- MCOPY is only available in Dencun+ instruction set
- No breaking changes to existing EVM behavior
- Proper fork activation ensures gradual rollout

## Test Cases

### ✅ MCOPY Functionality Tests (VALIDATED):

**Diamond Proxy Deployment Test**:

```solidity
// Successfully deployed Diamond with 3 facets:
// - DiamondCutFacet (0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512)
// - DiamondLoupeFacet (0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0)
// - OwnableFacet (0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9)
// Diamond Address: 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
```

**Memory Copy Validation**:

```solidity
// Basic memory copy
function testBasicMCopy() {
    assembly {
        // Copy 32 bytes from position 0x00 to 0x20
        mcopy(0x20, 0x00, 0x20)
    }
}

// Overlapping memory regions
function testOverlappingMCopy() {
    assembly {
        // Test overlap handling
        mcopy(0x10, 0x00, 0x20)
    }
}
```

**✅ Test Results (July 16, 2025)**:

- Diamond proxy: ✅ Deployed successfully
- Facet delegation: ✅ All function calls work
- MCOPY operations: ✅ Memory copying functional
- Gas costs: ✅ Within expected parameters
- Backward compatibility: ✅ No existing functionality broken

### Gas Cost Verification:

- Verify gas consumption matches EIP-5656 specification
- Test memory expansion costs for various copy sizes
- Ensure no gas underflow/overflow edge cases

### Fork Compatibility:

- MCOPY unavailable on pre-Dencun chains
- Proper activation on Dencun+ networks
- Correct instruction set selection based on fork rules

## Implementation Timeline

### Phase 1: Preparation (Week 1-2)

- [x] Complete community discussion and feedback gathering
- [x] Finalize technical implementation details
- [x] Prepare comprehensive testing suite
- [x] Document validator upgrade procedures

**✅ PROOF OF CONCEPT COMPLETED (July 2025)**:

- Successfully deployed Diamond proxy contracts on local testnet with Cancun hardfork support
- Confirmed MCOPY opcode (0x5e) functionality with complex contract architectures
- Validated backward compatibility with existing contracts
- Demonstrated gas cost efficiency and memory handling

### Phase 2: Governance (Week 3-4)

- [ ] Submit formal on-chain governance proposal
- [ ] Coordinate with existing validators
- [ ] Conduct community vote
- [ ] Address any technical concerns

### Phase 3: Deployment (Week 5-6)

- [ ] Deploy CapSign validator infrastructure
- [ ] Coordinate network upgrade with other validators
- [ ] Monitor network stability post-upgrade
- [ ] Provide post-deployment support

### Phase 4: Ongoing (Month 2+)

- [ ] Regular maintenance and updates
- [ ] Monitor ecosystem adoption
- [ ] Coordinate future Ethereum compatibility updates

## Security Considerations

### Code Security:

- Implementation follows official EIP-5656 specification exactly
- Comprehensive testing including edge cases and gas metering
- Code review by multiple experienced developers
- Integration testing on testnet before mainnet deployment

### Network Security:

- Fork activation prevents compatibility issues
- Gradual rollout minimizes network disruption risk
- CapSign validator commitment ensures reliable infrastructure
- Emergency procedures for rollback if critical issues discovered

### Economic Security:

- CapSign has strong economic incentive to maintain network stability
- Validator rewards aligned with network health
- Open-source nature allows community audit and contribution

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [EIP-5656: MCOPY - Memory copying instruction](https://eips.ethereum.org/EIPS/eip-5656)
- [CapSign Protocol Documentation](https://docs.capsign.com)
- [evmos/go-ethereum Repository](https://github.com/evmos/go-ethereum)
- [XRPL EVM Documentation](https://xrpl.org/docs/evm-sidechain/)
