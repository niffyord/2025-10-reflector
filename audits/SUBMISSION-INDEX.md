# Reflector Audit - High Severity Findings Submission

## Submission Summary

Two High severity vulnerabilities identified in the Reflector oracle protocol:

1. **H01**: Integer Overflow in extend_ttl Causes DoS Through Panic on Large Amounts
   - File: `oracle/src/assets.rs` - `extend_ttl` function
   - Impact: DoS via overflow panic, potential fund loss on revert

2. **H02**: Out-of-Bounds Access in extract_update_record_prices Due to Missing Mask-Price Validation
   - File: `oracle/src/prices.rs` - `extract_update_record_prices` and `extract_single_update_record_price` functions
   - Impact: DoS via panic when admin submits malformed PriceUpdate

## Files

- `H01-extend-ttl-overflow.md` - Complete submission report for finding #1
- `H02-mask-price-mismatch.md` - Complete submission report for finding #2

Both reports follow the Code4rena template with:
- ✅ Succinct title
- ✅ Severity: High
- ✅ Target function/file specified
- ✅ Summary paragraph
- ✅ Impact analysis with concrete loss paths
- ✅ Root cause with exact line references and GitHub permalinks
- ✅ Proof of concept with reproduction steps
- ✅ Recommended mitigation with code examples

## Validation Status

Both findings have been validated and confirmed as High severity issues by code review.

