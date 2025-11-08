Title: Beam TTL extension always panics because assets never get expiration slots
Severity: Medium
Target: beam-contract/src/lib.rs – `extend_asset_ttl` and asset initialization

## Summary
Beam passes an `initial_expiration_period` of `0` everywhere, so `assets::add_assets` never pushes expiration records for its assets. Later, `extend_asset_ttl` assumes those slots exist and calls `Vec::set`, which panics when the vector is empty. As a result, any attempt to pay XRF to keep a Beam feed alive will revert after burning gas, defeating the advertised “sponsor the feed” feature.

## Impact
- Sponsors cannot extend Beam feed lifetimes at all; every call to `extend_asset_ttl` fails after selecting a valid asset.
- Operations that rely on expiring idle assets (e.g., charging customers to keep bespoke feeds alive) are impossible, so the DAO cannot enforce or monetize TTL-based SLAs on Beam.
- Attackers can grief honest users by repeatedly calling `extend_asset_ttl` first, guaranteeing their transactions revert because the vector write always panics.

## Root Cause
- Beam hardcodes `initial_expiration_period = 0` both for admin configuration and for the user-facing `extend_asset_ttl` wrapper ([beam-contract/src/lib.rs#L105-L118](https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/lib.rs#L105-L118), [beam-contract/src/lib.rs#L360-L365](https://github.com/code-423n4/2025-10-reflector/blob/main/beam-contract/src/lib.rs#L360-L365)).
- `assets::add_assets` only appends expiration timestamps when `initial_expiration_period > 0`; otherwise `expiration` stays empty ([oracle/src/assets.rs#L53-L75](https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/assets.rs#L53-L75)).
- `assets::extend_ttl` later fetches that empty vector and calls `expiration.set(asset_index, …)`, which panics because Soroban’s `Vec::set` requires the index to exist ([oracle/src/assets.rs#L138-L160](https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/assets.rs#L138-L160), [`soroban-sdk` Vec::set docs](https://github.com/stellar/rs-soroban-sdk/blob/v23.0.3/soroban-sdk/src/vec.rs#L473-L489)).

## Proof of Concept
PoC test (`beam-contract/src/tests.rs`):
```rust
#[test]
#[should_panic(expected = "IndexBounds")]
fn beam_extend_ttl_always_panics() {
    let (env, client, init_data) = init_contract_with_admin();

    let fee_asset = env
        .register_stellar_asset_contract_v2(init_data.admin.clone())
        .address();
    let fee_config = FeeConfig::Some((fee_asset.clone(), 1_000_000));
    client.set_fee_config(&fee_config);

    let new_asset = Asset::Stellar(Address::generate(&env));
    let mut extra_assets = Vec::new(&env);
    extra_assets.push_back(new_asset.clone());
    client.add_assets(&extra_assets);

    let sponsor = Address::generate(&env);
    let token = StellarAssetClient::new(&env, &fee_asset);
    token.mint(&sponsor, &10_000_000);

    client.extend_asset_ttl(&sponsor, &new_asset, &1_000_000i128);
}
```
Run with:
```
cargo test -p reflector-beam-contract beam_extend_ttl_always_panics -- --nocapture
```
The test executes these steps:
1. Deploys Beam, sets a fee token, and then adds a new asset **after** fees are configured (Beam hardcodes `initial_expiration_period = 0` here).
2. Mints XRF to a sponsor and invokes `extend_asset_ttl` for the newly added asset.
3. The token burn succeeds, but `assets::extend_ttl` crashes with `HostError: Error(Object, IndexBounds)` when attempting to `set` the non-existent expiration slot, proving that TTL extensions are impossible for any asset added post-initialization.

## Recommended Mitigation
- Ensure Beam assets always have expiration slots: push zeros when `initial_expiration_period = 0`, or immediately call `assets::init_expiration_config` after adding assets.
- Alternatively, reject `extend_asset_ttl` for assets without an expiration record before burning any tokens, and document that Beam requires a non-zero initial period.
- Once slots exist, consider clamping minimum TTLs so TTL economics stay aligned with the DAO’s expectations.
