# Reflector audit details
- Total Prize Pool: $20,000 in USDC
    - HM awards: up to $17,280 in USDC
        - If no valid Highs or Mediums are found, the HM pool is $0
    - QA awards: $720 in USDC
    - Judge awards: $1,500 in USDC
    - Scout awards: $500 in USDC
- [Read our guidelines for more details](https://docs.code4rena.com/competitions)
- Starts October 27, 2025 20:00 UTC
- Ends November 11, 2025 20:00 UTC

### ❗ Important notes for wardens
1. Judging phase risk adjustments (upgrades/downgrades):
    - High- or Medium-risk submissions downgraded by the judge to Low-risk (QA) will be ineligible for awards.
    - Upgrading a Low-risk finding from a QA report to a Medium- or High-risk finding is not supported.
    - As such, wardens are encouraged to select the appropriate risk level carefully during the submission phase.

## Publicly known issues

_Anything included in this section is considered a publicly known issue and is therefore ineligible for awards._

- Contract admin can potentially invoke admin-level contract functions with some inconsistent/malformed arguments that may result in unexpected internal contract state. But it requires explicit approval from >50% of Reflector DAO members. It cannot be done simply by mistake, and therefore the contract code does not employ very strict validation rules in such cases.
- By design, it's normal to have a situation when an oracle does not have price data for a certain period in the past or reports "stale" data (updated more than 5 minutes ago). Reflector prioritizes safety over liveness. Consumer contracts must always check "timestamp" response field to decide whether the data is fresh enough.

✅ SCOUTS: Please format the response above 👆 so its not a wall of text and its readable.

# Overview

[ ⭐️ SPONSORS: add info here ]

## Links

- **Previous audits:**  https://github.com/reflector-network/reflector-contract/blob/master/audits/reflector_ottersec_audit_public_feed_2024.pdf
https://github.com/reflector-network/reflector-dao-contract/blob/master/audits/reflector_audit_certora_dao%2Bsubscriptions_2024.pdf
  - ✅ SCOUTS: If there are multiple report links, please format them in a list.
- **Documentation:** https://github.com/reflector-network/reflector-contract/tree/v3
- **Website:** https://reflector.network/
- **X/Twitter:** https://x.com/in_reflector 

---

# Scope

[ ✅ SCOUTS: add scoping and technical details here ]

### Files in scope
- ✅ This should be completed using the `metrics.md` file
- ✅ Last row of the table should be Total: SLOC
- ✅ SCOUTS: Have the sponsor review and and confirm in text the details in the section titled "Scoping Q amp; A"

*For sponsors that don't use the scoping tool: list all files in scope in the table below (along with hyperlinks) -- and feel free to add notes to emphasize areas of focus.*

| Contract | SLOC | Purpose | Libraries used |  
| ----------- | ----------- | ----------- | ----------- |
| [contracts/folder/sample.sol](https://github.com/code-423n4/repo-name/blob/contracts/folder/sample.sol) | 123 | This contract does XYZ | [`@openzeppelin/*`](https://openzeppelin.com/contracts/) |

### Files out of scope
✅ SCOUTS: List files/directories out of scope

# Additional context

## Areas of concern (where to focus for bugs)
- Data caching for the last several rounds
- Protocol upgrade transition (v1 stored data in individual temporary entries with the <asset,timestamp> key, v2 protocol stores all updates with the same timestamp in a single entry)
- Bit masks that store information whether a certain feed has been updated at a given period in the past

✅ SCOUTS: Please format the response above 👆 so its not a wall of text and its readable.

## Main invariants

- ReflectorPulse contract follows SEP-40 standard (https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0040.md), ReflectorBeam contract extends it with invocation sponsor.
- Only admin account can change contract settings, upgrade it, and publish prices.
- The admin account always has all cluster public keys as co-signers with >50% multisig threshold, meaning that any crucial state change requires DAO majority.
- A contract instance can only be initialized once.
- Oracle base asset, decimals, and timeframe never change after initialization.
- Each asset is unique, and can be added only once.
- Each oracle contract can support up to 256 assets and retain up to 256 historical update records.
- Each price feed (quoted asset) has expiration time; it stops receiving updates from Reflector cluster after the expiration.
- Anyone can bump price feed expiration time, paying for it with XRF tokens.
- Update timestamp can never be greater than current ledger timestamp.
- All public functions never panic. ReflectorBeam may panic only if the invocation commission charge fails.
- ReflectorBeam oracle caller address must have sufficient balance to pay for the invocation,
- Every price update emits a corresponding event.
- Price updates older than 256 periods can't be accessed through the contract, but can be loaded from the indexer by fetching update events.
- An oracle contract may have no price data for a certain period in the past or report "stale" data (updated more than 1 period ago) if there were no price movements during that time or cluster nodes failed to reach quorum.

✅ SCOUTS: Please format the response above 👆 so its not a wall of text and its readable.

## All trusted roles in the protocol

"admin" - an account that have permission to publish updates, change configuration, and upgrade the contract. Always has a M-of-N multisig configured (signers - Reflector DAO members), >50% of signers is required to execute any action.

✅ SCOUTS: Please format the response above 👆 using the template below👇

| Role                                | Description                       |
| --------------------------------------- | ---------------------------- |
| Owner                          | Has superpowers                |
| Administrator                             | Can change fees                       |

✅ SCOUTS: Please format the response above 👆 so its not a wall of text and its readable.

## Running tests

# check https://github.com/reflector-network/reflector-contract/tree/v3?tab=readme-ov-file#development
git clone -b v3 https://github.com/reflector-network/reflector-contract.git
cd ./reflector-contract
cargo build
cargo test

✅ SCOUTS: Please format the response above 👆 using the template below👇

```bash
git clone https://github.com/code-423n4/2023-08-arbitrum
git submodule update --init --recursive
cd governance
foundryup
make install
make build
make sc-election-test
```
To run code coverage
```bash
make coverage
```

✅ SCOUTS: Add a screenshot of your terminal showing the test coverage

## Miscellaneous
Employees of Reflector and Stellar Development Foundation and employees' family members are ineligible to participate in this audit.

Code4rena's rules cannot be overridden by the contents of this README. In case of doubt, please check with C4 staff.


# Scope

*See [scope.txt](https://github.com/code-423n4/2025-10-reflector/blob/main/scope.txt)*

### Files in scope


| File   | Logic Contracts | Interfaces | nSLOC | Purpose | Libraries used |
| ------ | --------------- | ---------- | ----- | -----   | ------------ |
| /beam-contract/src/cost.rs | ****| **** | 71 | ||
| /beam-contract/src/lib.rs | ****| **** | 143 | ||
| /oracle/src/assets.rs | ****| **** | 145 | ||
| /oracle/src/auth.rs | ****| **** | 19 | ||
| /oracle/src/events.rs | ****| **** | 32 | ||
| /oracle/src/lib.rs | ****| **** | 12 | ||
| /oracle/src/mapping.rs | ****| **** | 45 | ||
| /oracle/src/price_oracle.rs | ****| **** | 186 | ||
| /oracle/src/prices.rs | ****| **** | 251 | ||
| /oracle/src/protocol.rs | ****| **** | 37 | ||
| /oracle/src/settings.rs | ****| **** | 84 | ||
| /pulse-contract/src/lib.rs | ****| **** | 108 | ||
| /oracle/src/timestamps.rs | ****| **** | 18 | ||
| /oracle/src/types.rs | ****| **** | 50 | ||
| **Totals** | **** | **** | **1201** | | |

### Files out of scope

*See [out_of_scope.txt](https://github.com/code-423n4/2025-10-reflector/blob/main/out_of_scope.txt)*

| File         |
| ------------ |
| ./beam-contract/src/tests.rs |
| ./oracle/src/tests/mod.rs |
| ./oracle/src/tests/util_tests.rs |
| ./pulse-contract/src/tests/contract_admin_tests.rs |
| ./pulse-contract/src/tests/contract_interface_tests.rs |
| ./pulse-contract/src/tests/mod.rs |
| ./pulse-contract/src/tests/setup_tests.rs |
| Totals: 7 |

