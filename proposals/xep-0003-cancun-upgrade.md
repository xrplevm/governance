# XEP-3: XRPL EVM Cancun Hardfork Upgrade

- **Author**: CapSign Inc.
- **Status**: Draft
- **Type**: Network Upgrade
- **Created**: 2025-07-17
- **Requires**: Network-wide coordination

## Abstract

This proposal outlines the upgrade of XRPL EVM to support Ethereum's Cancun hardfork features, including the critical MCOPY opcode (EIP-5656) that enables modern smart contract patterns like Diamond proxies.

## Motivation

XRPL EVM currently uses go-ethereum v1.10.26, which predates Ethereum's Cancun hardfork (March 2024). This causes:

1. **Smart Contract Failures**: Modern contracts using MCOPY fail with `EVM error InvalidJump`
2. **Developer Friction**: Diamond proxies and advanced patterns cannot deploy
3. **Ecosystem Isolation**: XRPL EVM falls behind Ethereum's feature set
4. **Security Gaps**: Missing critical improvements like EIP-6780 SELFDESTRUCT changes

## Specification

### Target Features (Cancun Hardfork)

1. **EIP-5656 - MCOPY Opcode**

   - Memory copying instruction at opcode `0x5e`
   - 10-40% gas savings for memory operations
   - Critical for Diamond proxy patterns

2. **EIP-1153 - Transient Storage**

   - `TSTORE` and `TLOAD` opcodes
   - Cheaper temporary storage within transactions

3. **EIP-4844 - Blob Transactions**

   - Shard blob transaction support
   - `BLOBHASH` and `BLOBBASEFEE` opcodes

4. **EIP-4788 - Beacon Block Root**

   - Beacon block root access in EVM

5. **EIP-6780 - SELFDESTRUCT Restriction**
   - SELFDESTRUCT only works in same transaction

### Implementation Approach

**Option A: Direct go-ethereum Upgrade (Recommended)**

```go
// Current
github.com/ethereum/go-ethereum => github.com/evmos/go-ethereum v1.10.26-evmos-rc4

// Proposed
github.com/ethereum/go-ethereum => github.com/ethereum/go-ethereum v1.13.14
```

**Option B: Updated Evmos Dependency**
Wait for evmos/evmos to upgrade their go-ethereum fork to post-Cancun versions.

### Migration Timeline

**Phase 1: Compatibility Assessment (2 weeks)**

- Audit breaking changes between v1.10.26 → v1.13.14
- Test Cosmos SDK integration compatibility
- Identify required code modifications

**Phase 2: Implementation (4 weeks)**

- Update dependencies in xrplevm/node
- Fix integration issues
- Comprehensive testing on devnet

**Phase 3: Network Upgrade (4 weeks)**

- Testnet deployment and validation
- Community testing of Cancun features
- Coordinated mainnet upgrade

## Rationale

### Why Now?

1. **Proven Stability**: Cancun features have been live on Ethereum mainnet for 4+ months
2. **Critical Need**: Diamond contracts and modern patterns are failing
3. **Ecosystem Alignment**: Maintains compatibility with Ethereum tooling

### Alternative Considered

**Custom Fork Approach**: Initially considered maintaining a CapSign fork of evmos/go-ethereum with just MCOPY backported. **Rejected** because:

- Unnecessary maintenance overhead
- Solution exists upstream
- Limits access to other Cancun improvements

## Implementation

### Technical Changes Required

1. **Dependency Updates**

   ```diff
   - github.com/ethereum/go-ethereum => github.com/evmos/go-ethereum v1.10.26-evmos-rc4
   + github.com/ethereum/go-ethereum => github.com/ethereum/go-ethereum v1.13.14
   ```

2. **Integration Fixes**

   - Address breaking changes in Cosmos SDK integration
   - Update EVM parameter handling
   - Verify JSON-RPC compatibility

3. **Testing Requirements**
   - Full test suite validation
   - Diamond contract deployment verification
   - Gas pricing validation for new opcodes

### Evidence of Need

**Successful Test on Modern EVM** (Anvil with Cancun):

```
✅ DiamondCutFacet: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
✅ DiamondLoupeFacet: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
✅ OwnableFacet: 0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
✅ Diamond: 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
```

**Current Failure on XRPL EVM**:

```
❌ Error: Internal error: EVM error InvalidJump
```

## Security Considerations

1. **Breaking Changes**: Major go-ethereum upgrade requires careful testing
2. **Consensus Safety**: Must ensure deterministic behavior across validators
3. **Gas Accounting**: New opcodes must have correct gas pricing
4. **Backward Compatibility**: Existing contracts must continue working

## Testing Plan

### Pre-Upgrade Testing

1. **Unit Tests**: Verify all new opcodes function correctly
2. **Integration Tests**: Test Diamond proxy deployment end-to-end
3. **Gas Tests**: Validate gas consumption matches Ethereum mainnet
4. **Stress Tests**: Large contract deployments with MCOPY usage

### Post-Upgrade Validation

1. **Contract Migration**: Test existing deployed contracts
2. **RPC Compatibility**: Verify JSON-RPC endpoints work correctly
3. **Performance**: Monitor gas usage and execution times

## Backwards Compatibility

- **Smart Contracts**: All existing contracts remain fully functional
- **Tooling**: Better compatibility with modern Ethereum development tools
- **APIs**: JSON-RPC interfaces maintain compatibility

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [EIP-5656: MCOPY - Memory copying instruction](https://eips.ethereum.org/EIPS/eip-5656)
- [Cancun Network Upgrade](https://github.com/ethereum/execution-specs/blob/master/network-upgrades/mainnet-upgrades/cancun.md)
- [go-ethereum Cancun Release](https://github.com/ethereum/go-ethereum/releases/tag/v1.13.8)
