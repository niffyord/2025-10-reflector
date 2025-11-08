Title: Repeated `set_price` for the same timestamp shifts history masks and corrupts alignment
Severity: Medium
Target: oracle/src/prices.rs – `update_history_mask`, `set_price`

## Summary
Calling `set_price` multiple times with the same normalized `timestamp` shifts per-asset history bitmasks on every call but does not advance `last_timestamp`. This creates drift between mask positions and the actual stored snapshot timestamps, leading consumers to misinterpret which periods contain data.

## Impact
- Historical queries and cross/twap calculations may report updates in wrong periods or fail to find data that exists, degrading oracle correctness.
- Since writes are admin-only, a simple operational mistake (retrying a transaction with the same timestamp) can corrupt history structure for up to 256 periods.

## Root Cause
- `update_history_mask` unconditionally shifts each asset’s 256-bit history by one bit per call, regardless of whether the `timestamp` advanced beyond `last_timestamp`.
  - Missing-interval handling only runs when `update_delta > 1`; otherwise, a single shift occurs for the “current” update.
  - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L199-L219
- `set_last_timestamp` is only updated if `timestamp > last_timestamp`:
  - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L171-L179
- Therefore, if `timestamp == last_timestamp`, masks are shifted again but `last_timestamp` stays put, desynchronizing mask positions from the actual timeline.

## Proof of Concept
The test below deploys a local harness around `PriceOracleContractBase`, writes two identical snapshots at the same normalized timestamp, and then compares the history mask with the actual storage contents. After the duplicate write, `has_price(asset, period=1)` returns `true` even though no history record exists for `timestamp - resolution`, proving the mask drift.

```rust
use oracle::mapping;
use oracle::price_oracle::PriceOracleContractBase;
use oracle::prices;
use oracle::types::{Asset, ConfigData, FeeConfig, PriceUpdate};
use soroban_sdk::testutils::{Address as _, Ledger, LedgerInfo};
use soroban_sdk::{contract, contractimpl, Address, Bytes, Env, Vec};

#[contract]
pub struct Harness;

#[contractimpl]
impl Harness {
    pub fn config(env: Env, config: ConfigData, expiration: u32) {
        PriceOracleContractBase::config(&env, config, expiration);
    }

    pub fn set_price(env: Env, update: PriceUpdate, timestamp: u64) {
        PriceOracleContractBase::set_price(&env, update, timestamp);
    }

    pub fn has_price(env: Env, asset_index: u32, periods_ago: u32) -> bool {
        prices::has_price(&env, asset_index, periods_ago)
    }

    pub fn history_record_exists(env: Env, timestamp: u64) -> bool {
        prices::load_history_record(&env, timestamp).is_some()
    }
}

fn single_asset_update(env: &Env, price: i128) -> PriceUpdate {
    let mut updates = Vec::new(env);
    updates.push_back(price);

    let mut mask = [0u8; 32];
    let (byte, bitmask) = mapping::resolve_period_update_mask_position(0);
    mask[byte as usize] |= bitmask;

    PriceUpdate {
        prices: updates,
        mask: Bytes::from_array(env, &mask),
    }
}

#[test]
fn duplicate_timestamps_shift_history_mask() {
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

    let resolution: u32 = 300_000;
    let config = ConfigData {
        admin,
        history_retention_period: (100u64) * (resolution as u64),
        assets,
        base_asset,
        decimals: 14,
        resolution,
        cache_size: 0,
        fee_config: FeeConfig::None,
    };
    client.config(&config, &180);

    let snapshot = single_asset_update(&env, 42);
    let timestamp = (resolution as u64) * 2;
    client.set_price(&snapshot, &timestamp);
    client.set_price(&snapshot, &timestamp);

    let previous_period = timestamp - resolution as u64;
    let phantom_bit = client.has_price(&0, &1);
    let record_exists = client.history_record_exists(&previous_period);

    assert!(phantom_bit, "mask shows data for period 1");
    assert!(
        !record_exists,
        "temporary storage has no record for the previous period"
    );
}
```

## Recommended Mitigation
- In `set_price`, reject updates where `timestamp < last_timestamp` and treat `timestamp == last_timestamp` as a replacement without shifting masks (e.g., detect equality and update the existing snapshot without calling `update_history_mask`).
- Alternatively, change `update_history_mask` to only shift when `timestamp > last_timestamp`, and do a no-op when equal.
