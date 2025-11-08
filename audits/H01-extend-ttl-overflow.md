Title: Integer Overflow in extend_ttl Causes DoS Through Panic on Large Amounts

Severity: High

Target: `oracle/src/assets.rs` – `extend_ttl` function

## Summary

The `extend_ttl` function calculates the expiration time extension (`bump`) using unchecked arithmetic: `amount * 86400000 / fee`. When `amount` is large enough relative to a small `fee` value, the multiplication can overflow `i128::MAX`, and the subsequent `checked_add` operation on `asset_expiration` panics, causing a DoS attack vector. The token burn occurs before this check, meaning the transaction reverts after burning tokens, wasting user funds.

## Impact

- **DoS vulnerability**: Any user with sufficient token balance can call `extend_ttl` with a large `amount` to cause a panic, permanently blocking the function for that asset
- **Fund loss**: Token burn occurs before overflow validation, causing partial fund loss on revert
- **Exploitability**: With configurable fee values (e.g., `fee = 1` set by admin), the attack requires approximately `2.1×10¹¹` base units (~21k tokens with 7 decimals), making it feasible for attackers
- **Systemic risk**: If fees are misconfigured to small values, the oracle becomes vulnerable to DoS attacks

## Root Cause

- Line 141 in `oracle/src/assets.rs`: `let bump = amount * 86400000 / fee;` performs unchecked multiplication that can exceed `i128::MAX` (~9.2×10¹⁸) before division
  - Maximum safe `amount`: `i128::MAX / 86400000 ≈ 1.07×10¹¹`
  - With `fee = 1`, attack threshold: `~2.1×10¹¹` base units
- Line 139: Token burn occurs before overflow validation: `TokenClient::new(&e, &xrf).burn(&sponsor, &amount);`
- Line 156: `asset_expiration.checked_add(bump as u64).unwrap()` panics if `bump` exceeds `u64::MAX` or if the addition overflows
- The fee configuration is flexible and can be set to very small values via `set_fee_config`, enabling this attack vector
- Reference: https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/assets.rs#L139-L156

## Proof of Concept

```rust
// Setup: Admin sets fee to a very small value (malicious or mistaken)
// Oracle has fee_config = FeeConfig::Some((xrf_token, 1)) // fee = 1

// Attacker calls extend_ttl with large amount
let attack_amount: i128 = 2_100_000_000_000; // ~2.1×10¹² base units

// Step 1: Token burn succeeds
TokenClient::new(&e, &xrf).burn(&sponsor, &attack_amount); // ✅ Burns tokens

// Step 2: Calculate bump
let bump = attack_amount * 86400000 / 1;
// bump = 181_440_000_000_000_000
// This exceeds u64::MAX (18_446_744_073_709_551_615)

// Step 3: Attempt to add bump to expiration
asset_expiration.checked_add(bump as u64).unwrap(); // ❌ PANIC
// Transaction reverts, but tokens are already burned
```

**Reproduction steps:**
1. Initialize oracle with `fee_config` set to `(xrf_token, 1)` (very small fee)
2. Add an asset to the oracle
3. Mint substantial XRF tokens to an attacker address
4. Call `extend_ttl(attacker, asset, 2_100_000_000_000)`
5. Observe: Token burn succeeds, then panic occurs on overflow
6. Result: Transaction reverts, tokens are burned, function is DoS'd

**Expected behavior:** Transaction should revert before burning tokens, or overflow should be caught and handled gracefully.

**Actual behavior:** Tokens are burned, then panic occurs on overflow check, causing revert and fund loss.

## Recommended Mitigation

Use checked arithmetic throughout the calculation and validate bounds before token burn:

```rust
// Calculate bump with overflow protection
let bump = amount
    .checked_mul(86400000)
    .and_then(|x| x.checked_div(fee))
    .ok_or_else(|| e.panic_with_error(Error::InvalidAmount))?;

// Validate bump fits in u64 before burning tokens
if bump > u64::MAX as i128 {
    e.panic_with_error(Error::InvalidAmount);
}

// Now safe to burn tokens
TokenClient::new(&e, &xrf).burn(&sponsor, &amount);

// Add expiration with checked arithmetic
asset_expiration = asset_expiration
    .checked_add(bump as u64)
    .ok_or_else(|| e.panic_with_error(Error::InvalidAmount))?;
```

**Alternative:** Add validation in `set_fee_config` to ensure `fee` is above a minimum threshold (e.g., `>= 100_000`) to prevent small fee configurations that enable this attack.

