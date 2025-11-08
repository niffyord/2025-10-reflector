Title: Reflector – QA Report by <your-handle>
Severity: QA

## Summary
4 low-risk issues and 2 governance/centralization notes. Themes: time-unit/documentation mismatches, limits enforcement, result-size caps, and explicit initialization. These do not create asset loss but can cause misconfiguration, integration confusion, or operational surprises.

## Low-Risk Findings

- `L-01` Configuration time-unit mismatch (seconds vs milliseconds)
  - Description: Docs for configuration parameters say seconds, but internal logic treats them as milliseconds. Passing seconds will shrink periods 1000×, affecting normalization, staleness checks, and retention TTL.
  - References:
    - Config docs (seconds):
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/price_oracle.rs#L327-L334
    - Internals use milliseconds:
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/timestamps.rs#L4-L11
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/timestamps.rs#L23-L26
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/price_oracle.rs#L27-L34 (divides by 1000 on read)
  - Effect: Misconfigured resolution/retention leads to wrong period bucketing and unexpected None results or stale thresholds.
  - Recommendation: Either store ms but convert inputs from seconds at config-set time, or change docs and types to ms. Add validation that values are multiples of 1000 and within sane bounds.

- `L-02` Staleness threshold differs from README and is inconsistent across endpoints
  - Description: README states data is stale if older than 1 period, while lastprice gating returns None only when older than 2 periods; TWAP uses 1 period + 60 seconds.
  - References:
    - README invariant: > “stale” more than 1 period (1× timeframe)
      - https://github.com/code-423n4/2025-10-reflector/blob/main/README.md#L164-L167
    - Last-timestamp gating (2 periods):
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L16-L29
    - TWAP staleness (1 period + 60s):
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L249-L256
  - Effect: Integrators may consider a price “fresh” per documentation but get None in TWAP, or get lastprice when they expect staleness after a single period.
  - Recommendation: Align on a single threshold (prefer 1 period) and document the 60s grace if retained. Consider exposing a helper that returns “freshness status”.

- `L-03` Asset limit off-by-one and mismatch with stated invariant
  - Description: Code allows up to but effectively rejects the Nth element due to a post-insert `>=` check, and the hard-coded limit (1000) mismatches README’s 256 assets invariant.
  - References:
    - Limit constant and enforcement:
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/assets.rs#L5
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/assets.rs#L74-L80
    - README invariant (256 assets):
      - https://github.com/code-423n4/2025-10-reflector/blob/main/README.md#L151-L155
  - Effect: Confusion on maximum supported assets; batch adds that touch the boundary revert entirely.
  - Recommendation: Pre-validate `asset_list.len() + new.len() <= LIMIT` and decide on 256 vs 1000 for consistency. If 256 is desired, update constant and tests.

- `L-04` Result-size cap (20) is silent in API; TWAP returns None when window > 20
  - Description: Oracle caps history loads to 20 records; functions return fewer entries without explicit error, and TWAP returns None when it cannot satisfy the full window.
  - References:
    - History load cap and behavior:
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L214-L235
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/prices.rs#L243-L260
  - Effect: Integrations may assume larger windows succeed or that partial results are returned. Pulse endpoints inherit the same cap silently.
  - Recommendation: Document the 20-record cap in user-facing functions and consider reverting on `records > 20` for strictness, or return partials with an explicit flag.

## Governance / Centralization Risks

- `C-01` Admin upgradability and broad mutation powers
  - Description: Admin can update contract code and adjust core parameters (assets, cache, fee config) without a time-lock at the contract level. The README states a multisig governance model, but on-chain there’s no delay mechanism.
  - References:
    - Code update:
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/price_oracle.rs#L455-L468
    - Admin gate:
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/auth.rs#L19-L27
  - Effect: Operational centralization; risk is mitigated off-chain via DAO multisig, but worth explicitly acknowledging to integrators.
  - Recommendation: Document governance workflow (M-of-N, thresholds) in the contract docs and optionally add a time-lock or staged upgrades if feasible.

- `C-02` Protocol auto-upgrade scheduling
  - Description: When not on the latest protocol version, a call schedules an upgrade for T+1 day. Any address can trigger scheduling. Behavior differences exist during the transition window.
  - References:
    - Scheduling and upgrade logic:
      - https://github.com/code-423n4/2025-10-reflector/blob/main/oracle/src/protocol.rs#L23-L50
  - Effect: Temporary mixed behavior between v1 and v2 paths (legacy keys) for up to one day; acceptable but should be clearly communicated to downstream users.
  - Recommendation: Expose a public method to read scheduled upgrade time; publish an event when scheduling occurs; add docs for transitional behavior.

## Recommendations
- Clarify time units and enforce them at configuration set-time (convert seconds to ms or change docs/types to ms). Add bounds checks for resolution and retention.
- Align staleness thresholds across endpoints and documentation; consider exposing a “freshness” query helper.
- Pre-validate asset additions against the declared limit and reconcile the stated 256-asset invariant with the 1000 limit in code.
- Document the 20-record cap and standardize return semantics for over-window calls (revert vs partial + flag). Apply the same clarity to Pulse endpoints.


