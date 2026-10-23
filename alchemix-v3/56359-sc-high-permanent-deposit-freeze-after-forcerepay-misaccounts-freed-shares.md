# 56359 sc high permanent deposit freeze after forcerepay misaccounts freed shares

**Submitted on Oct 15th 2025 at 02:27:12 UTC by @Novathemachine for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56359
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

AlchemistV3 fails to release deposit-cap capacity after \_forceRepay(). When a liquidation burns vault shares, totalNormalizedDeposits remains pegged at the cap, so subsequent deposit() calls revert with IllegalState(). On mainnet, this locks all future collateral deposits for the affected yield token until governance intervenes.

## Vulnerability Details

* \_forceRepay() reduces account.debt, account.earmarked, and account.collateralBalance but never decreases the global counters that track deposits against depositCap (e.g., totalNormalizedDeposits / totalShares).
* The attacker opens a position that fills the vault’s deposit cap, then triggers liquidation. The liquidation burns shares, yet the cap tracker is unchanged.
* Any future deposit() call checks the unchanged cap counters and reverts, even though the actual shares were freed.
* Real integrations (Meta Yield Token vaults via Morpho V2) inherit the issue because AlchemistV3 is the canonical adapter.

Key files:

* src/AlchemistV3.sol::\_forceRepay()
* Test harness: src/test/AlchemistV3.t.sol
* PoC wrapper: src/test/poc/ForceRepayDepositCap.t.sol

## Impact Details

* Impact: Permanent freezing of deposits for the targeted yield token vault. No new collateral can enter until admins manually reset state, halting protocol growth and trapping would-be liquidity.
* Severity mapping: Immunefi v2.2 Critical (“Permanent freezing of funds”).
* A malicious actor can grief the system or stall growth by repeatedly filling caps and forcing liquidations.
* Fits program scope: AlchemistV3 is in-scope; no privileged access or excluded scenarios required.

## References

* <https://github.com/QuenumGerald/alc-v3-poc>
* README
* docs/force-repay-poc.md (full write-up, commands, trace)
* src/test/poc/ForceRepayDepositCap.t.sol
* src/test/AlchemistV3.t.sol::\_runDepositCapForceRepayPoc()

## Proof of Concept

## Proof of Concept

1. install dependencies
2. Run Poc locally : FOUNDRY\_PROFILE=default forge test\
   \--match-path src/test/poc/ForceRepayDepositCap.t.sol\
   \--match-test testPoC\_ForceRepayDepositCapFreeze\
   -vvvv --evm-version cancun
3. Observe tail of verbose trace:

"... AlchemistV3::deposit(...) \[delegatecall] ← \[Revert] IllegalState()"

* This happens immediately after \_forceRepay() burns shares. Even though collateral was removed, the deposit cap trackers remain at the original maximum, so the fresh deposit() call fails.

4. Understand the mechanics

* The PoC fills the vault by depositing up to its cap (totalNormalizedDeposits equals depositCap).
* The position is deliberately liquidated, invoking \_forceRepay().
* \_forceRepay() decreases account.debt and burns shares but never decrements the global cap counters (totalNormalizedDeposits / totalShares).
* Any subsequent deposit() reverts with IllegalState() because the system still believes the cap is full.
