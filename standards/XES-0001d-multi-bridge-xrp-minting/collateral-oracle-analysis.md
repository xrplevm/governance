# Why a Collateral Oracle Does Not Prevent Collateral Loss

## Summary

A Collateral Oracle — an on-chain mechanism that caps each bridge's minting to the amount of XRP locked in its XRPL door account — was evaluated as a security control for the multi-bridge XRP minting design. After analysis, we concluded that it does not prevent total collateral extraction in a bridge compromise scenario and introduces significant complexity without meaningful security gain over a rate limiter alone.

## The Circular Flow Attack

Consider two bridges (A and B), each with 500k XRP locked in their respective door accounts on XRPL (1M XRP total collateral). Bridge A is compromised.

1. The attacker mints 500k XRP on XRPL EVM via Bridge A (within its attestation cap).
2. The attacker exits 500k XRP through Bridge B. Bridge B releases 500k XRP from its door account to the attacker on XRPL.
3. The attacker redeposits that 500k XRP into Bridge A's door account on XRPL.
4. At the next attestation epoch, Bridge A's door account now shows 1M XRP. The collateral oracle faithfully reports this.
5. The attacker mints another 500k XRP via Bridge A (the updated attestation cap allows it).
6. The attacker withdraws the full 1M XRP from Bridge A's door account on XRPL.

**Result:** The attacker extracted all 1M XRP — the entire system collateral across both bridges. The collateral oracle did not prevent the loss; it actually enabled step 5 by reflecting the inflated door account balance as valid collateral.

## Why This Happens

The collateral oracle reports the *actual* balance in a door account, but a compromised bridge can manipulate its own door account balance through the circular flow (mint → exit via other bridge → redeposit → attest → mint again). The oracle cannot distinguish between legitimate user deposits and attacker-recycled funds. Because the minted XRP is fungible on XRPL EVM, cross-bridge exits are always possible, making this circular flow an inherent structural property.

## What Actually Works: Rate Limiting

A rate limiter that constrains how much XRP a bridge can mint per time window provides the real security benefit:

- It bounds the **velocity** of any attack, regardless of collateral manipulation.
- It creates a **governance response window** — the community can detect the attack and remove the compromised minter before total collateral is drained.
- It is simple to implement, requires no off-chain oracle infrastructure, and introduces no new trust assumptions.

The rate limiter's effectiveness depends on calibrating the minting cap so that governance response time is faster than the time needed for an attacker to cause material damage.

## Conclusion

The Collateral Oracle adds substantial complexity (a new Cosmos SDK module, an off-chain oracle committee, epoch-based consensus, quorum mechanisms, and staleness management) without preventing the fundamental attack vector. A rate limiter achieves the same practical security outcome — buying time for governance to respond — with far less complexity and no additional trust assumptions. We recommend proceeding with multi-minter authorization and rate limiting as the two core protocol changes.