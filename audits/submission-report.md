Title: Beam invocation cost config lacks admin auth
Severity: High
Target: beam-contract/src/lib.rs – set_invocation_costs_config

## Summary
Beam exposes `set_invocation_costs_config` as an admin-only knob, but the implementation forgets to enforce admin authorization. Any caller can overwrite the fee tiers and effectively disable or DoS Beam’s pay-per-use model.

## Impact
- Zero-cost read access: attackers burn the minimum authorization fee once, set all invocation costs to zero, and every subsequent call becomes free. Sponsors lose the intended XRF burn revenue while still paying node costs.
- Forced over-burn: alternatively, malicious callers can spike the cost vector so every invocation attempts to burn large amounts of XRF, pricing legitimate users out or draining their pre-authorized allowances.
- Both paths break the protocol’s monetization and availability assumptions with a single transaction.

## Root Cause
- `BeamOracleContract::set_invocation_costs_config` calls `set_costs_config` directly, skipping the base contract’s admin check (`beam-contract/src/lib.rs#L394-L405`).
- The storage setter `set_costs_config` writes the attacker-supplied vector into instance storage without validation (`beam-contract/src/cost.rs#L19-L33`).
- Fee charging later reads this mutable vector and applies it unconditionally (`beam-contract/src/cost.rs#L34-L60`).

Permalink references:
- https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/lib.rs#L394-L405
- https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/cost.rs#L19-L60

## Proof of Concept
Run inside `beam-contract/src/tests.rs` (all helpers already exist):

```rust
#[test]
fn anyone_can_zero_fees() {
    let (env, client, init_data) = init_contract_with_admin();
    env.mock_all_auths(); // any address acts without admin signature

    // Non-admin address rewrites costs to zero
    let attacker = Address::generate(&env);
    let zero_costs = Vec::from_array(&env, [0, 0, 0, 0, 0]);
    client.set_invocation_costs_config(&zero_costs); // succeeds without admin auth

    // Configure fee token so burns should occur
    let fee_asset = env
        .register_stellar_asset_contract_v2(init_data.admin.clone())
        .address();
    client.set_fee_config(&FeeConfig::Some((fee_asset.clone(), 1_000_000)));

    // Mint attacker some fee tokens, then call lastprice
    let token = TokenClient::new(&env, &fee_asset);
    token.mint(&attacker, &100_000_000);
    client.lastprice(&attacker, &init_data.assets.first_unchecked());

    // Balance stays untouched because cost=0 exits early
    assert_eq!(token.balance(&attacker), 100_000_000);
}
```

Expected result: the test passes, demonstrating fee tiers are editable without admin approval and that burns are skipped after setting zero costs.

## Recommended Mitigation
- Enforce admin authorization before writing fee configs. For example:
  ```rust
  pub fn set_invocation_costs_config(e: &Env, config: Vec<u64>) {
      oracle::auth::panic_if_not_admin(e);
      set_costs_config(e, &config);
  }
  ```
- Optionally, add bounds checks on the config vector (length, overflow guards) to prevent denial-of-service via extreme values.

---

Title: TWAP requests underpay by hardcoding periods = 1
Severity: Medium
Target: beam-contract/src/lib.rs – twap

## Summary
Beam’s TWAP endpoint bills every request as if it touched only one record, ignoring the `records` argument. Users can pull large rolling averages while paying the minimum fee, undercutting the intended per-period pricing model.

## Impact
- Fee leakage scales with requested window size. A 20-period TWAP costs the same as a single-period quote, reducing revenue by up to ~95% compared to configured pricing.
- Attackers can script frequent large-window TWAPs at bargain cost to extract the most valuable analytics endpoints for pennies.
- Over time, cumulative undercharging erodes the economics that fund oracle maintenance.

## Root Cause
- Beam calls `charge_invocation_fee(e, &caller, InvocationComplexity::Twap, 1);` (`beam-contract/src/lib.rs#L293-L296`).
- The fee calculator expects the actual number of periods; it only applies per-period modifiers when `periods > 1` (`beam-contract/src/cost.rs#L41-L57`).
- Pulse correctly relays user input; Beam’s TWAP is the only endpoint that hardcodes `periods = 1`.

Permalink references:
- https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/lib.rs#L293-L296
- https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/cost.rs#L41-L57

## Proof of Concept
Add to `beam-contract/src/tests.rs`:

```rust
#[test]
fn twap_undercharges() {
    let (env, client, init_data) = init_contract_with_admin();
    env.mock_all_auths();

    // Set fee token and costs (defaults suffice for TWAP and N modifier)
    let fee_asset = env
        .register_stellar_asset_contract_v2(init_data.admin.clone())
        .address();
    client.set_fee_config(&FeeConfig::Some((fee_asset.clone(), 1_000_000)));

    let caller = Address::generate(&env);
    let token = TokenClient::new(&env, &fee_asset);
    token.mint(&caller, &100_000_000);

    // Call TWAP for 20 periods, then for 2 periods
    client.twap(&caller, &init_data.assets.first_unchecked(), &20);
    let after_large = token.balance(&caller);
    client.twap(&caller, &init_data.assets.first_unchecked(), &2);
    let after_small = token.balance(&caller);

    assert_eq!(100_000_000 - after_large, 100_000_000 - after_small);
}
```

Expected result: both invocations burn exactly the same amount, proving the fee logic ignores `records`.

## Recommended Mitigation
- Forward the caller-supplied `records` (capped to 20) into the billing call:
  ```rust
  let billable = core::cmp::min(records, 20);
  charge_invocation_fee(e, &caller, InvocationComplexity::Twap, billable);
  ```
- Consider rejecting `records == 0` or values beyond the cache cap for consistency.

---

Title: Beam overcharges history reads above 20 records
Severity: Medium
Target: beam-contract/src/lib.rs – prices / x_prices / twap

## Summary
Beam charges users for every requested record even when the oracle layer quietly truncates responses to 20 entries. Large `records` inputs burn more tokens without returning additional data, and TWAP requests can fail after paying the inflated fee.

## Impact
- Users issuing `records > 20` pay for nonexistent data. For example, requesting 100 records burns 5× the intended fee while only 20 prices are returned.
- TWAP requests above 20 records burn the higher cost and then return `None` because the oracle can’t fulfill the window, effectively charging for failed calls.
- This makes Beam consumption unpredictable and erodes trust; integrators must clamp inputs manually to avoid wasting funds.

## Root Cause
- Beam forwards the raw `records` argument to `charge_invocation_fee` (`beam-contract/src/lib.rs#L206-L209`, `#L270-L279`, `#L293-L297`).
- The core oracle immediately truncates `records` to `records.min(20)` before loading data (`oracle/src/prices.rs#L214-L233`).
- TWAP insists on receiving all requested records and returns `None` if fewer are available (`oracle/src/prices.rs#L245-L257`).

Permalink references:
- https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/lib.rs#L206-L209
- https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/lib.rs#L270-L279
- https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L214-L257

## Proof of Concept
Extend `beam-contract/src/tests.rs`:

```rust
#[test]
fn history_overcharge() {
    let (env, client, init_data) = init_contract_with_admin();
    env.mock_all_auths();
    let fee_asset = env
        .register_stellar_asset_contract_v2(init_data.admin.clone())
        .address();
    client.set_fee_config(&FeeConfig::Some((fee_asset.clone(), 1_000_000)));

    let caller = Address::generate(&env);
    let token = TokenClient::new(&env, &fee_asset);
    token.mint(&caller, &100_000_000);

    client.prices(&caller, &init_data.assets.first_unchecked(), &25);
    let after_big = token.balance(&caller);
    client.prices(&caller, &init_data.assets.first_unchecked(), &5);
    let after_small = token.balance(&caller);

    // Same response size (max 20), but bigger fee burned for 25-request
    assert!(after_big - after_small > 0);
}

#[test]
fn twap_charged_then_none() {
    let (env, client, init_data) = init_contract_with_admin();
    env.mock_all_auths();
    let fee_asset = env
        .register_stellar_asset_contract_v2(init_data.admin.clone())
        .address();
    client.set_fee_config(&FeeConfig::Some((fee_asset.clone(), 1_000_000)));

    let caller = Address::generate(&env);
    let token = TokenClient::new(&env, &fee_asset);
    token.mint(&caller, &100_000_000);

    let result = client.twap(&caller, &init_data.assets.first_unchecked(), &30);
    assert!(result.is_none()); // oracle truncated to 20 records
    assert!(token.balance(&caller) < 100_000_000); // fee already burned
}
```

Expected result: first test shows higher burn for same-sized output; second demonstrates a paid call returning `None`.

## Recommended Mitigation
- Align billing with actual work: compute `let billable = records.min(20);` before calling `charge_invocation_fee`.
- For TWAP, return an error when `records > 20` instead of billing and failing, or expand the oracle layer to support larger windows end-to-end.
