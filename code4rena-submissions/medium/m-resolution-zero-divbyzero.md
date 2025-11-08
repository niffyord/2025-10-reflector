Title: `resolution == 0` bricks oracle math and can trigger divide-by-zero
Severity: Medium
Target: oracle/src/timestamps.rs – `normalize`; oracle/src/prices.rs – `retrieve_asset_price_data`

## Summary
The protocol accepts `resolution` without validation. If `resolution == 0`, timestamp normalization collapses to 0 and `retrieve_asset_price_data` divides by `resolution` when computing `period`, causing division-by-zero panics or permanently returning no data.

## Impact
- Admins (or migrations) can misconfigure the oracle into an unusable state where writes/reads fail unpredictably.
- Public reads may panic due to divide-by-zero in period calculation, breaching the “public functions never panic” invariant and impacting availability.

## Root Cause
- `timestamps::normalize` returns 0 for `timeframe == 0`, allowing misaligned timestamps to pass through as 0.
  - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/timestamps.rs#L4-L13
- `retrieve_asset_price_data` computes:
  - `period = (last - timestamp) / settings::get_resolution(e) as u64;`
  - If `resolution == 0`, this divides by zero.
  - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L23-L48

## Proof of Concept
The snippet below configures the oracle with `resolution = 0`, seeds a historical snapshot via the same internal helpers `set_price` uses (`update_history_mask` + `store_prices`), and then calls the public `price()` entry point. Because `last_timestamp > timestamp` while `resolution == 0`, `retrieve_asset_price_data` divides by zero and the call panics.

```rust
use oracle::mapping;
use oracle::price_oracle::PriceOracleContractBase;
use oracle::prices;
use oracle::types::{Asset, ConfigData, FeeConfig, PriceData, PriceUpdate};
use soroban_sdk::testutils::{Address as _, Ledger, LedgerInfo};
use soroban_sdk::{contract, contractimpl, Address, Bytes, Env, Vec};
use std::panic::{catch_unwind, AssertUnwindSafe};

#[contract]
pub struct Harness;

#[contractimpl]
impl Harness {
    pub fn config(env: Env, config: ConfigData, expiration: u32) {
        PriceOracleContractBase::config(&env, config, expiration);
    }

    pub fn price(env: Env, asset: Asset, timestamp: u64) -> Option<PriceData> {
        PriceOracleContractBase::price(&env, asset, timestamp)
    }

    pub fn unsafe_store(env: Env, update: PriceUpdate, all_prices: Vec<i128>, timestamp: u64) {
        prices::update_history_mask(&env, &all_prices, timestamp);
        prices::store_prices(&env, &update, timestamp, &all_prices);
    }
}

fn single_asset_snapshot(env: &Env, price: i128) -> (PriceUpdate, Vec<i128>) {
    let mut updates = Vec::new(env);
    updates.push_back(price);

    let mut mask = [0u8; 32];
    let (byte, bitmask) = mapping::resolve_period_update_mask_position(0);
    mask[byte as usize] |= bitmask;

    (
        PriceUpdate {
            prices: updates.clone(),
            mask: Bytes::from_array(env, &mask),
        },
        updates,
    )
}

#[test]
fn zero_resolution_divides_by_zero_on_reads() {
    let env = Env::default();
    let ledger_info = env.ledger().get();
    env.ledger().set(LedgerInfo {
        timestamp: 1200,
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

    let config = ConfigData {
        admin,
        history_retention_period: 86_400_000,
        assets,
        base_asset: base_asset.clone(),
        decimals: 14,
        resolution: 0,
        cache_size: 0,
        fee_config: FeeConfig::None,
    };
    client.config(&config, &180);

    let (snapshot, all_prices) = single_asset_snapshot(&env, 99);
    client.unsafe_store(&snapshot, &all_prices, &600_000u64);

    let result = catch_unwind(AssertUnwindSafe(|| {
        client.price(&base_asset, &1);
    }));

    assert!(result.is_err(), "period calculation divides by zero");
}
```

## Recommended Mitigation
- Validate `resolution` in `settings::init`, e.g., `assert!(resolution >= 60_000 && resolution <= 86_400_000);` (5 minutes to 1 day).
- Consider rejecting updates whose timestamps are not aligned to `resolution`, and never allow `resolution == 0`.
