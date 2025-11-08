# Additional Vulnerability Review

## Summary
- **Finding:** Unbounded gap-replay loop in `update_history_mask` DoSes price ingestion after long outages.
- **Impact:** Oracle admins cannot resume publishing prices once the gap exceeds the compute budget, breaking the "availability with bounded staleness" objective from the threat model and leaving consumers with permanently stale data.

## Threat Model Alignment
Objective 3 of the published threat model requires consumers to keep accessing fresh data as long as *one recent record exists*.【F:threatmodel.md†L12-L33】 The current padding strategy in `update_history_mask` attempts to uphold this by backfilling empty periods, but its implementation creates an inadvertent denial of service that violates the objective instead.【F:threatmodel.md†L46-L54】

## Detailed Walkthrough
1. **Gap detection:** When `set_price` is invoked, it delegates to `prices::update_history_mask`. The helper computes `update_delta = (timestamp - last_timestamp) / resolution` to see how many periods were missed since the previous update.【F:oracle/src/prices.rs†L120-L134】
2. **Padding loop:** If more than one period was skipped, the function enters a `for _ in 1..update_delta` loop that executes once per missing period. Every iteration allocates a `Vec` the size of the asset list, fills it with zeroes, and calls `mapping::update_history_mask` to shift the 256-bit history mask.【F:oracle/src/prices.rs†L122-L130】
3. **Per-asset work:** `mapping::update_history_mask` itself walks every asset index, shifts the existing `U256` mask, and writes the 32-byte slice back into storage. This is *O(asset_count)* work per iteration.【F:oracle/src/mapping.rs†L7-L40】
4. **Unbounded complexity:** The total effort therefore scales with `update_delta * asset_count`. With a 1-second resolution and 100 tracked assets, a 30-minute outage already yields ~180,000 inner iterations (`update_delta ≈ 1800`), which requires roughly 180,000 × 100 mask updates before the real snapshot is even processed. Soroban’s compute/budget caps will be exceeded long before finishing, causing `set_price` to trap and preventing the oracle from ever catching up.
5. **Permanent outage:** Because `last_timestamp` is only updated after `update_history_mask` succeeds, the failed transaction leaves the previous timestamp in place. Every subsequent attempt hits the same oversized `update_delta`, so admins are permanently locked out of publishing new data until they deploy patched code.【F:oracle/src/prices.rs†L161-L198】

## Why This Matters
- **Availability failure:** Consumers see the oracle freeze even though admins attempt to push a valid snapshot, violating the bounded-staleness promise in the threat model.【F:threatmodel.md†L12-L33】
- **Operational exploitability:** An attacker does not need contract permissions—simply disrupting the off-chain cluster (or waiting for the network to experience an extended outage) is enough to trigger the failure when operations resume. The longer the downtime or the larger the asset universe, the more certain the DoS becomes.
- **No automatic recovery:** There is no circuit breaker or cap that limits the padding loop, and the contract cannot self-heal; governance must deploy new code.

## Suggested Mitigations
1. **Bound the replay window:** Cap `update_delta` at 256 (the mask capacity) and treat any larger gap as a forced resynchronization without per-period padding.
2. **Amortize backlog processing:** Instead of replaying every empty slot in one transaction, persist the outstanding gap size and process it over multiple updates, or allow the admin to supply a batch of compressed placeholders in the new snapshot.
3. **Pre-flight checks:** Before entering the loop, assert that `update_delta * asset_count` is below a conservative threshold based on observed budget usage. Revert early with a descriptive error so automation can trigger a safe recovery procedure.

By fixing the padding algorithm, the protocol can meet its documented objective of remaining available after temporary outages while still preserving per-period history granularity.
