# Reflector Threat Model

## Context & Goals
- Reflector oracles combine on-chain Soroban contracts with off-chain validator nodes that achieve consensus on price snapshots before submitting multisig admin transactions (`doc.md:3-45`, `README.md:32-67`).
- Two deployment flavors share the same core logic: **Pulse** (free, fixed cadence) and **Beam** (paid, flexible, faster) (`doc.md:27-33`).
- Primary security objective is to preserve integrity of stored prices and configuration while providing transparent, queryable market data to downstream Stellar contracts (`README.md:129-168`).

## Scope
- Audit scope restricts analysis to the shared oracle library plus `beam-contract` and `pulse-contract` entrypoints (`scope.txt`).
- External cluster software, token economics, and auxiliary tooling are out of scope but inform trust boundaries.

## Security Objectives
1. **DAO-governed control plane** - only the multisig admin may configure assets, publish price batches, and upgrade code (`doc.md:9-19`, `oracle/src/price_oracle.rs:339-368`).
2. **Price data correctness** - timestamps must align to the configured resolution, masks must encode which assets were updated, and history should be replayable via events (`oracle/src/price_oracle.rs:431-452`, `oracle/src/prices.rs:112-198`, `oracle/src/events.rs:12-41`).
3. **Availability with bounded staleness** - consumers can fetch last price/cross price/TWAP as long as at least one recent record exists (`oracle/src/prices.rs:16-58`, `oracle/src/prices.rs:200-260`); Beam endpoints additionally charge fees to discourage abuse (`beam-contract/src/lib.rs:173-320`, `beam-contract/src/cost.rs:43-95`).
4. **Economic protection of feeds** - asset TTL extensions must burn the configured XRF fee, and fee settings should only change under admin control (`oracle/src/assets.rs:108-160`, `oracle/src/price_oracle.rs:403-418`).
5. **Deterministic upgrades** - protocol versioning handles legacy storage layout transitions without data loss (`oracle/src/prices.rs:33-58`, `oracle/src/prices.rs:162-198`, `oracle/src/protocol.rs:4-54`).

## Trust Zones & Critical Assets
| Trust Zone | Description | Critical Assets / Controls |
| --- | --- | --- |
| DAO multisig admin | Cluster node consortium signs `config`, `set_price`, `set_fee_config`, `update_contract` invocations (`oracle/src/price_oracle.rs:339-468`). | Admin address in instance storage, wasm hash, fee config, asset list, price history. |
| Data providers (off-chain) | Nodes compute prices and share signed Soroban txns before submission (`doc.md:13-25`). | Off-chain price inputs, timestamp normalization, WebSocket peer mesh. |
| Pulse consumers | Any contract/account that reads oracle data without paying. | Read-only APIs for prices, cross prices, TWAP (`pulse-contract/src/lib.rs:14-350`). |
| Beam consumers | Callers who must authenticate and burn fee tokens for each read (`beam-contract/src/lib.rs:173-320`, `beam-contract/src/cost.rs:43-95`). | Invocation cost table, token client burn, caller auth requirements. |
| Fee sponsors / ecosystem | Addresses burning XRF to extend TTL or pay per-query fees (`oracle/src/assets.rs:108-160`, `beam-contract/src/cost.rs:43-95`). | XRF balances, TTL vectors, cache size. |

## Entry Points & Data Flows
1. **Initialization** via `config` sets immutable parameters (base asset, decimals, resolution) and seeds the asset set (`oracle/src/price_oracle.rs:339-355`, `oracle/src/settings.rs:14-36`).
2. **Price ingestion** uses `set_price`, which validates timestamps, reconstructs per-asset prices from bitmask-compressed updates, emits events, updates bit history, writes temporary storage, and bumps TTLs (`oracle/src/price_oracle.rs:431-452`, `oracle/src/prices.rs:112-198`, `oracle/src/events.rs:12-41`).
3. **History queries** fetch single timestamps or rolling windows, including cross prices and TWAPs, and depend on resolution-based timestamp normalization plus cached records (`oracle/src/price_oracle.rs:150-317`, `oracle/src/prices.rs:16-260`).
4. **Economic knobs** include `extend_asset_ttl`, `set_fee_config`, and Beam `charge_invocation_fee`, all of which rely on the global fee config stored in instance state (`oracle/src/assets.rs:108-160`, `oracle/src/price_oracle.rs:403-418`, `beam-contract/src/cost.rs:43-95`).
5. **Upgrades/migrations** rely on protocol version markers and wasm-update calls (`oracle/src/protocol.rs:4-54`, `oracle/src/price_oracle.rs:465-468`).

## Threat Scenarios

### 1. Governance & Update Channel
- **Threat:** Admin multisig compromise or signer collusion lets attackers reconfigure assets, push malicious price batches, or upgrade to hostile code. This is the highest-impact failure because on-chain checks assume admin honesty (`oracle/src/price_oracle.rs:339-468`).
  **Controls:** Operationally enforced >50% multisig requirement (`doc.md:9-19`, `README.md:146-148`), timestamp validation (`oracle/src/price_oracle.rs:439-443`), and strict asset-limit checks.  
  **Residual Risk:** DAO processes and signer device security are off-chain assumptions; threat remains if quorum colludes. Consider defense-in-depth such as timelocks or dual governance for upgrades.

- **Threat:** Misconfiguration during initialization (e.g., wrong decimals or resolution) is irreversible because settings are write-once (`oracle/src/settings.rs:14-56`) and `config` has no rollback path.
  **Controls:** Requirement that admin signs `config` and asset uniqueness checks (`oracle/src/assets.rs:20-80`).  
  **Residual Risk:** Hard to detect until after deployment; incorporate pre-deployment simulations or mirrored staging environment.

### 2. Price Integrity & History Masking
- **Threat:** Incorrect or maliciously crafted bitmasks in `PriceUpdate` could desynchronize `update.mask` from `update.prices`, leading to stale or zero prices returned even though new data was stored (`oracle/src/prices.rs:61-88`).
  **Controls:** `set_price` compares `update.prices.len()` to asset count (`oracle/src/price_oracle.rs:433-438`), but it trusts the mask ordering.  
  **Residual Risk:** A compromised admin can still craft mismatched masks; downstream consumers should compare emitted events with expected asset order to detect anomalies.

- **Threat:** History bitmask drift (e.g., skipped timestamps or gaps) may cause `has_price` to return false despite stored data, degrading availability for TWAP/cross-price queries (`oracle/src/mapping.rs:3-72`, `oracle/src/prices.rs:112-143`).
  **Controls:** When gaps are detected, `update_history_mask` pads missing periods with zeroed updates before shifting masks (`oracle/src/prices.rs:118-134`).  
  **Residual Risk:** Long outages (>256 periods) still cause irrevocable history loss (`README.md:154-165`). Consider emitting additional metadata (e.g., cumulative counters) to cross-check bit history.

### 3. Protocol Upgrades & Migration
- **Threat:** Mixed protocol versions may read from v1 storage while writes go to v2, causing price retrieval failures or stale data if migration never completes (`oracle/src/prices.rs:33-58`, `oracle/src/protocol.rs:23-50`).
  **Controls:** Each read/write path checks `protocol::at_latest_protocol_version` and continues writing legacy entries until the scheduled upgrade finalizes after one day (`oracle/src/prices.rs:162-198`, `oracle/src/protocol.rs:34-50`).  
  **Residual Risk:** Requires liveness of admin to bump protocol markers; stalled upgrades could retain dead state indefinitely. Monitoring should alert when `protocol_update` is pending too long.

### 4. Economic Controls (TTL & Fees)
- **Threat:** Fee misconfiguration can disable TTL enforcement or allow denial-of-service if `FeeConfig::None` is set (no burn) or per-day fee is tiny (`oracle/src/settings.rs:83-100`). Assets would never expire or could be griefed by repeatedly burning minimal tokens.
  **Controls:** `extend_ttl` validates positive fee and amount and converts burn amount to milliseconds (`oracle/src/assets.rs:108-158`). `set_fee_config` requires admin auth and reinitializes expiration records (`oracle/src/price_oracle.rs:403-418`).  
  **Residual Risk:** Token economics sit off-chain; attackers that control the XRF token minting process could subsidize TTL bumps. Consider enforcing minimum fees or rate limits per asset.

- **Threat:** Beam invocation cost schedule can be altered by anyone because `set_invocation_costs_config` lacks an admin authorization check despite the comment implying one (`beam-contract/src/lib.rs:394-405`). Attackers could zero out costs (free-riding) or set exorbitant values, effectively DoS-ing Beam queries.
  **Controls:** None on-chain for this function; other Beam calls do require caller auth and burning fees (`beam-contract/src/lib.rs:173-320`).  
  **Residual Risk:** High & actionable; add `PriceOracleContractBase::admin` authorization before mutating the cost vector and consider emitting an event whenever pricing changes so clients can react.

### 5. Client-Facing Read APIs
- **Threat:** `fixed_div_floor` panics when dividend or divisor are non-positive, so malformed price data (e.g., zero quote asset) could brick cross-price or TWAP queries (`oracle/src/prices.rs:262-344`).
  **Controls:** `retrieve_asset_price_data` only returns prices if `has_price` is true for that timestamp (`oracle/src/prices.rs:31-58`), implicitly preventing zero values except when admin explicitly submits a zero price.  
  **Residual Risk:** Zero prices are considered valid updates (mask bit set but price 0). A malicious admin could submit zero quote prices to panic consumers. Downstream contracts should catch `Result`/`Option` failures and treat them as oracle faults.

- **Threat:** Staleness and replay on read paths - `obtain_last_record_timestamp` refuses to serve data older than 2 periods, so persistent outages make APIs return `None`, potentially breaking consumers that assume availability (`oracle/src/prices.rs:16-29`).
  **Controls:** Masks preserve up to 256 periods and events provide full history. Docs explicitly tell consumers to check timestamps (`README.md:158-167`).  
  **Residual Risk:** Consider exposing explicit freshness metadata (number of stale periods) so consumers can degrade gracefully instead of probing iteratively.

## Assumptions & Residual Risks
- Beam fee token client trusts the external XRF token contract and its burn semantics; compromised token logic can bypass fee burning entirely (`beam-contract/src/cost.rs:43-63`).
- Ledger timestamp honesty is assumed; Soroban provides this, but if validators manipulate timestamps then `set_price`'s `timestamp > ledger_timestamp` check can be bypassed (`oracle/src/price_oracle.rs:439-443`).
- Off-chain node software must deterministically compute prices and coordinate submissions; deviations aren't detectable on-chain beyond quorum failure (`doc.md:13-25`).
- Events are the only immutable audit log once in-contract history ages out; indexers must remain in sync to reconstruct >256-period history (`README.md:164-166`, `oracle/src/events.rs:12-41`).
