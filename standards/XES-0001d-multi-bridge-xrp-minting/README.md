---
title: Multi-Bridge XRP Minting
type: draft
description: Extends the erc20 module to support multiple authorized XRP minters with per-bridge composition bounds and per-minter rate limiting.
author: TBD
core_protocol_changes_required: true
---

## Abstract

The XRPL EVM sidechain currently restricts XRP minting to a single authorized address via the `owner_address` field in the erc20 module's `TokenPair` message. This limits the protocol to a single bridge provider for cross-chain XRP transfers from XRPL.

This specification introduces three independent but complementary protocol changes to support multiple bridge providers minting the same canonical XRP representation on XRPL EVM. First, the `x/erc20` module is extended to replace the single-minter authorization model with a set of approved minter addresses. Second, an bridge vault (`x/vault`) maintains per-bridge underlying balances and enforces composition bounds — a cap that prevents any one bridge from dominating the total supply, and a floor that prevents bank-run drain of healthy bridges when one is compromised. Third, a mint rate limiter constrains minting velocity per minter, bounding the damage window if a minter is compromised and buying time for governance response.

Together, these changes enable bridge redundancy and competition while mitigating the shared-security risk through both concentration-based and velocity-based minting controls.

## Motivation

### Problem

The XRPL EVM sidechain uses a single bridge provider (Axelar) for cross-chain XRP transfers from XRPL. The erc20 module's `TokenPair` message stores a single `owner_address` that is the only address authorized to mint XRP on the EVM side. This creates two problems:

1. **Single point of failure.** If Axelar experiences downtime or is compromised, there is no alternative path for XRP to enter XRPL EVM.
2. **Vendor lock-in.** The protocol cannot benefit from bridge competition, redundancy, or user choice without a protocol-level change.

Supporting a second bridge (Wormhole) requires that both bridges mint the **same canonical XRP asset** on XRPL EVM. The alternative — introducing bridge-specific wrapped tokens such as `axlXRP` or `wmhXRP` — would fragment liquidity, degrade user experience, and complicate application integration.

### Shared Security Risk

Allowing multiple bridges to mint a single fungible asset means the security of that asset becomes approximately as strong as the weakest authorized bridge. If any bridge is compromised, an attacker could mint unbacked XRP, inflating the canonical supply and affecting all holders regardless of which bridge they used. This is the most serious consequence of the multi-minter model and motivates the bridge vault and rate limiter specified in this document.

### Why Three Components in One Specification

The multi-minter extension to `x/erc20` is the minimal change required to unblock multi-bridge support. However, deploying it alone would expose the protocol to the shared security risk described above without any mitigation. The bridge vault and rate limiter provide complementary security hardening:

- The **bridge vault** caps the maximum share of total XRP supply that any single bridge may contribute, bounding the worst-case inflation from a full bridge compromise to `max_composition` of total supply. It also enforces a minimum share floor that prevents healthy bridges from being fully drained when another bridge is compromised.
- The **rate limiter** constrains minting velocity, buying time for governance to respond to an attack.

Specifying all three components together ensures that the multi-minter feature is adopted with a clear commitment to the security controls that make it safe.

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

#### 2.1 ERC-20 Hook Interfaces

The erc20 module MUST define and export the following hook interfaces:

```go
type MintHook interface {
    BeforeMint(ctx sdk.Context, minter sdk.AccAddress, to sdk.AccAddress, amount math.Int, token string) error
}

type BurnHook interface {
    BeforeBurn(ctx sdk.Context, burner sdk.AccAddress, amount math.Int, token string) error
}
```

The erc20 keeper MUST accept a list of `MintHook` and `BurnHook` implementations and call them in order:

- `BeforeMint` MUST be called **before** the bank module's `MintCoins` in the erc20 keeper's `MintCoins` function. If any hook returns an error, the mint MUST be rejected.
- `BeforeBurn` MUST be called **before** the burn is executed in the erc20 keeper's burn path. If any `BeforeBurn` hook returns an error, the burn MUST be rejected.

The rate limiting module MUST implement `MintHook`. The bridge vault module MUST implement both `MintHook` and `BurnHook`. Modules register themselves with the erc20 keeper during app initialization. This design ensures the erc20 module has no knowledge of rate limiting or bridge vault logic — it simply calls its registered hooks. The hook interfaces also enable future security modules to add pre-mint or pre-burn checks without modifying the erc20 module.

If no hooks are registered, the erc20 module MUST proceed with minting and burning as normal (no-op hook path).

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

### Bridge Vault (`x/vault`)

This component introduces an on-chain vault that aggregates multiple bridge-specific underlying balances into a single canonical XRP token. It enforces per-bridge composition bounds — a cap that prevents any one bridge from dominating the total supply, and a floor that prevents a bank-run drain from healthy underlyings when one bridge is compromised.

The bridge vault MUST implement the `MintHook` interface (for composition cap enforcement) and the `BurnHook` interface (for composition floor enforcement).

#### 3.1 Bridge Registration

A bridge registration links a minter address to its composition bounds:

```protobuf
message RegisteredBridge {
  string minter_address  = 1;  // EVM address of the bridge minter; looked up via contract.Caller()
  string max_composition = 2;  // decimal in [0, 1], e.g. "0.60"
  string min_composition = 3;  // decimal in [0, 1], e.g. "0.20"
}
```

- `minter_address` is the EVM address that the bridge uses to call `mint(to, amount)` and `burn(from, amount)` on the XRP ERC-20 contract. The vault identifies the calling bridge by matching `contract.Caller()` against this field. This address MUST also be present in the corresponding `TokenPair.owner_addresses`.
- `max_composition` is the maximum fraction of the vault's total underlying that this bridge may contribute. A mint that would push a bridge's share above `max_composition` MUST be rejected.
- `min_composition` is the minimum fraction of the vault's total underlying that this bridge must retain after a burn. A burn that would push a bridge's share below `min_composition` MUST be rejected.

Bridges MUST be keyed by `minter_address` for O(1) lookup at mint/burn time.

##### `MsgRegisterBridge`

```protobuf
message MsgRegisterBridge {
  string authority       = 1;
  string minter_address  = 2;
  string max_composition = 3;
  string min_composition = 4;
}
```

- `authority` MUST be the governance module account address.
- The message MUST fail if a bridge with the same `minter_address` already exists.
- The message MUST fail if `minter_address` is not present in any `TokenPair.owner_addresses`.
- The message MUST fail if `max_composition` or `min_composition` cannot be parsed as a decimal in `[0, 1]`.
- The message MUST fail if `min_composition >= max_composition`.
- A new `RegisteredBridge` and an empty `BridgeUnderlying` entry MUST be created.

##### `MsgUnregisterBridge`

```protobuf
message MsgUnregisterBridge {
  string authority       = 1;
  string minter_address  = 2;
}
```

- `authority` MUST be the governance module account address.
- The message MUST fail if no bridge with `minter_address` exists.
- The message MUST fail if the bridge's `BridgeUnderlying.balance` is non-zero (remaining balance must be drained before removal).
- The `RegisteredBridge` and its `BridgeUnderlying` entry MUST be deleted.

##### `MsgUpdateBridgeComposition`

```protobuf
message MsgUpdateBridgeComposition {
  string authority       = 1;
  string minter_address  = 2;
  string max_composition = 3;
  string min_composition = 4;
}
```

- Allows governance to tighten or relax composition bounds for a registered bridge without removing it.
- The same validation rules as `MsgRegisterBridge` apply to the updated bounds.

#### 3.2 Vault State

The bridge vault maintains one `BridgeUnderlying` record per registered bridge and a global vault summary:

```protobuf
message BridgeUnderlying {
  string minter_address = 1;
  string balance        = 2;  // cumulative net credits for this bridge (minted minus burned)
}

message BridgeVault {
  repeated BridgeUnderlying underlyings    = 1;
  string                   total_minted   = 2;  // total native XRP outstanding via the vault
}
```

- `BridgeUnderlying.balance` represents the amount of native XRP that was minted into the vault by this specific bridge and has not yet been burned through this bridge.
- `BridgeVault.total_minted` is the sum of all `BridgeUnderlying.balance` values. It MUST be kept consistent with the sum of individual balances at all times.

`BridgeUnderlying` records MUST be keyed by `minter_address`. The single `BridgeVault` record holds aggregated totals.

#### 3.3 Composition Cap Enforcement (BeforeMint Hook)

The `x/vault` module's `BeforeMint` implementation MUST perform the following checks:

1. Look up the caller address (`contract.Caller()`) in the registered bridge set.
2. If the caller is not a registered bridge, the hook MUST return `nil` (pass-through; multi-minter authorization still applies, but vault enforcement does not).
3. Let `B` = `BridgeUnderlying.balance` for this bridge, `T` = `BridgeVault.total_minted`, and `A` = `amount`.
4. Compute the projected composition after the mint:
   ```
   projected_composition = (B + A) / (T + A)
   ```
5. If `projected_composition > max_composition` for this bridge, the mint MUST be rejected with `ErrCompositionCapExceeded`.
6. Credit `amount` to `BridgeUnderlying.balance` and increment `BridgeVault.total_minted` by `amount`.
7. Allow the native XRP mint to the recipient.

The integer arithmetic for the composition check MUST be performed with sufficient precision to avoid rounding errors that could allow a bypass. Implementations SHOULD use `sdk.Dec` or equivalent fixed-point arithmetic.

Special case: if `T + A == 0` (first mint into an empty vault), the check MUST be skipped since the composition is trivially `1.0` and would always fail regardless of `max_composition`. Implementations MUST allow the first mint per bridge unconditionally as long as `T == 0`.

The error type MUST be:

- `ErrCompositionCapExceeded`: The mint would push this bridge's share above `max_composition`.

#### 3.4 Composition Floor Enforcement (BeforeBurn Hook)

The `x/vault` module's `BeforeBurn` implementation MUST perform the following checks:

1. Look up the caller address (`contract.Caller()`) in the registered bridge set.
2. If the caller is not a registered bridge, the hook MUST return `nil`.
3. Let `B` = `BridgeUnderlying.balance` for this bridge, `T` = `BridgeVault.total_minted`, and `A` = `amount`.
4. If `B < A`, the burn MUST be rejected with `ErrInsufficientBridgeBalance` (the bridge cannot burn more than it has credited).
5. Compute the projected composition after the burn:
   ```
   projected_composition = (B - A) / (T - A)
   ```
6. If `projected_composition < min_composition` for this bridge, the burn MUST be rejected with `ErrCompositionFloorReached`.
7. Debit `amount` from `BridgeUnderlying.balance` and decrement `BridgeVault.total_minted` by `amount`.
8. Allow the native XRP burn from the sender.

Special case: if `T - A == 0` (the vault would become empty after this burn), the floor check MUST be skipped since there is no remaining supply to compute a ratio against.

The error types MUST be:

- `ErrInsufficientBridgeBalance`: The bridge's credited balance is less than the requested burn amount.
- `ErrCompositionFloorReached`: The burn would push this bridge's share below `min_composition`.

#### 3.5 Governance Parameters

```protobuf
message VaultParams {
  bool   enabled = 1;  // global kill switch; if false, both hooks are no-ops
}
```

- `enabled`: If `false`, the bridge vault hooks MUST be no-ops, allowing minting and burning to proceed without composition enforcement. This provides an emergency circuit breaker in case of a vault implementation bug. Defaults to `true`.

#### 3.6 Events

The following events MUST be emitted by the `x/vault` module:

- `EventBridgeRegistered`: Emitted on `MsgRegisterBridge`. MUST include `minter_address`, `max_composition`, `min_composition`.
- `EventBridgeUnregistered`: Emitted on `MsgUnregisterBridge`. MUST include `minter_address`.
- `EventBridgeCompositionUpdated`: Emitted on `MsgUpdateBridgeComposition`. MUST include `minter_address`, `max_composition`, `min_composition`.
- `EventCompositionCapExceeded`: Emitted when a mint is rejected. MUST include `minter_address`, `requested_amount`, `current_balance`, `total_minted`, `max_composition`.
- `EventCompositionFloorReached`: Emitted when a burn is rejected. MUST include `minter_address`, `requested_amount`, `current_balance`, `total_minted`, `min_composition`.
- `EventVaultMint`: Emitted on a successful vault mint. MUST include `minter_address`, `amount`, `new_balance`, `new_total`.
- `EventVaultBurn`: Emitted on a successful vault burn. MUST include `minter_address`, `amount`, `new_balance`, `new_total`.

### Enforcement Order

When all three components are active, mint requests MUST be evaluated in the following order:

1. Is the sender in `owner_addresses`? (`x/erc20` authorization)
2. Is the mint within the composition cap? (`x/vault` `BeforeMint` hook, if active)
3. Is the mint within the rate limit? (rate limiter `BeforeMint` hook, if active)

For burn requests:

1. Is the burn within the composition floor? (`x/vault` `BeforeBurn` hook, if active)

A failure at any layer MUST reject the mint or burn. If the bridge vault or rate limiter modules are not yet activated, the corresponding checks MUST be skipped (no-op). The multi-minter authorization MUST function independently regardless of whether the other components are active.

## Rationale

### Multi-Minter vs. Bridge Aggregator Contract

An alternative to modifying `TokenPair` would be to deploy a bridge aggregator contract that acts as the sole minter and delegates to bridge-specific logic internally. This was rejected because:

- It would require both Axelar and Wormhole to integrate with a new contract rather than using their existing minting flows.
- The Axelar minting path is already proven. Extending the authorization model is a smaller, safer change.
- An aggregator contract introduces a new single point of failure and additional smart contract risk.

### Block-Based Rolling Window for Rate Limiting

A fixed-window rate limiter (resetting at regular intervals) was considered but rejected because it allows boundary-burst attacks: an attacker can mint the full limit at the end of one window and again at the start of the next. A pure sliding window avoids this but requires per-transaction timestamp tracking.

The block-based rolling window with bucketed accumulation provides a practical middle ground: it rolls forward by `bucket_size` blocks (no full reset), uses blocks as the native time unit (deterministic across validators), and limits state overhead via aggregation into buckets.

### Extensibility via Hooks

The rate limiting and bridge vault logic are implemented as `MintHook` and `BurnHook` implementations rather than being embedded directly in the erc20 module. This separation of concerns means:

- The erc20 module remains focused on token pair management and ERC-20 compatibility.
- The rate limiter and bridge vault can be activated, upgraded, or disabled independently of the erc20 module.
- Future security modules (e.g., compliance checks, additional security layers) can register as hooks without modifying the erc20 module.

### Bridge Vault vs. Isolated Wrapped Tokens

An alternative design is to give each bridge its own wrapped token (e.g., `axlXRP`, `wmhXRP`) and let users trade between them on a DEX. This was rejected because:

- **Liquidity fragmentation:** Two separate XRP-denominated assets split liquidity across every pool on the chain. DeFi protocols, CEXes, and user wallets would all need to reason about two assets where one is expected today.
- **Bridge UX:** Users would need to explicitly pick which wrapped token to hold and actively manage conversion when switching between exit paths.
- **Contagion via DEX:** A DEX pool holding both tokens acts as a price-equalizing mechanism. If one token depegs, arbitrageurs drain the other token from the pool to buy the cheap depegged token, spreading the loss.

The bridge vault preserves a single canonical XRP token on XRPL EVM while providing per-bridge accounting for composition enforcement.

### Caller-Based Bridge Identification

The bridge vault identifies which bridge is calling by matching `contract.Caller()` against `RegisteredBridge.minter_address`. This was chosen over alternatives because:

- **Zero bridge UI changes:** Bridges call `mint(to, amount)` and `burn(from, amount)` exactly as they do today. No routing parameters, no vault address, no explicit bridge selection.
- **Zero bridge contract changes:** The existing Axelar and Wormhole minting contracts are unmodified.
- **No explicit routing:** A router contract or explicit bridge ID parameter in the call would require all bridge front-ends and relayers to be updated in lockstep with the vault deployment.

The caller address is a tamper-proof identity: it is set by the EVM call stack and cannot be spoofed by calldata.

### Composition Bounds Design

**Why a cap (`max_composition`)?** Without a cap, a single bridge could grow to represent 100% of the vault. If that bridge is then compromised, the attacker controls the entire canonical XRP supply on XRPL EVM. A cap forces supply diversification, bounding the maximum damage from any single bridge failure.

**Why a floor (`min_composition`)?** Without a floor, a run on a healthy bridge can drain it to zero even while a compromised bridge continues operating. The floor protects the vault's healthy underlyings from being fully extracted through the bridge exit mechanism, leaving at least `min_composition` of the healthy bridge's underlying in the vault.

**Why both are needed?** The cap limits upside concentration; the floor limits downside drain. Together they create a bounded corridor for each bridge's share of total supply.

### Three Components in One Document

The three components form a coherent security stack. Multi-minter authorization enables the feature, the bridge vault limits per-bridge concentration and drain, and the rate limiter constrains the velocity of any attack. Separating them into independent specifications risks the multi-minter extension shipping without a concrete commitment to the security mitigations. Specifying them together signals to reviewers, implementers, and bridge operators that multi-minter support comes with clearly defined security controls.

### Considered Alternative: Bridge Attestation Oracle

An earlier draft of this specification included a `x/collateral` module — a bridge attestation oracle that capped each bridge's minting to the amount of XRP locked in its door account on XRPL, as reported by an off-chain oracle committee.

This approach was abandoned for two reasons:

1. **Multi-chain door accounts:** Axelar's door account on XRPL serves all Axelar-connected chains simultaneously, not only XRPL EVM. Attesting the total balance of the door account would vastly overstate the collateral backing XRPL EVM XRP specifically. Attributing a per-chain share of the door account balance requires off-chain accounting that cannot be verified on-chain.

2. **Circular flow attack:** Even for a single-chain bridge, the oracle is vulnerable to a circular flow attack when the attacker controls both the minting key and the door account. The attacker mints XRP, exits through a second bridge to recover XRPL XRP, redeposits into the compromised door account, waits for the next attestation epoch, and mints again — extracting all collateral over multiple cycles. The oracle faithfully reports the recycled balance and cannot distinguish legitimate from recycled funds.

The bridge vault avoids both problems: it requires no external attestation and its composition bounds constrain damage from a full bridge compromise to at most `max_composition` of the total supply.

### Known Limitation: Circular Flow Attack on the Bridge Vault

The bridge vault does not eliminate the circular flow attack — it bounds it. Consider a compromised bridge with `max_composition = 0.60`. The attacker can:

1. Mint XRP via the compromised bridge until that bridge holds 60% of the vault (the cap is reached).
2. Exit XRP through the second bridge to acquire XRPL XRP on the source chain.
3. Redeposit into the compromised bridge's door account (irrelevant to the vault — the vault does not check external balances).
4. The vault does not update on deposit. The attacker cannot mint more until users deposit through the compromised bridge or the other bridge's share grows.

The cap limits the attacker's share of on-chain XRP to `max_composition` regardless of how much XRP they cycle through the other bridge.

The floor further complicates the attack: burning XRP out through the second (healthy) bridge is rejected if it would drop the healthy bridge below `min_composition`. This prevents the attacker from draining the healthy bridge's supply while minting through the compromised one.

## Backwards Compatibility

### Multi-Minter Authorization (`x/erc20`)

- **Protobuf breaking change.** The `TokenPair` message adds a new field (`owner_addresses`, field 6) and deprecates `owner_address` (field 5). Since a new field number is used, this is wire-compatible for decoders that tolerate unknown fields. However, clients that rely on `owner_address` being populated MUST be updated. As a transitional measure, `owner_address` SHOULD be populated with the first element of `owner_addresses` in query responses.
- **State migration required.** A chain upgrade handler MUST migrate all existing `TokenPair` records. The upgrade is a coordinated halt-and-restart.
- **Precompile ABI unchanged.** The ERC-20 precompile's `mint(address,uint256)` function signature does not change. EVM callers are unaffected.
- **Error codes preserved.** `ErrMinterIsNotOwner` continues to be returned for unauthorized minters.

### Mint Rate Limiting

- **New module.** The rate limiter introduces new state objects (`MinterRateLimit`, `MintBucket`) and new governance parameters. No existing interfaces are modified beyond the addition of the `MintHook` and `BurnHook` calls in the erc20 keeper.
- **erc20 module change.** The erc20 keeper MUST be extended to accept and call `MintHook` and `BurnHook` implementations. This is an internal interface change that does not affect the external precompile ABI or Cosmos message interface.
- **Behavioral change.** Previously-successful mints MAY be rejected if they exceed the rate limit. Bridge operators MUST be aware of the configured rate limits.

### Bridge Vault (`x/vault`)

- **New module.** The `x/vault` module introduces its own state objects (`RegisteredBridge`, `BridgeUnderlying`, `BridgeVault`), messages (`MsgRegisterBridge`, `MsgUnregisterBridge`, `MsgUpdateBridgeComposition`), and governance parameters.
- **No external interface changes.** Bridges continue to call `mint(to, amount)` and `burn(from, amount)` on the XRP ERC-20 contract exactly as before. The vault intercepts these calls via hooks, invisible to the bridge.
- **Behavioral change.** Previously-successful mints MAY be rejected if they would exceed the bridge's composition cap. Previously-successful burns MAY be rejected if they would drop the bridge below the composition floor. Bridge operators MUST register their minter address in the vault before the vault is activated.

### Activation

Each component SHOULD be gated behind a distinct chain upgrade handler with a unique upgrade name. The multi-minter extension MUST be activated first. The bridge vault and rate limiter CAN be activated independently but are expected to ship together in a single upgrade for simplicity and reduced operational burden on validators.

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

### Bridge Vault (`x/vault`)

| # | Description | Expected Result |
|---|---|---|
| 3.1 | Registered bridge mints within composition cap | Succeeds; bridge balance and total updated |
| 3.2 | Registered bridge mints amount that would push its share above `max_composition` | Rejected with `ErrCompositionCapExceeded` |
| 3.3 | Unregistered caller mints (pass-through) | Succeeds if authorized by `x/erc20`; vault is no-op |
| 3.4 | Registered bridge burns within composition floor | Succeeds; bridge balance and total updated |
| 3.5 | Registered bridge burns amount that would drop its share below `min_composition` | Rejected with `ErrCompositionFloorReached` |
| 3.6 | Bridge attempts to burn more than its credited balance | Rejected with `ErrInsufficientBridgeBalance` |
| 3.7 | Two registered bridges have independent composition tracking | Bridge A minting does not affect Bridge B's mint cap |
| 3.8 | `MsgRegisterBridge` with `min_composition >= max_composition` | Rejected |
| 3.9 | `MsgUnregisterBridge` when bridge has non-zero balance | Rejected |
| 3.10 | `MsgUpdateBridgeComposition` tightens cap below current composition | Accepted (existing supply is grandfathered; future mints enforce new cap) |
| 3.11 | First mint into empty vault | Succeeds regardless of `max_composition` (bootstrap special case) |
| 3.12 | `VaultParams.enabled = false` | Both mint and burn hooks are no-ops; composition not enforced |

### Cross-Component Integration

| # | Description | Expected Result |
|---|---|---|
| C.1 | Unauthorized sender, within vault caps and rate limits | Rejected by `x/erc20` authorization |
| C.2 | Authorized sender, exceeds composition cap, within rate limit | Rejected by `x/vault` BeforeMint hook |
| C.3 | Authorized sender, within composition cap, exceeds rate limit | Rejected by rate limiter |
| C.4 | Bridge vault not active: authorized sender mints without composition check | Succeeds (vault hook is no-op) |
| C.5 | Rate limiter not active: authorized sender mints without rate limit check | Succeeds (rate limit hook is no-op) |
| C.6 | All three components active, authorized sender within all limits | Succeeds |
| C.7 | Authorized burn rejected by vault floor | Rejected before bridge exit is processed |

## Reference Implementation

Reference implementations will be provided as pull requests to the following repositories:

- **Multi-Minter (`x/erc20`):** [`xrplevm/evm`](https://github.com/xrplevm/evm) — erc20 module protobuf, keeper, precompile, and state migration changes.
- **Bridge Vault (`x/vault`):** [`xrplevm/evm`](https://github.com/xrplevm/evm) or [`xrplevm/node`](https://github.com/xrplevm/node) — bridge registration state, composition cap and floor enforcement, MintHook and BurnHook implementations.
- **Rate Limiting:** [`xrplevm/evm`](https://github.com/xrplevm/evm) or [`xrplevm/node`](https://github.com/xrplevm/node) — rate limit state, bucketed accumulation, mint hook implementation, and enforcement logic.

Per the XES specification process, this document cannot be promoted to a Candidate Specification until the relevant changes have been merged into the node repository's `main` branch.

## Security Considerations

### Shared Security Model (Multi-Minter Alone)

With only the multi-minter extension deployed, all authorized bridges share a single security domain. A compromise of any bridge allows the attacker to mint unbounded XRP, diluting the entire canonical supply on XRPL EVM. The security posture becomes approximately as strong as the weakest authorized bridge. This is the primary motivation for the bridge vault and rate limiter, which SHOULD be deployed together with the multi-minter extension.

### Bridge Compromise Blast Radius (Bridge Vault)

With the bridge vault active, a fully compromised bridge can mint at most `max_composition` of the vault's total supply before the cap halts further minting. For example, with `max_composition = 0.60` and a vault total of 1,000,000 XRP, the attacker can mint at most 600,000 XRP before hitting the cap (assuming the vault is initially balanced). This bounds the worst-case inflation from a single bridge failure to a known fraction of total supply.

The composition floor provides a complementary bound: a healthy bridge's underlying cannot be extracted below `min_composition` of total supply via burn calls, regardless of what the compromised bridge is doing. This limits cross-bridge drainage.

### Circular Flow Attack

As described in the Rationale section, the bridge vault bounds but does not eliminate the circular flow attack. A fully compromised bridge (both minting key and door account) can:

1. Mint up to `max_composition` of the vault total.
2. Exit through the second bridge, draining that bridge's share toward `min_composition`.
3. The vault prevents further minting via the cap and further burning via the floor.

The attack is self-limiting: the attacker cannot exceed `max_composition` without cross-bridge cooperation, and the floor prevents draining healthy bridges. The combined effect of the vault cap and rate limiter bounds both the magnitude and velocity of the attack, maximizing the governance response window.

**Governance monitoring:** The repeated pattern of large mints followed by immediate cross-bridge exits is fully observable on-chain. Bridge operators and governance SHOULD monitor composition ratios and react to anomalous patterns before caps are reached.

### Rate Limit Gaming

An attacker who compromises a bridge can mint at exactly the rate limit continuously. The rate limiter does not prevent exploitation — it slows it. The value of rate limiting is buying time for governance to respond (e.g., removing the compromised minter via an emergency proposal).

This imposes a constraint on parameter selection: **the governance response time MUST be faster than the time it takes for an attacker to exhaust the rate limit and cause material damage.** Rate limit parameters SHOULD be calibrated accordingly.

### Cross-Bridge Liquidity Extraction

Because both bridges mint the same fungible XRP, users can deposit through one bridge and withdraw through another. This creates asymmetric pressure on bridge liabilities: one bridge's door account may be drained while the other accumulates excess XRP. The bridge vault's floor partially constrains this for the on-chain portion (burns cannot drop a bridge below `min_composition`), but rebalancing of door accounts remains an off-chain concern. Bridge operators SHOULD monitor their door account balances and manage rebalancing accordingly.

### Governance Key Management

Adding and removing minters (`x/erc20`), registering bridges and updating composition bounds (`x/vault`), and configuring rate limits are all governance-controlled actions. The security of the entire multi-bridge system ultimately depends on the security and responsiveness of governance.

Recommendations:

- Emergency minter removal SHOULD support expedited governance proposals to minimize response time.
- Rate limit emergency pause (`max_amount_per_window = 0`) provides an immediate circuit breaker that governance can trigger.
- Bridge vault kill switch (`VaultParams.enabled = false`) provides an emergency bypass if the vault implementation has a bug that would block legitimate minting or burning.
- Composition bounds (`max_composition`, `min_composition`) SHOULD be set conservatively at launch and adjusted via governance as operational patterns are established.

### State Migration Risk (`x/erc20`)

The protobuf change from `owner_address` to `owner_addresses` requires a state migration during a chain upgrade. An incorrect migration could result in token pairs with empty `owner_addresses`, which would permanently lock minting for those pairs until corrected by a subsequent upgrade. Thorough migration testing on testnets is essential before mainnet deployment.

### Defense-in-Depth Evaluation Order

When all three components are active, mint requests MUST be evaluated in the following order:

1. **Authorization (`x/erc20`):** Is the sender in `owner_addresses`?
2. **Composition cap (`x/vault`):** Would this mint push the bridge above `max_composition`?
3. **Rate limit:** Is `window_total + amount <= max_amount_per_window`?

Burn requests are evaluated:

1. **Composition floor (`x/vault`):** Would this burn drop the bridge below `min_composition`?

A failure at any layer MUST reject the mint or burn. If the bridge vault or rate limiter modules are not yet activated, the corresponding checks MUST be skipped (no-op). The multi-minter authorization MUST function independently regardless of whether the other components are active.

## Future Work

### Composition-Based Dynamic Minting Fees

The bridge vault's composition tracking opens the door to a dynamic fee mechanism that incentivizes vault rebalancing at the protocol level. The core idea: minting through a bridge that is already over-represented in the vault becomes progressively more expensive, while minting through an under-represented bridge becomes cheaper (potentially zero-fee).

For example, if Bridge A holds 55% of the vault against a `max_composition` of 60%, a mint through Bridge A would incur a higher fee than a mint through Bridge B at 45%. Conversely, a mint through Bridge B — which moves the vault toward balance — would incur a reduced fee or no fee at all.

This creates a price signal that propagates to bridge routing: aggregators, front-ends, and arbitrageurs would naturally route deposits through the cheaper (under-represented) bridge, keeping the vault balanced without requiring manual intervention or governance action. The protocol effectively optimizes bridge routing through fee incentives rather than hard caps alone.

Key design questions for a future specification:

- **Fee curve shape:** Linear, exponential, or step-function relationship between composition deviation and fee. Steeper curves near the cap provide stronger rebalancing pressure.
- **Fee destination:** Fees could be burned (deflationary), sent to a community pool, or redistributed to the under-represented bridge's depositors as a rebalancing incentive.
- **Interaction with hard caps:** Dynamic fees would complement — not replace — the hard composition cap and floor. The fee provides a soft incentive before the hard limit is reached.
- **Burn-side fees:** Symmetric fees on burns (cheaper to exit through the over-represented bridge, more expensive to exit through the under-represented one) would further accelerate rebalancing.
