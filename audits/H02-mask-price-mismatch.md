Title: Out-of-Bounds Access in extract_update_record_prices Due to Missing Mask-Price Validation

Severity: High

Target: `oracle/src/prices.rs` – `extract_update_record_prices` and `extract_single_update_record_price` functions

## Summary

The `extract_update_record_prices` and `extract_single_update_record_price` functions use `get_unchecked` to access the `prices` vector without bounds validation. The `set_price` function validates that `update.prices.len() <= assets::load_all_assets(e).len()`, but does not verify that the number of set bits in `update.mask` equals `update.prices.len()`. A malformed `PriceUpdate` with more mask bits set than available prices causes `update_index` to exceed vector bounds, leading to panic and DoS.

## Impact

- **DoS vulnerability**: Admin can accidentally submit a malformed `PriceUpdate` with mismatched mask and prices, causing the contract to panic permanently
- **Oracle halt**: If `set_price` panics, price updates cannot be recorded, halting oracle functionality
- **Data corruption risk**: If panic doesn't occur (unlikely in Rust), prices could be misaligned with assets, causing incorrect price queries
- **Admin mistake vulnerability**: While admin actions require multisig, a single malformed update can permanently disable price updates

## Root Cause

- Line 436 in `oracle/src/price_oracle.rs`: Validation only checks `update.prices.len() > assets::load_all_assets(e).len()`, but does not verify mask-prices consistency
  - Reference: https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/price_oracle.rs#L436-L446
- Line 66 in `oracle/src/prices.rs`: `check_period_updated` increments `update_index` for each set bit in the mask
  - Reference: https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L61-L73
- Line 68: `price = update.prices.get_unchecked(update_index);` accesses prices without bounds checking
- Line 81: `return update.prices.get_unchecked(update_index);` also uses unchecked access
  - Reference: https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L76-L86
- **Missing validation**: No check that `count_set_bits(update.mask) == update.prices.len()` before processing

## Proof of Concept

```rust
// Setup: Oracle has 5 assets (indices 0-4)

// Admin submits malformed PriceUpdate
let malformed_update = PriceUpdate {
    prices: Vec::from_array(&e, [100, 200]), // Only 2 prices
    mask: Bytes::from_array(&e, &[
        0b11111111, // Bits set for assets 0-7 (8 bits set)
        0b11111111, // More bits set
        // ... total bits set exceeds prices.len()
    ]),
};

// Call set_price
PriceOracleContractBase::set_price(e, malformed_update, timestamp);

// set_price validation passes:
// - update.prices.len() (2) <= assets.len() (5) ✓

// extract_update_record_prices processes:
let mut update_index = 0;
for asset_index in 0..5 { // total = 5 assets
    if check_period_updated(&update.mask, asset_index) {
        // Asset 0: mask bit set → update_index=0, get_unchecked(0) = 100 ✓
        // Asset 1: mask bit set → update_index=1, get_unchecked(1) = 200 ✓
        // Asset 2: mask bit set → update_index=2, get_unchecked(2) ❌ OUT OF BOUNDS
        // Panic: index out of bounds: the len is 2 but the index is 2
    }
}
```

**Reproduction steps:**
1. Deploy oracle with 5 assets
2. Admin (or attacker with admin access) calls `set_price` with:
   - `prices: [100, 200]` (2 prices)
   - `mask: [0b11111111, 0b11111111, ...]` (bits set for 8+ assets, but only 2 prices)
3. `set_price` validation passes (prices.len() <= assets.len())
4. `extract_update_record_prices` is called with `total = 5`
5. Iteration reaches asset 2 with mask bit set
6. `update_index = 2`, calls `update.prices.get_unchecked(2)` but vector only has 2 elements
7. **Result**: Panic "index out of bounds"

**Expected behavior:** `set_price` should validate that the number of set bits in the mask equals `prices.len()` before processing.

**Actual behavior:** Validation passes, but `get_unchecked` panics when accessing out-of-bounds index.

## Recommended Mitigation

Add validation in `set_price` to ensure mask-prices consistency before processing:

```rust
// In oracle/src/price_oracle.rs, add before extract_update_record_prices:

// Count set bits in mask
let mut set_bits_count = 0;
let mask_bytes = update.mask.len();
for i in 0..mask_bytes {
    let byte = update.mask.get(i).unwrap_or_default();
    set_bits_count += byte.count_ones() as u32;
}

// Validate mask-prices consistency
if set_bits_count != update.prices.len() {
    panic_with_error!(&e, Error::InvalidPricesUpdate);
}
```

**Alternative:** Replace `get_unchecked` with bounds-checked `get()`:
```rust
// In oracle/src/prices.rs
price = update.prices.get(update_index)
    .ok_or_else(|| e.panic_with_error(Error::InvalidPricesUpdate))?;
```

This provides defense-in-depth, but the primary fix should be validation in `set_price` to prevent malformed updates from entering the system.

