---
title: Multi-Bridge XRP Minting
type: draft
description: Extends the erc20 module to support multiple authorized XRP minters with per-minter rate limiting.
author: TBD
core_protocol_changes_required: true
---

## Abstract

The XRPL EVM sidechain currently restricts XRP minting to a single authorized address via the `owner_address` field in the erc20 module's `TokenPair` message. This limits the protocol to a single bridge provider for cross-chain XRP transfers from XRPL.

This specification introduces two independent but complementary protocol changes to support multiple bridge providers minting the same canonical XRP representation on XRPL EVM. First, the `x/erc20` module is extended to replace the single-minter authorization model with a set of approved minter addresses. Second, a mint rate limiter constrains minting velocity per bridge, bounding the damage window if a bridge is compromised and buying time for governance response.

Together, these changes enable bridge redundancy and competition while mitigating the shared-security risk through velocity-based minting controls.

## Motivation

### Problem

The XRPL EVM sidechain uses a single bridge provider (Axelar) for cross-chain XRP transfers from XRPL. The erc20 module's `TokenPair` message stores a single `owner_address` that is the only address authorized to mint XRP on the EVM side. This creates two problems:

1. **Single point of failure.** If Axelar experiences downtime or is compromised, there is no alternative path for XRP to enter XRPL EVM.
2. **Vendor lock-in.** The protocol cannot benefit from bridge competition, redundancy, or user choice without a protocol-level change.

Supporting a second bridge (Wormhole) requires that both bridges mint the **same canonical XRP asset** on XRPL EVM. The alternative — introducing bridge-specific wrapped tokens such as `axlXRP` or `wmhXRP` — would fragment liquidity, degrade user experience, and complicate application integration.

### Shared Security Risk

Allowing multiple bridges to mint a single fungible asset means the security of that asset becomes approximately as strong as the weakest authorized bridge. If any bridge is compromised, an attacker could mint unbacked XRP, inflating the canonical supply and affecting all holders regardless of which bridge they used. This is the most serious consequence of the multi-minter model and motivates the rate limiter specified in this document.

### Why Two Components in One Specification

The multi-minter extension to `x/erc20` is the minimal change required to unblock multi-bridge support. However, deploying it alone would expose the protocol to the shared security risk described above without any mitigation. The rate limiter provides security hardening by constraining minting velocity, reducing the damage window if a bridge is compromised and buying time for governance response.

Specifying both components together ensures that the multi-minter feature is adopted with a clear commitment to the security control that makes it safe.

### Non-Goals

The following are explicitly out of scope:

- **Per-bridge wrapped tokens.** This specification preserves a single canonical XRP asset on XRPL EVM.
- **Per-bridge redemption guarantees.** Once minted, XRP is fungible. Users may exit through any bridge, and the protocol does not enforce origin-based redemption.
- **Liquidity balancing between bridges.** Cross-bridge liquidity flows are a structural property of the fungible asset model and are not addressed here.
- **On-chain provenance tracking.** The system does not record which bridge originally minted a given unit of XRP in a way that affects fungibility.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Multi-Minter Authorization (`x/erc20`)

This component extends the erc20 module to support multiple authorized minter addresses per token pair, replacing the current single-owner model.

#### 1.1 Protobuf Changes

The `TokenPair` message in `cosmos.evm.erc20.v1` MUST be extended with a new field:

```protobuf
message TokenPair {
  string erc20_address = 1;
  string denom = 2;
  bool enabled = 3;
  Owner contract_owner = 4;
  string owner_address = 5 [deprecated = true];
  repeated string owner_addresses = 6;
}
```

Field 5 (`owner_address`) MUST be marked as deprecated. Field 6 (`owner_addresses`) is added as `repeated string` to hold the set of authorized minter addresses.

A new field number (6) is used rather than reusing field 5 because the wire format of `string` and `repeated string` differs in Protocol Buffers. Reusing the field number would break wire compatibility with existing encoded state.

#### 1.2 Validation Rules

The `owner_addresses` field MUST satisfy the following invariants:

- Each entry MUST be a valid Cosmos SDK account address (bech32-encoded).
- No duplicate addresses SHALL be present. Implementations MUST reject a `TokenPair` containing duplicate entries in `owner_addresses`.
- For token pairs where `contract_owner == OWNER_MODULE`, the list MUST contain at least one address.
- For token pairs where `contract_owner == OWNER_EXTERNAL`, the `owner_addresses` field MUST be empty (external ownership is not affected by this specification).

#### 1.3 State Migration

A chain upgrade handler MUST perform the following migration for every stored `TokenPair`:

1. If `contract_owner == OWNER_MODULE` and `owner_address != ""`:
   - Set `owner_addresses = [owner_address]`.
   - Clear `owner_address` to `""`.
2. If `contract_owner == OWNER_EXTERNAL` or `owner_address == ""`:
   - No migration is performed.

The migration MUST be atomic — either all token pairs are migrated successfully, or the upgrade fails and the chain halts.

#### 1.4 Minting Authorization Logic

The `MintCoins` function in the erc20 keeper currently performs the following check:

```go
contractOwnerAddr, err := sdk.AccAddressFromBech32(pair.OwnerAddress)
if !sender.Equals(contractOwnerAddr) {
    return types.ErrMinterIsNotOwner
}
```

This MUST be replaced with:

```go
authorized := false
for _, addr := range pair.OwnerAddresses {
    ownerAddr, err := sdk.AccAddressFromBech32(addr)
    if err != nil {
        continue
    }
    if sender.Equals(ownerAddr) {
        authorized = true
        break
    }
}
if !authorized {
    return types.ErrMinterIsNotOwner
}
```

The existing error type `ErrMinterIsNotOwner` MUST be preserved for backwards compatibility with clients that match on error codes.

#### 1.5 Precompile Changes

The ERC-20 precompile's `Mint` function routes to `erc20Keeper.MintCoins` and extracts the minter address from `contract.Caller()`. No changes to the precompile's external ABI are REQUIRED. The authorization change is transparent to EVM callers — the precompile continues to accept the same `mint(address,uint256)` signature.

#### 1.6 Governance Messages

Two new Cosmos SDK messages MUST be introduced to manage the minter set:

##### `MsgAddMinter`

```protobuf
message MsgAddMinter {
  string authority = 1;
  string token = 2;
  string minter_address = 3;
}
```

- `authority` MUST be the governance module account address.
- `token` is the ERC-20 contract address or Cosmos denomination identifying the token pair.
- `minter_address` is the Cosmos SDK account address (bech32-encoded) to add.
- The message MUST fail if the address is already present in `owner_addresses`.
- The message MUST fail if the token pair has `contract_owner == OWNER_EXTERNAL`.

##### `MsgRemoveMinter`

```protobuf
message MsgRemoveMinter {
  string authority = 1;
  string token = 2;
  string minter_address = 3;
}
```

- `authority` MUST be the governance module account address.
- `token` identifies the token pair.
- `minter_address` is the address to remove.
- The message MUST fail if the address is not present in `owner_addresses`.
- The message MUST fail if removing the address would leave `owner_addresses` empty for a token pair with `contract_owner == OWNER_MODULE`.

Both messages MUST emit appropriate events containing the token identifier, the affected address, and the resulting `owner_addresses` list.

#### 1.7 Ownership Transfer (Deprecated)

The existing `TransferOwnership` and `TransferOwnershipProposal` functions operate on the single `owner_address` field and are incompatible with the multi-minter model. Both functions MUST be deprecated.

- `TransferOwnership` MUST be removed. Callers SHOULD use `MsgAddMinter` and `MsgRemoveMinter` to manage the minter set.
- `TransferOwnershipProposal` MUST be removed. Governance SHOULD use `MsgAddMinter` and `MsgRemoveMinter` instead.
- `MsgTransferOwnership` MUST be rejected at the message server with an appropriate error indicating that the function is deprecated and callers should use the new minter management messages.
- The `GetOwnerAddress` helper MUST be removed. Callers that need to check ownership SHOULD iterate `owner_addresses` directly.

#### 1.8 Query Changes

The `QueryTokenPair` and `QueryTokenPairs` query responses MUST return the `owner_addresses` field. The deprecated `owner_address` field SHOULD continue to be populated with the first element of `owner_addresses` (if non-empty) for backwards compatibility with existing clients, but clients SHOULD migrate to reading `owner_addresses`.

### Mint Rate Limiting (via `MintHook`)

#### 2.1 ERC-20 Mint Hook Interface

The erc20 module MUST define and export a `MintHook` interface:

```go
type MintHook interface {
    BeforeMint(ctx sdk.Context, minter sdk.AccAddress, to sdk.AccAddress, amount math.Int, token string) error
}
```

The erc20 keeper MUST accept a list of `MintHook` implementations and call them in order:

- `BeforeMint` MUST be called **before** the bank module's `MintCoins` in the erc20 keeper's `MintCoins` function. If any hook returns an error, the mint MUST be rejected.

The rate limiting module MUST implement this interface and register itself with the erc20 keeper during app initialization. This design ensures the erc20 module has no knowledge of rate limiting logic — it simply calls its registered hooks. The hook interface also enables future security modules to add pre-mint checks without modifying the erc20 module.

If no hooks are registered (e.g., the rate limiter is not yet activated), the erc20 module MUST proceed with minting as normal (no-op hook path).

#### 2.2 Rate Limit Configuration

A per-minter rate limit configuration MUST be stored on-chain:

```protobuf
message MinterRateLimit {
  string minter_address = 1;
  string max_amount_per_window = 2;
  uint64 window_size = 3;
  uint64 bucket_size = 4;
}
```

- `minter_address` identifies the minter this rate limit applies to. This address MUST also be present in the corresponding `TokenPair.owner_addresses` (see Multi-Minter Authorization).
- `max_amount_per_window` is the maximum amount of XRP (in drops) that this minter MAY mint within the rolling window. String representation for arbitrary precision.
- `window_size` is the lookback window in blocks (M). For example, 7200 blocks for approximately 24 hours at 12-second block times.
- `bucket_size` is the granularity of accumulation in blocks (N). Minting amounts are aggregated into fixed-size block buckets.

Rate limit configurations MUST be managed via governance. Setting `max_amount_per_window = "0"` MUST act as an emergency pause, rejecting all mints for that minter.

An OPTIONAL global rate limit configuration MAY be introduced:

```protobuf
message GlobalRateLimit {
  string max_amount_per_window = 1;
  uint64 window_size = 2;
  uint64 bucket_size = 3;
}
```

If configured, the global rate limit MUST be checked in addition to per-minter limits. A mint MUST be rejected if it would exceed either the per-minter or global limit.

#### 2.3 Bucketed Accumulation

Minting amounts MUST be tracked using fixed-size block buckets:

```protobuf
message MintBucket {
  string minter_address = 1;
  uint64 bucket_start_block = 2;
  string minted_amount = 3;
}
```

- `bucket_start_block` is the first block of this bucket's range. The bucket covers blocks `[bucket_start_block, bucket_start_block + bucket_size)`.
- `minted_amount` is the cumulative amount minted by this minter during the bucket's block range.

When a mint occurs at block height `H`:

1. Compute the current bucket: `bucket_start = H - (H % bucket_size)`.
2. Add the mint amount to the current bucket's `minted_amount`.
3. If no bucket exists for this range, create one with `minted_amount = mint_amount`.

Buckets older than `window_size` blocks MUST be pruned. Pruning MAY be performed lazily during mint operations or eagerly via an `EndBlock` hook.

#### 2.4 Window Evaluation

To evaluate the rate limit at block height `H`:

1. Compute the window start: `window_start = H - window_size`.
2. Sum `minted_amount` across all buckets where `bucket_start_block >= window_start` for the given minter.
3. The resulting sum is `window_total`.

The mint MUST be rejected if `window_total + mint_amount > max_amount_per_window`.

This block-based rolling window provides:

- **Determinism.** All validators compute the same result since blocks are the native unit of time.
- **No boundary-burst risk.** Unlike a fixed window that resets entirely, the rolling window slides forward by `bucket_size` blocks, preventing an attacker from minting `2x` the limit at window boundaries.
- **Tunable granularity.** Smaller `bucket_size` values provide finer-grained enforcement at the cost of more state entries. Larger values reduce state overhead with coarser enforcement.

#### 2.5 Enforcement Order

Rate limit checks MUST be performed after multi-minter authorization. The full mint validation order is:

1. Is the sender in `owner_addresses`? (`x/erc20` authorization)
2. Is the mint within the rate limit? (rate limiter `BeforeMint` hook, if active)

#### 2.6 Governance Parameters

The following parameters MUST be configurable via governance:

| Parameter | Scope | Description |
|---|---|---|
| `max_amount_per_window` | Per-minter | Maximum mintable amount within the rolling window |
| `window_size` | Per-minter | Lookback depth in blocks |
| `bucket_size` | Per-minter | Accumulation granularity in blocks |
| `global_max_amount_per_window` | Global | Optional cap across all minters combined |
| `global_window_size` | Global | Lookback depth for the global limit |
| `global_bucket_size` | Global | Accumulation granularity for the global limit |

#### 2.7 Events

The following events MUST be emitted:

- `EventRateLimitExceeded`: Emitted when a mint is rejected due to rate limiting. MUST include `minter_address`, `requested_amount`, `window_total`, `max_amount_per_window`.
- `EventRateLimitConfigUpdated`: Emitted when a minter's rate limit configuration is changed via governance.

## Rationale

### Multi-Minter vs. Bridge Aggregator Contract

An alternative to modifying `TokenPair` would be to deploy a bridge aggregator contract that acts as the sole minter and delegates to bridge-specific logic internally. This was rejected because:

- It would require both Axelar and Wormhole to integrate with a new contract rather than using their existing minting flows.
- The Axelar minting path is already proven. Extending the authorization model is a smaller, safer change.
- An aggregator contract introduces a new single point of failure and additional smart contract risk.

### Block-Based Rolling Window for Rate Limiting

A fixed-window rate limiter (resetting at regular intervals) was considered but rejected because it allows boundary-burst attacks: an attacker can mint the full limit at the end of one window and again at the start of the next. A pure sliding window avoids this but requires per-transaction timestamp tracking.

The block-based rolling window with bucketed accumulation provides a practical middle ground: it rolls forward by `bucket_size` blocks (no full reset), uses blocks as the native time unit (deterministic across validators), and limits state overhead via aggregation into buckets.

### Extensibility via Mint Hooks

The rate limiting logic is implemented as a `MintHook` rather than being embedded directly in the erc20 module. This separation of concerns means:

- The erc20 module remains focused on token pair management and ERC-20 compatibility.
- The rate limiter can be activated, upgraded, or disabled independently of the erc20 module.
- Future security modules (e.g., compliance checks, additional security layers) can register as hooks without modifying the erc20 module.

### Two Components in One Document

The two components form a coherent security stack. Multi-minter authorization enables the feature, and the rate limiter constrains the velocity of potential exploitation. Separating them into independent specifications risks the multi-minter extension shipping without a concrete commitment to the security mitigation. Specifying them together signals to reviewers, implementers, and bridge operators that multi-minter support comes with a clearly defined security control.

### Considered and Rejected: Collateral Oracle

A third component — a Collateral Oracle (`x/collateral`) — was evaluated as a security control. The oracle would be a dedicated on-chain module with an off-chain committee that periodically attests the XRP balance locked in each bridge's door account on XRPL. Each bridge's minting would be capped at the attested locked amount, preventing any bridge from minting more XRP than it has collateral for.

After analysis, this mechanism was determined to be ineffective against total collateral extraction in a multi-bridge setting due to a circular flow attack:

1. Bridge A is compromised. The attacker mints XRP on XRPL EVM via Bridge A (within its attestation cap).
2. The attacker exits XRP through Bridge B. Bridge B releases XRP from its door account to the attacker on XRPL.
3. The attacker redeposits that XRP into Bridge A's door account on XRPL.
4. At the next attestation epoch, Bridge A's door account shows an inflated balance. The oracle faithfully reports this.
5. The attacker mints again via Bridge A (the updated attestation cap allows it).
6. By repeating this cycle, the attacker can extract the entire system collateral across all bridges.

The attestation oracle cannot distinguish between legitimate user deposits and attacker-recycled funds. Because minted XRP is fungible on XRPL EVM, cross-bridge exits are always possible, making this circular flow an inherent structural property of the multi-bridge design.

Additionally, the oracle would introduce significant complexity — a new Cosmos SDK module, an off-chain oracle committee, epoch-based consensus, quorum mechanisms, and staleness management — along with new trust assumptions (the oracle committee) and operational overhead. A rate limiter achieves the same practical security outcome (buying time for governance to respond) with far less complexity and no additional trust assumptions.

A detailed analysis of this attack vector is available in `collateral-oracle-analysis.md`.

## Backwards Compatibility

### Multi-Minter Authorization (`x/erc20`)

- **Protobuf breaking change.** The `TokenPair` message adds a new field (`owner_addresses`, field 6) and deprecates `owner_address` (field 5). Since a new field number is used, this is wire-compatible for decoders that tolerate unknown fields. However, clients that rely on `owner_address` being populated MUST be updated. As a transitional measure, `owner_address` SHOULD be populated with the first element of `owner_addresses` in query responses.
- **State migration required.** A chain upgrade handler MUST migrate all existing `TokenPair` records. The upgrade is a coordinated halt-and-restart.
- **Precompile ABI unchanged.** The ERC-20 precompile's `mint(address,uint256)` function signature does not change. EVM callers are unaffected.
- **Error codes preserved.** `ErrMinterIsNotOwner` continues to be returned for unauthorized minters.

### Mint Rate Limiting

- **New module.** The rate limiter introduces new state objects (`MinterRateLimit`, `MintBucket`) and new governance parameters. No existing interfaces are modified beyond the addition of the `MintHook` call in the erc20 keeper.
- **erc20 module change.** The erc20 keeper MUST be extended to accept and call `MintHook` implementations. This is an internal interface change that does not affect the external precompile ABI or Cosmos message interface.
- **Behavioral change.** Previously-successful mints MAY be rejected if they exceed the rate limit. Bridge operators MUST be aware of the configured rate limits.

### Activation

Each component SHOULD be gated behind a distinct chain upgrade handler with a unique upgrade name. The multi-minter extension MUST be activated first. The rate limiter CAN be activated independently but is expected to ship together with the multi-minter extension in a single upgrade for simplicity and reduced operational burden on validators.

## Test Cases

### Multi-Minter Authorization (`x/erc20`)

| # | Description | Expected Result |
|---|---|---|
| 1.1 | Single address in `owner_addresses` mints XRP | Succeeds |
| 1.2 | Two addresses in `owner_addresses`, both mint independently | Both succeed |
| 1.3 | Address not in `owner_addresses` attempts to mint | Rejected with `ErrMinterIsNotOwner` |
| 1.4 | `MsgAddMinter` adds a new address via governance | New address can mint |
| 1.5 | `MsgRemoveMinter` removes an address via governance | Removed address can no longer mint |
| 1.6 | `MsgRemoveMinter` attempts to remove the last address | Rejected (list cannot be empty for `OWNER_MODULE`) |
| 1.7 | `MsgAddMinter` with duplicate address | Rejected |
| 1.8 | State migration: existing `owner_address` migrated to `owner_addresses[0]` | Post-migration, minting works with migrated address |
| 1.9 | Token pair with `OWNER_EXTERNAL`: `owner_addresses` remains empty | No change, external pairs unaffected |

### Mint Rate Limiting

| # | Description | Expected Result |
|---|---|---|
| 2.1 | Mint within rate limit window | Succeeds |
| 2.2 | Mint that would exceed per-minter rate limit | Rejected with `EventRateLimitExceeded` |
| 2.3 | After enough blocks pass for window to roll, capacity is restored | Mint succeeds |
| 2.4 | Rate limits per minter are independent | Minter A exhausting its limit does not affect minter B |
| 2.5 | Emergency pause: `max_amount_per_window = 0` | All mints rejected |
| 2.6 | Global rate limit exceeded with per-minter limit still available | Mint rejected |
| 2.7 | Bucket pruning: old buckets are cleaned up | State does not grow unbounded |

### Cross-Component Integration

| # | Description | Expected Result |
|---|---|---|
| C.1 | Unauthorized sender, within rate limit | Rejected by `x/erc20` authorization |
| C.2 | Authorized sender, exceeds rate limit | Rejected by rate limiter |
| C.3 | Rate limiter not active: authorized sender mints without rate limit check | Succeeds (rate limit hook is no-op) |
| C.4 | Both components active, authorized sender within rate limit | Succeeds |

## Reference Implementation

Reference implementations will be provided as pull requests to the following repositories:

- **Multi-Minter (`x/erc20`):** [`xrplevm/evm`](https://github.com/xrplevm/evm) — erc20 module protobuf, keeper, precompile, and state migration changes.
- **Rate Limiting:** [`xrplevm/evm`](https://github.com/xrplevm/evm) or [`xrplevm/node`](https://github.com/xrplevm/node) — rate limit state, bucketed accumulation, mint hook implementation, and enforcement logic.

Per the XES specification process, this document cannot be promoted to a Candidate Specification until the relevant changes have been merged into the node repository's `main` branch.

## Security Considerations

### Shared Security Model (Multi-Minter Alone)

With only the multi-minter extension deployed, all authorized bridges share a single security domain. A compromise of any bridge allows the attacker to mint unbounded XRP, diluting the entire canonical supply on XRPL EVM. The security posture becomes approximately as strong as the weakest authorized bridge. This is the primary motivation for the rate limiter, which SHOULD be deployed together with the multi-minter extension.

### Cross-Bridge Liquidity Extraction

Because both bridges mint the same fungible XRP, users can deposit through one bridge and withdraw through another. This creates asymmetric pressure on bridge liabilities: one bridge's door account may be drained while the other accumulates excess XRP.

This is a structural property of the single-asset model and is explicitly a non-goal to solve in this specification. Bridge operators SHOULD monitor their door account balances and manage rebalancing off-chain. The circular flow attack described in the Rationale section ("Considered and Rejected: Collateral Oracle") demonstrates that on-chain collateral attestation cannot prevent this cross-bridge extraction.

### Rate Limit Gaming

An attacker who compromises a bridge can mint at exactly the rate limit continuously. The rate limiter does not prevent exploitation — it slows it. The value of rate limiting is buying time for governance to respond (e.g., removing the compromised minter via an emergency proposal).

This imposes a constraint on parameter selection: **the governance response time MUST be faster than the time it takes for an attacker to exhaust the rate limit and cause material damage.** Rate limit parameters SHOULD be calibrated accordingly.

### Governance Key Management

Adding and removing minters (`x/erc20`) and configuring rate limits are governance-controlled actions. The security of the entire multi-bridge system ultimately depends on the security and responsiveness of governance.

Recommendations:

- Emergency minter removal SHOULD support expedited governance proposals to minimize response time.
- Rate limit emergency pause (`max_amount_per_window = 0`) provides an immediate circuit breaker that governance can trigger.

### State Migration Risk (`x/erc20`)

The protobuf change from `owner_address` to `owner_addresses` requires a state migration during a chain upgrade. An incorrect migration could result in token pairs with empty `owner_addresses`, which would permanently lock minting for those pairs until corrected by a subsequent upgrade. Thorough migration testing on testnets is essential before mainnet deployment.

### Defense-in-Depth Evaluation Order

When both components are active, mint requests MUST be evaluated in the following order:

1. **Authorization (`x/erc20`):** Is the sender in `owner_addresses`?
2. **Rate limit:** Is `window_total + amount <= max_amount_per_window`?

A failure at either layer MUST reject the mint. If the rate limiter module is not yet activated, the rate limit check MUST be skipped (no-op). The multi-minter authorization MUST function independently regardless of whether the rate limiter is active.
