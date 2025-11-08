Title: Unbounded `decimals` allows 10^N overflow causing public read panics
Severity: High
Target: oracle/src/prices.rs – `x_last_price`, `x_price`, `fixed_div_floor`

## Summary
`decimals` is not bounded at initialization. Multiple public read paths compute powers of 10 using `i128.pow(decimals)`. When `decimals > 38`, `10^decimals` overflows `i128`, causing panics in `x_last_price`, `x_price` (when base == quote), and during cross-price scaling in `fixed_div_floor`.

## Impact
- Any consumer calling `x_last_price(base, base)` or `x_price(base, base, ts)` will panic if `decimals` is misconfigured above the `i128` limit, halting critical oracle reads.
- Cross-price requests can also panic due to scaling in `fixed_div_floor` if `decimals` is too large relative to price magnitudes.
- Violates the invariant “All public functions never panic” in the README, undermining protocol availability guarantees.

## Root Cause
- No upper bound is enforced for `decimals` during `config` initialization in `PriceOracleContractBase`.
- Code computes `10^decimals` directly:
  - `x_price`/`x_last_price`: returns `10^decimals` when `base_asset == quote_asset`.
    - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L270-L275
    - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L271-L274 (returns `10i128.pow(decimals)`)
  - `fixed_div_floor` uses `10^ashift` and `10^bshift` for scaling:
    - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L327-L344
    - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L332-L341

## Proof of Concept
The following self-contained Soroban harness registers `PriceOracleContractBase`, configures it with `decimals = 39`, and then calls `x_price(base, base, ts)` inside `catch_unwind`. The call deterministically panics because `load_cross_price` computes `10i128.pow(39)` for identical assets, overflowing `i128`.

```rust
use oracle::price_oracle::PriceOracleContractBase;
use oracle::types::{Asset, ConfigData, FeeConfig, PriceData};
use soroban_sdk::testutils::{Address as _, Ledger, LedgerInfo};
use soroban_sdk::{contract, contractimpl, Address, Env, Vec};
use std::panic::{catch_unwind, AssertUnwindSafe};

#[contract]
pub struct Harness;

#[contractimpl]
impl Harness {
    pub fn config(env: Env, config: ConfigData, expiration: u32) {
        PriceOracleContractBase::config(&env, config, expiration);
    }

    pub fn x_price(env: Env, base: Asset, quote: Asset, timestamp: u64) -> Option<PriceData> {
        PriceOracleContractBase::x_price(&env, base, quote, timestamp)
    }
}

#[test]
fn x_price_panics_when_decimals_exceed_i128_limit() {
    let env = Env::default();
    let ledger_info = env.ledger().get();
    env.ledger().set(LedgerInfo {
        timestamp: 900,
        ..ledger_info
    });
    env.cost_estimate().budget().reset_unlimited();
    env.mock_all_auths();

    let contract_id = Address::generate(&env);
    env.register_at(&contract_id, Harness, ());
    let client = HarnessClient::new(&env, &contract_id);

    let admin = Address::generate(&env);
    let base_asset = Asset::Stellar(Address::generate(&env));
    let mut assets = Vec::new(&env);
    assets.push_back(base_asset.clone());

    let resolution = 300_000u32;
    let config = ConfigData {
        admin,
        history_retention_period: (100u64) * (resolution as u64),
        assets,
        base_asset: base_asset.clone(),
        decimals: 39,
        resolution,
        cache_size: 0,
        fee_config: FeeConfig::None,
    };
    client.config(&config, &180);

    let timestamp = 1u64;
    let result = catch_unwind(AssertUnwindSafe(|| {
        client.x_price(&base_asset, &base_asset, &timestamp);
    }));

    assert!(result.is_err(), "overflow panic expected");
}
```

## Recommended Mitigation
- Enforce `1 <= decimals <= 38` during initialization (`settings::init`), rejecting configs outside this safe range.
- Optionally, precompute and cache `10^decimals` during `config` with checked math; abort on overflow to avoid latent panics.
