# \#99606 [SC-Critical] Junior LP redemption funded before close-VPI rebate transfers value from remaining LPs to the exiting trader

**Submitted on Nov 14th 2026 at 13:08:15 UTC by @AGalx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99606
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An attacker who already holds Junior LP shares can request redemption, open a trade that pays **positive virtual price impact (VPI)** into `HousePool`, and have part of the redemption **funded and its shares burned at the VPI-inflated Junior valuation before the same trade closes**. On close, the negative VPI rebate can return the previously collected amount to the trader, but the shares already funded for redemption have been burned and cannot absorb the ensuing pool-value reduction. The remaining Junior LPs bear the corresponding loss. In matched local Foundry tests, funding the redemption before rather than after the close transfers **$1,853.953547 (0 bps confidence) or $1,733.062817 (10 bps)** of terminal value from the remaining Junior LP cohort to the exiting trader/LP. A fourth, no-claim witness confirms that **funding/share burn before close—not the subsequent ERC20 claim time—is the operative boundary**, with no oracle-price movement or privileged attacker action. If deployed with the tested configuration and sufficient liquidity, the sequence could extract existing Junior LP value through the settlement order alone.

## Vulnerability Details

### Root cause: redemption is funded on recognized positive VPI before the same position's offsetting close rebate is settled

The assessed source is the Plether Immunefi competition snapshot at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The affected *interaction* crosses `CfdEngineSettlementSidecar.sol` (trade cash flows and rebate payment), `HousePool.sol` (reconciliation, redemption pricing and funding), and `TrancheVault.sol` (pending redemption shares, burn, escrow and claim). The issue is not a price manipulation or an unauthorized transfer from a trader account: it concerns **when a future, contingent repayment associated with a trader’s VPI is recognized relative to an LP’s irreversible redemption funding**.

**1. Opening a skew-increasing position adds positive VPI to recognized pool cash and Junior NAV.** `CfdMath.calculateVPI()` returns a positive charge when the trade increases skew cost. `CfdEngineSettlementSidecar.executeOpen()` collects `tradeCostAssets` and records the resulting revenue arriving at the pool:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

In the PoC, Charlie's LONG pays positive opening VPI of **$3,497.726792 (0 bps)** or **$3,496.328471 (10 bps)**. The test asserts the corresponding increase in `HousePool.totalAssets()`, and that the Junior NAV rises before the redemption is funded. At 10 bps, the Engine also reports an **+$800 terminal LP price delta**. The PoC measures and retains that additional NAV component, rather than misattributing the entire NAV movement to VPI.

**2. An existing LP can prepare a redeem request while the shares still participate in the pricing supply.** `TrancheVault.requestRedeem()` transfers Charlie's shares into the vault escrow, but does not burn them:

```solidity
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
```

The PoC explicitly asserts that `totalSupply()` and Junior NAV remain unchanged by the request alone. Charlie can therefore request redemption after the normal deposit cooldown, wait for the public epoch to mature, and only then open the VPI-paying LONG shortly before settlement. This request is not the payout: no value is extracted until the pool **funds** the redemption and the vault pays the claim.

**3. The pool funds the maturing Junior redemption using the currently reconciled pricing state and available withdrawal budget.** `HousePool._settleLpEpoch()` reconciles its Engine input, captures Junior principal and supply, and computes the capacity left after senior obligations and other withdrawal constraints:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

When a Junior redemption is fundable, `HousePool._fundRedemptionPhase()` transfers the funded USDG to `TrancheVault` and invokes `fundRedeemEpoch()`. The vault increments `fundedShares` / `fundedAssets`, escrows the assets and **burns the funded shares**. The pool's liquidity protections can limit the first funding but, in the reproduced attack, do not eliminate this pre-close funding. The PoC handles any remaining shares with a second settlement after the close. It counts only actually paid USDG and any residual assets once, not a withdrawal preview.

**4. The close reimburses VPI after some Junior ownership has already exited.** When a close generates an action rebate that can be paid, `CfdEngineSettlementSidecar._executeClose()` transfers cash out of `HousePool` and credits Charlie's trader settlement balance:

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

In the attack witness, the recorded gross close VPI is the negative of Charlie's positive opening VPI, reaching the lifetime rebate clamp; the test reconciles the actual credited rebate after PnL, fees and carry. **The already-funded redemption cannot be repriced after this rebate leaves the pool.** Bob's remaining Junior shares therefore bear the reduction that Charlie would otherwise have shared by remaining LP or funding redemption after close.

**Broken economic invariant (candidate):** an LP should not be able to realize a larger, irreversibly funded withdrawal simply by placing that funding between its own positive VPI payment and the corresponding close rebate, when the same final trading exposure and market prices are used in the control. This is a **guard-to-settlement / cohort-exit accounting differential**, not merely an intermediate NAV fluctuation.

### Permissionless sequence reproduced locally

1. **Initial LPs:** Charlie deposits **6,000,000 USDG** into Junior, and Bob independently deposits **4,000,000 USDG**. The two requests are activated through the normal LP epoch. Charlie is already a Junior LP before opening his trade. The fixture applies the documented launch Senior caps (**40M USDG and 80%**) in `setUp()`; that owner-level setup is **not** an attacker action.
2. **Independent skew:** Alice has **100,000 USDG** of trader collateral and opens a **3,000,000 USDG LONG** through the public Router. She remains passive and her position remains open throughout Charlie's attack. Charlie also begins with **100,000 USDG** trader collateral.
3. **Pre-queue withdrawal:** After the normal cooldown, Charlie calls `TrancheVault.requestRedeem()` for all his Junior shares. The shares are escrowed but remain included in `totalSupply()` until funded.
4. **VPI-positive open:** Charlie opens a **1,000,000 USDG LONG**, increasing market skew. A permissionless keeper executes the order. Positive VPI is collected, increasing pool cash and the value used for the Junior redemption quote. The oracle mark remains **$1.00**.
5. **Irreversible redemption before close:** Once the epoch is mature, the public `OrderRouter.settleLpEpoch()` funds available Charlie shares at this pre-close price. `TrancheVault.fundRedeemEpoch()` burns the funded shares and fixes their USDG claim. In the attack witness Charlie claims promptly; in an additional witness he deliberately waits to claim until **after** his close. Both outcomes are economically identical.
6. **Rebate-producing close:** Charlie closes the complete LONG through the public Router, using valid oracle updates at the same mark. The protocol pays the allowable negative close VPI to Charlie's trader account from the pool. Bob stays invested.
7. **Remainder and final wealth:** Charlie settles and claims any residual redemption after close. The strengthened test now **asserts unconditionally** that attack, close-first and claim-after-close scenarios finish with **zero residual shares/refundable shares, zero remaining claimable USDG and all requested shares funded/burned**. Each claim is reconciled against observed Charlie ERC20 balance increases. Wealth is computed from wallet cash, Clearinghouse balance, residual shares and still-claimable assets without double-counting.

The Pyth test doubles are used only for oracle updates; no fabricated signed oracle tick or privileged price-signer capability is assumed. Alice does not cooperate after opening her LONG. The attack requires existing LP liquidity, sufficient withdrawal capacity for some pre-close funding, the tested market skew and sufficient trader capital. The PoC's time warps simulate legitimate cooldown/epoch passage, not an ability to bypass timestamps on chain.

### Matched controls isolate the timing effect

The supplied Foundry test restores **the same fixture** for **four matched sequences** at both 0 and 10 bps confidence, implemented inside two `test_...` methods:

- `ExitAfterOpenBeforeClose` (**attack**): first redemption funding/share burn and claim occur before Charlie closes, with the residual redemption funded after close.
- `CloseBeforeExitFunding` (**timing control**): the same trade is closed first, and only then are the redemption shares funded.
- `FundBeforeCloseClaimAfterClose` (**new no-claim control**): first funding/burn occurs before close, but the fixed USDG claim is deliberately left unclaimed until after close. Its terminal Charlie wealth **and Bob's remaining LP value match the attack exactly** at each confidence level; the initial claim remains fixed across close.
- `RemainLpThroughClose` (**exposure control**): Charlie stays invested through close, with residual share value counted exactly once. Its previously recorded outcome agrees with close-first within **$0.000002**.

The updated test **unconditionally asserts** full funding/burning of requested shares, no residual/refundable shares, zero unpaid USDG claims, actual ERC20 payout reconciliation, and correct funding/close timestamp ordering. It captures per-event HousePool assets, Junior NAV/supply/share conversion, share burns, escrow claims, Charlie cash received, Bob share value, free assets and liability. Close timestamp and Pyth inputs are matched between controls: price `100000000` at exponent `-8` ($1), publish/close time `1709802002`, and the respective confidence `0` or `100000`.

The new checkpoint reports **2 passed, 0 failed, 0 skipped** with Foundry 1.5.1-stable and Solc 0.8.35. These are the two top-level tests; each exercises the four scenarios. The execution is local with the repository's Pyth/USDG doubles. This report was revised from the supplied checkpoint and PoC; no independent Foundry rerun is claimed.

### Prior-audit kill pass and the distinct mechanism demonstrated here

The prior-audit review was conducted separately from the PoC. The most relevant overlapping public scan lead is **“Temporary pool-liquidity reduction can amplify a skew-reducing close rebate”** (`audits/scan-plether-fi-plether-core-d4541b49.md`, lines 1742–1808). The analysis below compares this specific proposed mechanism with the reproduced behavior; it does not claim that the scan lead has been officially confirmed as an Immunefi known issue.

**That scan lead's alleged mechanism:** reducing HousePool liquidity before a skew-reducing close can increase the *magnitude of the negative close VPI rebate* through the pool-depth calculation. Its relevant consequence is an **extra rebate paid to the trader**.

**HYP-25's mechanism:** an existing Junior LP has its redemption *funded and shares burned* when positive VPI already received by the pool is reflected in the share valuation. That same trader subsequently closes, recovering its allowed VPI rebate **after** the funded shares have exited. The surviving LP cohort absorbs that reduction. The affected invariant is **economic cohort allocation at irreversible redemption funding**, not merely the size of the close rebate. A pre-close liquidity reduction also occurs in our test, so overlapping steps are acknowledged rather than hidden.

The hardened PoC measures both mechanisms independently:


**Oracle confidence: 0 bps**

- Actual close rebate: funding-first: $3,497.726792
- Actual close rebate: close-first: $3,496.503861
- Additional close rebate: **$1.222931**
- Charlie's full timing uplift: **$1,853.953547**

**Oracle confidence: 10 bps**

- Actual close rebate: funding-first: $3,496.328471
- Actual close rebate: close-first: $3,496.328471
- Additional close rebate: **$0.000000**
- Charlie's full timing uplift: **$1,733.062817**


At **10 bps**, the gross VPI and *actual credited close rebate* are **exactly equal in the attack and close-first control**, yet **1,733.062817 USDG** moves from the surviving Junior cohort to Charlie under the paired terminal-wealth measurement. Thus the scan lead's extra-rebate mechanism **cannot explain the measured extraction** in that witness. At **0 bps**, the $1.222931 rebate difference is only about **0.066%** of the $1,853.953547 timing uplift; the bulk remains a funding/cohort-allocation difference.

The additional `FundBeforeCloseClaimAfterClose` witness funds and burns the *same* shares before close but deliberately defers the ERC20 claim until after close. Its final Charlie wealth and remaining-LP value are **identical** to the immediate-claim attack at both oracle confidence settings; the funded USDG entitlement remains fixed across the close. **Share funding/burning, not claim timing, is the relevant settlement boundary.**

## Impact Details

**Proposed in-scope impact:** *Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield*, if the platform accepts the demonstrated change in remaining Junior LP wealth as an improper transfer of principal/economic ownership. The PoC establishes a local counterfactual loss in the remaining LP cohort and an equal attacker uplift, not an arbitrary debit from Bob's wallet. Final validity and severity remain subject to triage.

### Measured outcomes (USDG-dollar equivalents)


**Pyth confidence: 0 bps**

- Charlie absolute wealth delta — attack: +$3,991.567380
- Charlie absolute wealth delta — close-first: +$2,137.613833
- Attacker uplift caused by timing: **+$1,853.953547**
- Remaining LP cohort's decrease vs control: **−$1,853.953547**
- Actually paid across attack redemptions: $6,004,793.898797

**Pyth confidence: 10 bps**

- Charlie absolute wealth delta — attack: +$4,070.074766
- Charlie absolute wealth delta — close-first: +$2,337.011949
- Attacker uplift caused by timing: **+$1,733.062817**
- Remaining LP cohort's decrease vs control: **−$1,733.062817**
- Actually paid across attack redemptions: $6,005,272.405189


**Do not confuse these amounts:** the absolute wealth increases include normal trading economics and other transfers that also exist in the controls. The claimed extraction is **the incremental terminal wealth transferred by settling the LP redemption before the close rather than afterward**, with a corresponding difference in the remaining LP cohort's final share value. The gross ~$6 million redemption payout includes return of Charlie's own principal and is **not** the amount stolen.

The hardened test now **requires** zero residual Charlie position, zero pending/refundable redemption shares and zero claimable USDG for attack, close-first and no-claim scenarios. The first and second funded shares must total the requested shares; each funding must match the share-supply burn, and the paid claims must match Charlie’s observed ERC20 balance change. Alice’s independent LONG remains unchanged. The starting market and LP balances are held constant across the paired cases; the result does not require favorable oracle movement. The no-claim scenario confirms that **funding/burn rather than claim timing** fixes the economic outcome.

### Funding and cash settlement proof

The first funding is partial because the pool preserves liquidity against open trader liability; the rest is funded after Charlie closes. Importantly, these are **actual paid USDG amounts**, not only redemption previews:


**Confidence: 0 bps**

- First funded payout (attack): $5,999,996.826971
- Second payout after close: $4,797.071826
- Total paid (attack): **$6,004,793.898797**
- Total paid (close-first control): $6,002,941.167295

**Confidence: 10 bps**

- First funded payout (attack): $6,000,795.629190
- Second payout after close: $4,476.775999
- Total paid (attack): **$6,005,272.405189**
- Total paid (close-first control): $6,003,539.341486


The `FundBeforeCloseClaimAfterClose` witness receives the same totals as the attack, but intentionally defers claiming the first USDG escrow entitlement until after close. The test asserts the funded claim's amount did not change across close. Therefore the irreversible **redemption funding/share burn** is the relevant boundary. The difference in these total payouts should not be confused with gross wealth or standalone profit because the final comparison also accounts for Charlie's trading cash flows and any residual claims.

### Cost model and economic limits

The preserved attack-path measurements are **5,471,497 / 5,512,417 EVM execution gas** across **nine transaction-shaped actions** at 0 / 10 bps, including permissionless keeper executions charged to Charlie. The illustrative Arbitrum fee model assumes **four 2 KB Pyth updates**, **21,000 intrinsic gas per transaction**, a **50% L2 execution margin**, and a **10%-padded 23,115-gas L1-data allowance** based on an earlier NodeInterface sample, not on deployed Plether transactions. This yields approximately **8,516,173 / 8,577,553 modeled gas**.


**Pyth confidence: 0 bps**

- Timing uplift before network cost: $1,853.953547
- Modeled network cost: $0.357679
- Timing uplift after modeled gas: **+$1,853.595868**

**Pyth confidence: 10 bps**

- Timing uplift before network cost: $1,733.062817
- Modeled network cost: $0.360257
- Timing uplift after modeled gas: **+$1,732.702560**


The assumptions use **0.021 gwei** as the rounded Arbitrum gas price and **$2,000 per ETH**. At 10 bps, the approximate break-even gas price under the modeled gas units is **101 gwei**. Arbitrum incorporates both L2 execution and L1 data posting into a single network fee. Pyth Core currently documents a zero on-chain EVM update fee; any external price-data acquisition or production-specific integration costs are not measured. These figures are **models, not actual deployed-instance transaction receipts**. The model also excludes the funding or opportunity cost of Charlie's **6 million USDG LP principal**, and does not prove that comparable withdrawal capacity and skew are always available.

### Constraints and demonstrated scope

- **Code/roles:** all vulnerable paths identified in the report are in `packages/perps/src`; the PoC makes legitimate owner-level configuration changes **only in fixture setup** to match the documented launch caps, not as an attacker step. Charlie's sequence uses LP and trader functions available to ordinary accounts and public keeper paths.
- **Market:** flat $1.00 mock Pyth prices at 0 and 10 bps confidence; no oracle signer compromise, no deliberately stale public tick, and no Bob/Alice collusion after independent setup.
- **Liquidity:** first funding is partial; the remaining shares are funded after close. The test strictly asserts zero residual shares/claim, matching share burns, and actual paid ERC20 cash. The no-claim witness proves that claims may occur after close without eliminating the timing differential.
- **Deployment:** this is a local source-contest reproduction. No live Arbitrum One exploitation, prevalence estimate or fixed release is asserted. The same-period buy/sell execution rules and available pool budget must hold in any real deployment.

## References

**Assessed competition source:** [`a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8); the target file most directly responsible for the irreversible redemption funding is [`HousePool.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol).

- [CfdMath: sign convention for VPI](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L142-L179)
- [CfdEngineSettlementSidecar: positive-open VPI cash collection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464)
- [TrancheVault: redeem request escrows shares without burning them](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L630)
- [HousePool: reconcile, redemption snapshots and Junior funding capacity](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L888)
- [HousePool: priced and budgeted redemption funding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L1027)
- [TrancheVault: funded assets enter escrow and funded shares burn](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1478-L1502)
- [TrancheVault: claimRedeem entrypoint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1014-L1020)
- [CfdEngineSettlementSidecar: close rebate paid from pool to trader](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624)
- [CfdEngineSettlementSidecar: lifetime VPI accrual updated on close](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L636-L650)
- [CfdEngineProtocolLens: HousePool accounting snapshot](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L49-L73)

- [Official Plether audit-competition scope and PoC requirements](https://immunefi.com/audit-competition/plether/scope/)
- [Public Plether pre-contest AI scan (depth/rebate known-issue comparison)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md)
- [Arbitrum official gas estimation guide](https://docs.arbitrum.io/arbitrum-essentials/how-to-estimate-gas)
- [Arbitrum NodeInterface reference](https://docs.arbitrum.io/arbitrum-essentials/nodeinterface/reference)
- [Pyth Core EVM on-chain fee documentation](https://docs.pyth.network/pro/on-chain-fees)

**Local PoC:** `Hyp25PositiveVpiJuniorExit.t.sol`, reproduced in the full report under `## Proof of Concept`. Save it in the locked checkout as `packages/perps/test/perps/accounting/close/Hyp25PositiveVpiJuniorExit.t.sol`. It is the four-scenario version supplied on 14 November 2026; production source files remain unchanged.


## Proof of Concept

## Proof of Concept

**File:** `packages/perps/test/perps/accounting/close/Hyp25PositiveVpiJuniorExit.t.sol`  
**Locked commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`  
**Foundry:** 1.5.1-stable; **Solc:** 0.8.35.

From the repository root:

```bash
forge test --root packages/perps --match-path test/perps/accounting/close/Hyp25PositiveVpiJuniorExit.t.sol -vv
```

The supplied local checkpoint reports **2 passed, 0 failed, 0 skipped**. Each test includes four snapshot-matched scenarios. The complete code below reproduces the supplied test with **only cosmetic HYP-25 label normalization** (one comment, two test method names, and two log messages). No execution logic, assertions, scenarios, or parameters were changed. The original source had a reported 2/2 passing local run; the cosmetically relabeled version has not been rerun in this session. Only the repository's pre-existing USDG/Pyth test doubles are used; production contracts are unchanged. The runtime results quoted in the description are based on the researcher's local run, not an independent rerun in this session.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice Local production-route test for HYP-25 and matched timing/exposure witnesses.
/// @dev Only the repository's USDG and Pyth test doubles are used; in-scope contracts remain real.
contract Hyp25PositiveVpiJuniorExitTest is BasePerpTest {

    address internal constant BOB = address(0xB0B);
    address internal constant CHARLIE = address(0xC4A2);
    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0x4B33);

    uint256 internal constant CHARLIE_LP_CAPITAL = 6_000_000e6;
    uint256 internal constant BOB_LP_CAPITAL = 4_000_000e6;
    uint256 internal constant TRADER_CAPITAL = 100_000e6;
    uint256 internal constant ALICE_LONG = 3_000_000e18;
    uint256 internal constant CHARLIE_LONG = 1_000_000e18;
    uint256 internal constant RECONCILIATION_TOLERANCE = 100_000; // $0.10 USDG

    enum Sequence {
        ExitAfterOpenBeforeClose,
        CloseBeforeExitFunding,
        RemainLpThroughClose,
        FundBeforeCloseClaimAfterClose
    }

    struct EconomicSnapshot {
        uint256 poolAssets;
        uint256 juniorNav;
        uint256 juniorSupply;
        uint256 juniorAssetsPerShare;
        uint256 sharesBurnedAtEvent;
        uint256 sharesBurnedCumulative;
        uint256 charlieUsdGReceived;
        uint256 claimableUsdG;
        uint256 pendingRedeemShares;
        uint256 claimableRedeemShares;
        uint256 bobShareValue;
        uint256 freeAssets;
        uint256 maxLiability;
        uint64 timestamp;
        uint64 oraclePublishTime;
        uint64 oracleConfidence;
        int64 oraclePriceRaw;
        int32 oracleExponent;
    }

    struct Metrics {
        EconomicSnapshot afterOpenVpi;
        EconomicSnapshot afterFirstFunding;
        EconomicSnapshot afterFirstClaim;
        EconomicSnapshot beforeCloseVpi;
        EconomicSnapshot afterCloseVpi;
        EconomicSnapshot afterSecondFunding;
        EconomicSnapshot afterSecondClaim;
        uint256 poolBeforeCharlieOpen;
        uint256 poolAfterCharlieOpen;
        uint256 poolBeforeCharlieClose;
        uint256 poolAfterCharlieClose;
        uint256 juniorNavAfterCharlieOpen;
        uint256 freeAssetsAfterCharlieOpen;
        uint256 maxJuniorWithdrawAfterCharlieOpen;
        uint256 maxLiabilityAfterCharlieOpen;
        uint256 settlementBufferAfterCharlieOpen;
        int256 terminalNavPnlAfterCharlieOpen;
        int256 nonVpiNavAdjustmentAfterCharlieOpen;
        uint256 juniorNavBeforeRequest;
        uint256 juniorNavAfterRequest;
        uint256 juniorSupplyBeforeRequest;
        uint256 juniorSupplyAfterRequest;
        uint256 charlieSharesRequested;
        uint256 charlieSharesEscrowed;
        uint256 redeemEpoch;
        uint256 redeemActivation;
        uint256 redemption1FundedShares;
        uint256 redemption1Paid;
        uint256 redemption1ClaimableAtFunding;
        uint256 redemption2FundedShares;
        uint256 redemption2Paid;
        uint256 redemption2ClaimableAtFunding;
        uint256 redemptionResidualShares;
        uint256 redemptionResidualClaimAssets;
        uint256 charlieWalletBalanceBeforeRedemption;
        uint256 charlieWalletBalanceDeltaObserved;
        uint256 charlieStartWealth;
        uint256 charlieEndWealth;
        uint256 charlieRemainingShareValue;
        uint256 bobEndWealth;
        uint256 oldLpShares;
        uint256 oldLpAssetsFinal;
        uint256 juniorAssetsFinal;
        uint256 charlieOpenFee;
        uint256 charlieCloseFee;
        uint256 charlieOpenBounty;
        uint256 charlieCloseBounty;
        uint256 charlieOpenActionChargeCollected;
        uint256 charlieCloseActionChargeCollected;
        uint256 charlieCloseRebatePaid;
        uint256 closeOraclePriceRaw;
        uint256 closeOracleConfidence;
        uint256 closeOraclePublishTime;
        int32 closeOracleExponent;
        uint256 setupGas;
        uint256 redeemRequestGas;
        uint256 charlieOpenCommitGas;
        uint256 charlieCloseCommitGas;
        uint256 charlieOpenGas;
        uint256 charlieCloseGas;
        uint256 lpSettlementGas;
        uint256 redeemClaimGas;
        int256 charlieOpenVpi;
        int256 aliceOpenVpi;
        int256 charlieCloseVpi;
        int256 charlieOpenCarry;
        int256 charlieCloseCarry;
        int256 charlieClosePnl;
        int256 charlieCloseBalanceDelta;
        uint64 closeTimestamp;
        uint256 charliePositionSizeFinal;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "launch Senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "launch Senior share cap");
    }

    function test_Hyp25_MatchedScenarios_ZeroConfidence() public {
        _compareScenarios(0);
    }

    function test_Hyp25_MatchedScenarios_TenBpsConfidence() public {
        _compareScenarios(100_000);
    }

    function _compareScenarios(
        uint64 confidence
    ) internal {
        uint256 baseline = vm.snapshotState();
        Metrics memory attack = _runScenario(Sequence.ExitAfterOpenBeforeClose, confidence);
        assertTrue(vm.revertToStateAndDelete(baseline), "restore exact fixture before temporal witness");

        uint256 timingBaseline = vm.snapshotState();
        Metrics memory timing = _runScenario(Sequence.CloseBeforeExitFunding, confidence);
        assertTrue(vm.revertToStateAndDelete(timingBaseline), "restore exact fixture before no-claim witness");

        uint256 noClaimBaseline = vm.snapshotState();
        Metrics memory noClaim = _runScenario(Sequence.FundBeforeCloseClaimAfterClose, confidence);
        assertTrue(vm.revertToStateAndDelete(noClaimBaseline), "restore exact fixture before exposure witness");

        uint256 exposureBaseline = vm.snapshotState();
        Metrics memory exposure = _runScenario(Sequence.RemainLpThroughClose, confidence);
        assertTrue(vm.revertToStateAndDelete(exposureBaseline), "restore exact fixture after exposure witness");

        _logMetrics("attack_exit_after_open", attack, confidence);
        _logMetrics("temporal_witness_close_first", timing, confidence);
        _logMetrics("fund_before_close_claim_after_close", noClaim, confidence);
        _logMetrics("exposure_witness_remain_lp", exposure, confidence);

        int256 charlieUpliftVsTiming = int256(attack.charlieEndWealth) - int256(timing.charlieEndWealth);
        int256 oldCohortLossVsTiming = int256(timing.oldLpAssetsFinal) - int256(attack.oldLpAssetsFinal);
        int256 reconciliation = charlieUpliftVsTiming - oldCohortLossVsTiming;
        int256 charlieUpliftVsNoClaim = int256(noClaim.charlieEndWealth) - int256(timing.charlieEndWealth);
        int256 oldCohortLossVsNoClaim = int256(timing.oldLpAssetsFinal) - int256(noClaim.oldLpAssetsFinal);
        int256 charlieUpliftVsExposure = int256(attack.charlieEndWealth) - int256(exposure.charlieEndWealth);
        int256 oldCohortLossVsExposure = int256(exposure.oldLpAssetsFinal) - int256(attack.oldLpAssetsFinal);

        emit log_named_int("hyp25.charlie_uplift_vs_close_first", charlieUpliftVsTiming);
        emit log_named_int("hyp25.old_cohort_loss_vs_close_first", oldCohortLossVsTiming);
        emit log_named_int("hyp25.reconciliation_difference", reconciliation);
        emit log_named_int("hyp25.charlie_uplift_no_claim_vs_close_first", charlieUpliftVsNoClaim);
        emit log_named_int("hyp25.old_cohort_loss_no_claim_vs_close_first", oldCohortLossVsNoClaim);
        emit log_named_int(
            "hyp25.attack_vs_close_first_gross_close_vpi_delta", attack.charlieCloseVpi - timing.charlieCloseVpi
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_actual_close_rebate_delta",
            int256(attack.charlieCloseRebatePaid) - int256(timing.charlieCloseRebatePaid)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_pool_before_close_delta",
            int256(attack.beforeCloseVpi.poolAssets) - int256(timing.beforeCloseVpi.poolAssets)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_pool_after_close_delta",
            int256(attack.afterCloseVpi.poolAssets) - int256(timing.afterCloseVpi.poolAssets)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_free_assets_before_close_delta",
            int256(attack.beforeCloseVpi.freeAssets) - int256(timing.beforeCloseVpi.freeAssets)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_free_assets_after_close_delta",
            int256(attack.afterCloseVpi.freeAssets) - int256(timing.afterCloseVpi.freeAssets)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_liability_before_close_delta",
            int256(attack.beforeCloseVpi.maxLiability) - int256(timing.beforeCloseVpi.maxLiability)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_liability_after_close_delta",
            int256(attack.afterCloseVpi.maxLiability) - int256(timing.afterCloseVpi.maxLiability)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_close_timestamp_delta",
            int256(uint256(attack.closeTimestamp)) - int256(uint256(timing.closeTimestamp))
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_oracle_price_raw_delta",
            int256(attack.closeOraclePriceRaw) - int256(timing.closeOraclePriceRaw)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_oracle_confidence_delta",
            int256(attack.closeOracleConfidence) - int256(timing.closeOracleConfidence)
        );
        emit log_named_int(
            "hyp25.attack_vs_close_first_oracle_publish_time_delta",
            int256(attack.closeOraclePublishTime) - int256(timing.closeOraclePublishTime)
        );
        uint256 closeVpiDelta = _abs(attack.charlieCloseVpi - timing.charlieCloseVpi);
        uint256 closeVpiDeltaOfUpliftBps =
            charlieUpliftVsTiming > 0 ? closeVpiDelta * 10_000 / uint256(charlieUpliftVsTiming) : 0;
        emit log_named_uint("hyp25.close_vpi_delta_assets", closeVpiDelta);
        emit log_named_uint("hyp25.close_vpi_delta_as_uplift_bps", closeVpiDeltaOfUpliftBps);
        emit log_named_int("hyp25.charlie_uplift_vs_remain_lp", charlieUpliftVsExposure);
        emit log_named_int("hyp25.old_cohort_loss_vs_remain_lp", oldCohortLossVsExposure);

        _assertFullyFundedAndPaid("attack", attack);
        _assertFullyFundedAndPaid("close-first", timing);
        _assertFullyFundedAndPaid("fund-before-close/no-claim", noClaim);
        assertGt(charlieUpliftVsTiming, 0, "late exit after positive VPI increases Charlie's settled wealth");
        assertGt(oldCohortLossVsTiming, 0, "matched old Junior cohort loses the same value");
        assertGt(attack.charlieOpenVpi, 0, "Charlie pays positive open VPI");
        assertEq(attack.charliePositionSizeFinal, 0, "attack closes Charlie's whole position");
        assertEq(timing.charliePositionSizeFinal, 0, "timing witness closes Charlie's whole position");
        assertEq(noClaim.charliePositionSizeFinal, 0, "no-claim witness closes Charlie's whole position");
        assertEq(exposure.charliePositionSizeFinal, 0, "exposure witness closes Charlie's whole position");
        assertLt(
            attack.afterFirstFunding.timestamp,
            attack.closeTimestamp,
            "attack funds and burns shares before Charlie's close"
        );
        assertLt(
            noClaim.afterFirstFunding.timestamp,
            noClaim.closeTimestamp,
            "no-claim witness funds and burns shares before Charlie's close"
        );
        assertGt(
            timing.afterFirstFunding.timestamp,
            timing.closeTimestamp,
            "close-first witness funds and burns shares only after Charlie's close"
        );

        assertApproxEqAbs(
            _abs(charlieUpliftVsTiming),
            _abs(oldCohortLossVsTiming),
            RECONCILIATION_TOLERANCE,
            "Charlie timing uplift reconciles to old LP cohort loss"
        );
        assertEq(
            noClaim.charlieEndWealth,
            attack.charlieEndWealth,
            "claiming funded USDG after close preserves the attack's terminal wealth"
        );
        assertEq(
            noClaim.oldLpAssetsFinal, attack.oldLpAssetsFinal, "claim timing alone does not change remaining LP value"
        );
        assertEq(noClaim.charlieCloseVpi, attack.charlieCloseVpi, "claim timing does not change close VPI");
        assertEq(
            noClaim.charlieCloseRebatePaid,
            attack.charlieCloseRebatePaid,
            "claim timing does not change actual close rebate paid"
        );
        assertEq(
            noClaim.beforeCloseVpi.poolAssets,
            attack.beforeCloseVpi.poolAssets,
            "unclaimed funded USDG remains outside HousePool just like the claimed payout"
        );
        assertEq(noClaim.redeemEpoch, attack.redeemEpoch, "same matured LP redemption epoch");
        assertEq(noClaim.charlieSharesRequested, attack.charlieSharesRequested, "same requested share amount");
        assertEq(
            noClaim.redemption1FundedShares,
            attack.redemption1FundedShares,
            "same first settlement funds and burns the same shares"
        );
        assertEq(
            noClaim.redemption1ClaimableAtFunding,
            attack.redemption1ClaimableAtFunding,
            "same fixed USDG claim after the first funding"
        );
        assertEq(
            noClaim.afterFirstFunding.poolAssets,
            attack.afterFirstFunding.poolAssets,
            "same HousePool cash immediately after first funding"
        );
        assertEq(
            noClaim.afterFirstFunding.juniorNav,
            attack.afterFirstFunding.juniorNav,
            "same Junior NAV immediately after first funding"
        );
        assertEq(
            noClaim.afterFirstFunding.juniorSupply,
            attack.afterFirstFunding.juniorSupply,
            "same Junior supply after first burn"
        );
        assertEq(
            noClaim.afterFirstFunding.sharesBurnedAtEvent,
            attack.afterFirstFunding.sharesBurnedAtEvent,
            "same number of shares burned by first funding"
        );
        assertEq(
            noClaim.afterFirstFunding.bobShareValue,
            attack.afterFirstFunding.bobShareValue,
            "same Bob share value after first funding"
        );
        assertEq(attack.closeTimestamp, timing.closeTimestamp, "attack and close-first close timestamps match");
        assertEq(attack.closeTimestamp, noClaim.closeTimestamp, "attack and no-claim close timestamps match");
        assertEq(attack.closeOraclePriceRaw, timing.closeOraclePriceRaw, "matched close oracle price");
        assertEq(attack.closeOraclePriceRaw, noClaim.closeOraclePriceRaw, "no-claim close oracle price");
        assertEq(attack.closeOracleConfidence, timing.closeOracleConfidence, "matched close confidence");
        assertEq(attack.closeOracleConfidence, noClaim.closeOracleConfidence, "no-claim close confidence");
        assertEq(attack.closeOraclePublishTime, timing.closeOraclePublishTime, "matched close oracle publish time");
        assertEq(attack.closeOraclePublishTime, noClaim.closeOraclePublishTime, "no-claim close publish time");
        assertEq(attack.closeOracleExponent, timing.closeOracleExponent, "matched close oracle exponent");
        assertEq(attack.closeOracleExponent, noClaim.closeOracleExponent, "no-claim close oracle exponent");
        assertEq(
            attack.closeOraclePublishTime,
            attack.closeTimestamp,
            "attack close uses the recorded Pyth publish timestamp"
        );
        assertEq(
            timing.closeOraclePublishTime,
            timing.closeTimestamp,
            "close-first close uses the recorded Pyth publish timestamp"
        );
        assertEq(
            noClaim.closeOraclePublishTime,
            noClaim.closeTimestamp,
            "no-claim close uses the recorded Pyth publish timestamp"
        );

        if (confidence == 100_000) {
            assertEq(
                attack.charlieCloseVpi,
                timing.charlieCloseVpi,
                "at 10 bps, attack and close-first gross close VPI are identical"
            );
            assertEq(
                attack.charlieCloseRebatePaid,
                timing.charlieCloseRebatePaid,
                "at 10 bps, attack and close-first actual close rebates are identical"
            );
        }

        if (charlieUpliftVsTiming > 0 && oldCohortLossVsTiming > 0) {
            emit log_string("HYP-25 local economic differential: reproduced");
        } else {
            emit log_string("HYP-25 local economic differential: not reproduced by this fixture");
        }
    }

    function _assertFullyFundedAndPaid(
        string memory label,
        Metrics memory m
    ) internal view {
        assertEq(m.redemptionResidualShares, 0, string.concat(label, ": no residual or refundable shares"));
        assertEq(m.redemptionResidualClaimAssets, 0, string.concat(label, ": no residual USDG claim"));
        assertEq(
            m.redemption1FundedShares + m.redemption2FundedShares,
            m.charlieSharesRequested,
            string.concat(label, ": all requested shares are funded and burned")
        );
        assertEq(
            m.afterFirstFunding.sharesBurnedAtEvent,
            m.redemption1FundedShares,
            string.concat(label, ": first funded shares equal the first share-supply burn")
        );
        assertEq(
            m.afterSecondFunding.sharesBurnedAtEvent,
            m.redemption2FundedShares,
            string.concat(label, ": second funded shares equal the second share-supply burn")
        );
        assertEq(
            m.redemption1Paid + m.redemption2Paid,
            m.charlieWalletBalanceDeltaObserved,
            string.concat(label, ": recorded claims equal Charlie's actual ERC20 balance increase")
        );
    }

    function _runScenario(
        Sequence sequence,
        uint64 confidence
    ) internal returns (Metrics memory m) {
        uint256 beforeSetupGas = gasleft();
        _createJuniorCohort();
        _fundTraderHyp(CHARLIE, TRADER_CAPITAL);
        _fundTraderHyp(ALICE, TRADER_CAPITAL);
        m.setupGas = beforeSetupGas - gasleft();
        m.charlieWalletBalanceBeforeRedemption = settlementToken.balanceOf(CHARLIE);

        uint256 charlieShares = juniorVault.balanceOf(CHARLIE);
        m.oldLpShares = juniorVault.totalSupply() - charlieShares;
        m.charlieStartWealth = _charlieWealth(0, charlieShares);
        assertGt(charlieShares, 0, "Charlie is a funded Junior LP before the attack");

        OrderV3Types.OrderReceipt memory aliceOpen =
            _executeOrder(ALICE, CfdTypes.Side.LONG, ALICE_LONG, 50_000e6, false, confidence);
        m.aliceOpenVpi = aliceOpen.economics.vpiAssets;
        assertGt(aliceOpen.economics.vpiAssets, 0, "Alice's public LONG creates the intended passive skew");

        uint256 cooldownEnd = juniorVault.lastDepositTime(CHARLIE) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        (uint256 expectedRedeemEpoch,) = juniorVault.getRequestEpochWindow();
        m.redeemActivation = juniorVault.depositEpochStart(expectedRedeemEpoch);

        uint256 redeemRequestId;
        if (sequence != Sequence.RemainLpThroughClose) {
            m.juniorNavBeforeRequest = juniorVault.totalAssets();
            m.juniorSupplyBeforeRequest = juniorVault.totalSupply();
            uint256 vaultEscrowBefore = juniorVault.balanceOf(address(juniorVault));
            uint256 beforeRequestGas = gasleft();
            vm.prank(CHARLIE);
            redeemRequestId = juniorVault.requestRedeem(charlieShares, CHARLIE, CHARLIE);
            m.redeemEpoch = redeemRequestId;
            m.charlieSharesRequested = charlieShares;
            m.charlieSharesEscrowed = juniorVault.balanceOf(address(juniorVault)) - vaultEscrowBefore;
            m.juniorNavAfterRequest = juniorVault.totalAssets();
            m.juniorSupplyAfterRequest = juniorVault.totalSupply();
            assertEq(m.charlieSharesEscrowed, charlieShares, "requested shares enter vault escrow");
            assertEq(juniorVault.balanceOf(CHARLIE), 0, "Charlie no longer holds requested shares directly");
            assertEq(m.juniorSupplyAfterRequest, m.juniorSupplyBeforeRequest, "escrow does not burn shares");
            assertEq(m.juniorNavAfterRequest, m.juniorNavBeforeRequest, "request alone does not change Junior NAV");
            assertEq(redeemRequestId, expectedRedeemEpoch, "request uses the predicted permissionless LP epoch");
            m.redeemRequestGas += beforeRequestGas - gasleft();
        }

        m.poolBeforeCharlieOpen = pool.totalAssets();
        uint256 charlieBalanceBeforeOpen = clearinghouse.balanceAssets(CHARLIE);
        OrderV3Types.OrderReceipt memory charlieOpen =
            _executeOrder(CHARLIE, CfdTypes.Side.LONG, CHARLIE_LONG, 50_000e6, false, confidence);
        m.poolAfterCharlieOpen = pool.totalAssets();
        m.charlieOpenVpi = charlieOpen.economics.vpiAssets;
        m.charlieOpenFee = charlieOpen.economics.executionFeeAssets;
        m.charlieOpenCarry = charlieOpen.economics.carryAssets;
        m.charlieOpenBounty = charlieOpen.bounty.bountyPaidAssets;
        m.charlieOpenActionChargeCollected = charlieOpen.economics.actionChargeCollectedAssets;
        m.charlieOpenGas = _lastExecutionGas;
        m.charlieOpenCommitGas = _lastCommitGas;
        m.juniorNavAfterCharlieOpen = juniorVault.totalAssets();
        m.freeAssetsAfterCharlieOpen = pool.getFreeAssets();
        m.maxJuniorWithdrawAfterCharlieOpen = pool.getMaxJuniorWithdraw();
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory poolSnapshot =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        m.maxLiabilityAfterCharlieOpen = poolSnapshot.maxLiabilityAssets;
        m.settlementBufferAfterCharlieOpen = poolSnapshot.supplementalReservedAssets;
        m.terminalNavPnlAfterCharlieOpen = poolSnapshot.terminalLpPriceDeltaAssets;
        assertGt(m.charlieOpenVpi, 0, "Charlie pays positive VPI on the LONG open");
        assertEq(
            m.poolAfterCharlieOpen - m.poolBeforeCharlieOpen,
            uint256(m.charlieOpenVpi),
            "the collected positive VPI is reflected in pool cash"
        );
        if (sequence != Sequence.RemainLpThroughClose) {
            assertGt(
                m.juniorNavAfterCharlieOpen,
                m.juniorNavAfterRequest,
                "Junior NAV includes positive VPI before redemption funding"
            );
            int256 pendingPositiveVpi = m.aliceOpenVpi + m.charlieOpenVpi;
            m.nonVpiNavAdjustmentAfterCharlieOpen =
                int256(m.juniorNavAfterCharlieOpen - m.juniorNavAfterRequest) - pendingPositiveVpi;
            assertApproxEqAbs(
                m.nonVpiNavAdjustmentAfterCharlieOpen,
                m.terminalNavPnlAfterCharlieOpen,
                5,
                "NAV residual matches the Engine terminal price adjustment"
            );
        }
        m.afterOpenVpi = _captureSnapshot(m, redeemRequestId, confidence, juniorVault.totalSupply());
        assertGe(
            charlieBalanceBeforeOpen - clearinghouse.balanceAssets(CHARLIE),
            uint256(m.charlieOpenVpi),
            "successful open collects the positive VPI charge"
        );

        if (sequence == Sequence.ExitAfterOpenBeforeClose) {
            _settleAndClaimRedeem(redeemRequestId, confidence, m, true);
            _advanceToCloseWindow(m.redeemActivation);
        } else if (sequence == Sequence.FundBeforeCloseClaimAfterClose) {
            _settleRedeemFunding(redeemRequestId, confidence, m, true);
            _advanceToCloseWindow(m.redeemActivation);
        } else {
            _advanceToCloseWindow(m.redeemActivation);
        }

        uint256 supplyBeforeClose = juniorVault.totalSupply();
        m.beforeCloseVpi = _captureSnapshot(m, redeemRequestId, confidence, supplyBeforeClose);
        m.poolBeforeCharlieClose = pool.totalAssets();
        m.closeOraclePriceRaw = 100_000_000;
        m.closeOracleExponent = -8;
        m.closeOracleConfidence = confidence;
        m.closeOraclePublishTime = block.timestamp + 1;
        uint256 balanceBeforeClose = clearinghouse.balanceAssets(CHARLIE);
        OrderV3Types.OrderReceipt memory charlieClose =
            _executeOrder(CHARLIE, CfdTypes.Side.LONG, CHARLIE_LONG, 0, true, confidence);
        m.poolAfterCharlieClose = pool.totalAssets();
        m.charlieCloseVpi = charlieClose.economics.vpiAssets;
        m.charlieCloseFee = charlieClose.economics.executionFeeAssets;
        m.charlieCloseCarry = charlieClose.economics.carryAssets;
        m.charlieClosePnl = charlieClose.economics.realizedPnlAssets;
        m.charlieCloseBounty = charlieClose.bounty.bountyPaidAssets;
        m.charlieCloseActionChargeCollected = charlieClose.economics.actionChargeCollectedAssets;
        m.charlieCloseBalanceDelta = int256(clearinghouse.balanceAssets(CHARLIE)) - int256(balanceBeforeClose);
        m.charlieCloseGas = _lastExecutionGas;
        m.charlieCloseCommitGas = _lastCommitGas;
        int256 closeRebate = m.charlieCloseBalanceDelta - m.charlieClosePnl
            + int256(m.charlieCloseFee + uint256(m.charlieCloseCarry) + m.charlieCloseBounty);
        m.charlieCloseRebatePaid = closeRebate > 0 ? uint256(closeRebate) : 0;
        if (sequence == Sequence.ExitAfterOpenBeforeClose) {
            assertEq(m.charlieCloseVpi, -m.charlieOpenVpi, "close rebate reaches the lifetime VPI clamp");
            assertEq(
                m.charlieCloseRebatePaid,
                uint256(-m.charlieCloseVpi),
                "full negative VPI is observed as paid after fees, carry and PnL reconciliation"
            );
        }
        m.closeTimestamp = uint64(block.timestamp);
        m.afterCloseVpi = _captureSnapshot(m, redeemRequestId, confidence, supplyBeforeClose);

        if (sequence == Sequence.ExitAfterOpenBeforeClose) {
            // Retry the same matured epoch after the liability from Charlie's LONG is removed.
            _settleAndClaimRedeem(redeemRequestId, confidence, m, false);
        } else if (sequence == Sequence.FundBeforeCloseClaimAfterClose) {
            uint256 dueBeforeClose = m.redemption1ClaimableAtFunding;
            assertEq(
                juniorVault.claimableRedeemAssets(redeemRequestId, CHARLIE),
                dueBeforeClose,
                "funded redemption claim stays fixed across Charlie's close"
            );
            _claimFundedRedeem(redeemRequestId, m.redemption1FundedShares, confidence, m, true);
            _settleAndClaimRedeem(redeemRequestId, confidence, m, false);
        } else if (sequence == Sequence.CloseBeforeExitFunding) {
            _settleAndClaimRedeem(redeemRequestId, confidence, m, true);
        }

        m.charliePositionSizeFinal = _positionSize(CHARLIE);
        m.redemptionResidualShares = juniorVault.balanceOf(CHARLIE);
        m.redemptionResidualClaimAssets = 0;
        if (redeemRequestId != 0) {
            m.redemptionResidualShares += juniorVault.pendingRedeemRequest(redeemRequestId, CHARLIE);
            m.redemptionResidualShares += juniorVault.refundableRedeemRequest(redeemRequestId, CHARLIE);
            m.redemptionResidualClaimAssets = juniorVault.claimableRedeemAssets(redeemRequestId, CHARLIE);
        }
        m.charlieRemainingShareValue = juniorVault.convertToAssets(m.redemptionResidualShares);
        m.charlieEndWealth = _charlieWealth(redeemRequestId, m.redemptionResidualShares);
        m.bobEndWealth = settlementToken.balanceOf(BOB) + clearinghouse.balanceAssets(BOB)
            + juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        m.oldLpAssetsFinal = juniorVault.convertToAssets(m.oldLpShares);
        m.juniorAssetsFinal = juniorVault.totalAssets();
        if (sequence != Sequence.RemainLpThroughClose) {
            m.charlieWalletBalanceDeltaObserved =
                settlementToken.balanceOf(CHARLIE) - m.charlieWalletBalanceBeforeRedemption;
            assertEq(
                m.redemption1Paid + m.redemption2Paid,
                m.charlieWalletBalanceDeltaObserved,
                "all recorded claims reconcile to Charlie's observed USDG wallet delta"
            );
        }
        assertEq(_positionSize(ALICE), ALICE_LONG, "Alice remains passive with the original LONG");
    }

    function _createJuniorCohort() internal {
        settlementToken.mint(CHARLIE, CHARLIE_LP_CAPITAL);
        settlementToken.mint(BOB, BOB_LP_CAPITAL);

        vm.startPrank(CHARLIE);
        settlementToken.approve(address(juniorVault), CHARLIE_LP_CAPITAL);
        uint256 charlieDepositId = juniorVault.requestDeposit(CHARLIE_LP_CAPITAL, CHARLIE, CHARLIE);
        vm.stopPrank();

        vm.startPrank(BOB);
        settlementToken.approve(address(juniorVault), BOB_LP_CAPITAL);
        uint256 bobDepositId = juniorVault.requestDeposit(BOB_LP_CAPITAL, BOB, BOB);
        vm.stopPrank();

        uint256 activation = juniorVault.depositEpochStart(charlieDepositId);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        uint256 beforeSettleGas = gasleft();
        router.settleLpEpoch(_nextPythUpdate(0));
        uint256 settleGas = beforeSettleGas - gasleft();

        uint256 charlieAssets = juniorVault.claimableDepositRequest(charlieDepositId, CHARLIE);
        uint256 bobAssets = juniorVault.claimableDepositRequest(bobDepositId, BOB);
        assertEq(charlieAssets, CHARLIE_LP_CAPITAL, "Charlie deposit is fully activated");
        assertEq(bobAssets, BOB_LP_CAPITAL, "Bob deposit is fully activated");
        vm.prank(CHARLIE);
        juniorVault.claimDeposit(charlieDepositId, charlieAssets, CHARLIE, CHARLIE);
        vm.prank(BOB);
        juniorVault.claimDeposit(bobDepositId, bobAssets, BOB, BOB);
        // The setup settlement is identical in every snapshot; scenario gas records only the exploit sequence.
        settleGas;
    }

    function _fundTraderHyp(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _settleAndClaimRedeem(
        uint256 requestId,
        uint64 confidence,
        Metrics memory m,
        bool first
    ) internal {
        uint256 fundedShares = _settleRedeemFunding(requestId, confidence, m, first);
        _claimFundedRedeem(requestId, fundedShares, confidence, m, first);
    }

    function _settleRedeemFunding(
        uint256 requestId,
        uint64 confidence,
        Metrics memory m,
        bool first
    ) internal returns (uint256 fundedShares) {
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }

        uint256 supplyBeforeFunding = juniorVault.totalSupply();
        uint256 beforeSettleGas = gasleft();
        router.settleLpEpoch(_nextPythUpdate(confidence));
        m.lpSettlementGas += beforeSettleGas - gasleft();

        fundedShares = juniorVault.claimableRedeemRequest(requestId, CHARLIE);
        uint256 claimableAssets = juniorVault.claimableRedeemAssets(requestId, CHARLIE);
        if (first) {
            m.redemption1FundedShares += fundedShares;
            m.redemption1ClaimableAtFunding += claimableAssets;
            m.afterFirstFunding = _captureSnapshot(m, requestId, confidence, supplyBeforeFunding);
        } else {
            m.redemption2FundedShares += fundedShares;
            m.redemption2ClaimableAtFunding += claimableAssets;
            m.afterSecondFunding = _captureSnapshot(m, requestId, confidence, supplyBeforeFunding);
        }
    }

    function _claimFundedRedeem(
        uint256 requestId,
        uint256 fundedShares,
        uint64 confidence,
        Metrics memory m,
        bool first
    ) internal {
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, CHARLIE);
        uint256 claimableAssets = juniorVault.claimableRedeemAssets(requestId, CHARLIE);
        assertEq(fundedShares, claimableShares, "claim uses exactly the shares funded at this settlement");

        uint256 paid;
        uint256 supplyBeforeClaim = juniorVault.totalSupply();
        if (fundedShares > 0) {
            uint256 balanceBefore = settlementToken.balanceOf(CHARLIE);
            vm.prank(CHARLIE);
            uint256 beforeClaimGas = gasleft();
            uint256 returnedAssets = juniorVault.claimRedeem(requestId, fundedShares, CHARLIE, CHARLIE);
            m.redeemClaimGas += beforeClaimGas - gasleft();
            paid = settlementToken.balanceOf(CHARLIE) - balanceBefore;
            assertEq(returnedAssets, paid, "claim return equals Charlie's observed ERC20 transfer");
            assertEq(paid, claimableAssets, "claim pays exactly the funded redemption assets");
            assertEq(
                juniorVault.claimableRedeemAssets(requestId, CHARLIE),
                0,
                "claimed USDG is no longer counted as an outstanding claim"
            );
        } else {
            assertEq(claimableAssets, 0, "no funded shares means no claimable USDG");
        }

        if (first) {
            m.redemption1Paid += paid;
            m.afterFirstClaim = _captureSnapshot(m, requestId, confidence, supplyBeforeClaim);
        } else {
            m.redemption2Paid += paid;
            m.afterSecondClaim = _captureSnapshot(m, requestId, confidence, supplyBeforeClaim);
        }
    }

    function _captureSnapshot(
        Metrics memory m,
        uint256 requestId,
        uint64 confidence,
        uint256 supplyBeforeEvent
    ) internal view returns (EconomicSnapshot memory s) {
        s.poolAssets = pool.totalAssets();
        s.juniorNav = juniorVault.totalAssets();
        s.juniorSupply = juniorVault.totalSupply();
        s.juniorAssetsPerShare = juniorVault.convertToAssets(10 ** uint256(juniorVault.decimals()));
        assertGe(supplyBeforeEvent, s.juniorSupply, "snapshot event cannot increase share supply");
        s.sharesBurnedAtEvent = supplyBeforeEvent - s.juniorSupply;
        if (m.juniorSupplyBeforeRequest >= s.juniorSupply) {
            s.sharesBurnedCumulative = m.juniorSupplyBeforeRequest - s.juniorSupply;
        }
        s.charlieUsdGReceived = settlementToken.balanceOf(CHARLIE) - m.charlieWalletBalanceBeforeRedemption;
        if (requestId != 0) {
            s.claimableUsdG = juniorVault.claimableRedeemAssets(requestId, CHARLIE);
            s.pendingRedeemShares = juniorVault.pendingRedeemRequest(requestId, CHARLIE)
                + juniorVault.refundableRedeemRequest(requestId, CHARLIE);
            s.claimableRedeemShares = juniorVault.claimableRedeemRequest(requestId, CHARLIE);
        }
        s.bobShareValue = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        s.freeAssets = pool.getFreeAssets();
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory poolSnapshot =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        s.maxLiability = poolSnapshot.maxLiabilityAssets;
        s.timestamp = uint64(block.timestamp);
        s.oraclePublishTime = uint64(block.timestamp);
        s.oracleConfidence = confidence;
        s.oraclePriceRaw = 100_000_000;
        s.oracleExponent = -8;
    }

    function _advanceToCloseWindow(
        uint256 activation
    ) internal {
        uint256 target = activation + 1;
        if (block.timestamp < target) {
            vm.warp(target);
        }
    }

    uint256 private _lastExecutionGas;
    uint256 private _lastCommitGas;

    function _executeOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint64 confidence
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("hyp25", account, router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = isClose;
        request.targetPrice = ((side == CfdTypes.Side.LONG) == isClose) ? 2e8 : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });

        uint256 commitGasBefore = gasleft();
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        _lastCommitGas = commitGasBefore - gasleft();
        bytes[] memory updateData = _nextPythUpdate(confidence);
        vm.recordLogs();
        vm.prank(KEEPER);
        uint256 gasBefore = gasleft();
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, updateData);
        _lastExecutionGas = gasBefore - gasleft();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "permissionless keeper executes");

        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == id
            ) {
                (bytes32 receiptHash,, uint64 terminalTime, OrderV3Types.OrderReceipt memory found) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                assertTrue(router.lifecycleBook().verifyReceipt(found, terminalTime), "authenticated execution receipt");
                assertEq(router.lifecycleBook().terminalOutcome(id).receiptHash, receiptHash);
                return found;
            }
        }
        revert("missing terminal Router receipt");
    }

    function _nextPythUpdate(
        uint64 confidence
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        uint64 publishTime = uint64(block.timestamp + 1);
        vm.warp(publishTime);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(100_000_000), confidence, int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = "";
    }

    function _charlieWealth(
        uint256 redeemRequestId,
        uint256 residualShares
    ) internal view returns (uint256) {
        uint256 wealth = settlementToken.balanceOf(CHARLIE) + clearinghouse.balanceAssets(CHARLIE)
            + juniorVault.convertToAssets(residualShares);
        if (redeemRequestId != 0) {
            wealth += juniorVault.claimableRedeemAssets(redeemRequestId, CHARLIE);
        }
        return wealth;
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _abs(
        int256 value
    ) internal pure returns (uint256) {
        return value < 0 ? uint256(-value) : uint256(value);
    }

    function _logMetrics(
        string memory label,
        Metrics memory m,
        uint64 confidence
    ) internal {
        emit log_string(label);
        emit log_named_uint("hyp25.confidence_raw", confidence);
        emit log_named_uint("hyp25.redeem_activation", m.redeemActivation);
        emit log_named_uint("hyp25.junior_nav_before_request", m.juniorNavBeforeRequest);
        emit log_named_uint("hyp25.junior_nav_after_request", m.juniorNavAfterRequest);
        emit log_named_uint("hyp25.junior_supply_before_request", m.juniorSupplyBeforeRequest);
        emit log_named_uint("hyp25.junior_supply_after_request", m.juniorSupplyAfterRequest);
        emit log_named_uint("hyp25.charlie_shares_requested", m.charlieSharesRequested);
        emit log_named_uint("hyp25.charlie_shares_escrowed", m.charlieSharesEscrowed);
        emit log_named_int("hyp25.charlie_open_vpi", m.charlieOpenVpi);
        emit log_named_int("hyp25.alice_open_vpi", m.aliceOpenVpi);
        emit log_named_uint("hyp25.charlie_open_fee", m.charlieOpenFee);
        emit log_named_int("hyp25.charlie_open_carry", m.charlieOpenCarry);
        emit log_named_uint("hyp25.charlie_open_bounty", m.charlieOpenBounty);
        emit log_named_uint("hyp25.charlie_open_action_charge_collected", m.charlieOpenActionChargeCollected);
        emit log_named_uint("hyp25.pool_before_charlie_open", m.poolBeforeCharlieOpen);
        emit log_named_uint("hyp25.pool_after_charlie_open", m.poolAfterCharlieOpen);
        emit log_named_uint("hyp25.junior_nav_after_charlie_open", m.juniorNavAfterCharlieOpen);
        emit log_named_uint("hyp25.free_assets_after_charlie_open", m.freeAssetsAfterCharlieOpen);
        emit log_named_uint("hyp25.max_junior_withdraw_after_charlie_open", m.maxJuniorWithdrawAfterCharlieOpen);
        emit log_named_uint("hyp25.max_liability_after_charlie_open", m.maxLiabilityAfterCharlieOpen);
        emit log_named_uint("hyp25.settlement_buffer_after_charlie_open", m.settlementBufferAfterCharlieOpen);
        emit log_named_int("hyp25.terminal_nav_pnl_after_charlie_open", m.terminalNavPnlAfterCharlieOpen);
        emit log_named_int("hyp25.non_vpi_nav_adjustment_after_charlie_open", m.nonVpiNavAdjustmentAfterCharlieOpen);
        emit log_named_uint("hyp25.redemption1_funded_shares", m.redemption1FundedShares);
        emit log_named_uint("hyp25.redemption1_paid", m.redemption1Paid);
        emit log_named_int("hyp25.charlie_close_vpi", m.charlieCloseVpi);
        emit log_named_uint("hyp25.charlie_close_fee", m.charlieCloseFee);
        emit log_named_int("hyp25.charlie_close_carry", m.charlieCloseCarry);
        emit log_named_int("hyp25.charlie_close_pnl", m.charlieClosePnl);
        emit log_named_uint("hyp25.charlie_close_bounty", m.charlieCloseBounty);
        emit log_named_uint("hyp25.charlie_close_action_charge_collected", m.charlieCloseActionChargeCollected);
        emit log_named_int("hyp25.charlie_close_balance_delta", m.charlieCloseBalanceDelta);
        emit log_named_uint("hyp25.charlie_close_rebate_paid", m.charlieCloseRebatePaid);
        emit log_named_uint("hyp25.pool_before_charlie_close", m.poolBeforeCharlieClose);
        emit log_named_uint("hyp25.pool_after_charlie_close", m.poolAfterCharlieClose);
        emit log_named_uint("hyp25.redemption2_funded_shares", m.redemption2FundedShares);
        emit log_named_uint("hyp25.redemption2_paid", m.redemption2Paid);
        emit log_named_uint("hyp25.redemption_residual_shares", m.redemptionResidualShares);
        emit log_named_uint("hyp25.redemption_residual_claim_assets", m.redemptionResidualClaimAssets);
        emit log_named_uint("hyp25.charlie_remaining_share_value", m.charlieRemainingShareValue);
        emit log_named_uint("hyp25.charlie_start_wealth", m.charlieStartWealth);
        emit log_named_uint("hyp25.charlie_end_wealth", m.charlieEndWealth);
        emit log_named_int("hyp25.charlie_net_wealth_delta", int256(m.charlieEndWealth) - int256(m.charlieStartWealth));
        emit log_named_uint("hyp25.bob_end_wealth", m.bobEndWealth);
        emit log_named_uint("hyp25.old_lp_assets_final", m.oldLpAssetsFinal);
        emit log_named_uint("hyp25.junior_assets_final", m.juniorAssetsFinal);
        emit log_named_uint("hyp25.charlie_open_execution_gas", m.charlieOpenGas);
        emit log_named_uint("hyp25.charlie_close_execution_gas", m.charlieCloseGas);
        emit log_named_uint("hyp25.charlie_open_commit_gas", m.charlieOpenCommitGas);
        emit log_named_uint("hyp25.charlie_close_commit_gas", m.charlieCloseCommitGas);
        emit log_named_uint("hyp25.redemption_request_evm_gas", m.redeemRequestGas);
        emit log_named_uint("hyp25.lp_settlement_evm_gas", m.lpSettlementGas);
        emit log_named_uint("hyp25.redeem_claim_evm_gas", m.redeemClaimGas);
        emit log_named_uint("hyp25.setup_evm_gas", m.setupGas);
        emit log_named_uint("hyp25.close_timestamp", m.closeTimestamp);
        emit log_named_uint("hyp25.close_oracle_price_raw", m.closeOraclePriceRaw);
        emit log_named_int("hyp25.close_oracle_exponent", m.closeOracleExponent);
        emit log_named_uint("hyp25.close_oracle_confidence", m.closeOracleConfidence);
        emit log_named_uint("hyp25.close_oracle_publish_time", m.closeOraclePublishTime);
        _logEconomicSnapshot("after_open_vpi", m.afterOpenVpi);
        _logEconomicSnapshot("after_first_funding_before_claim", m.afterFirstFunding);
        _logEconomicSnapshot("after_first_claim", m.afterFirstClaim);
        _logEconomicSnapshot("before_close_vpi", m.beforeCloseVpi);
        _logEconomicSnapshot("after_close_vpi", m.afterCloseVpi);
        _logEconomicSnapshot("after_second_funding", m.afterSecondFunding);
        _logEconomicSnapshot("after_second_claim", m.afterSecondClaim);
    }

    function _logEconomicSnapshot(
        string memory label,
        EconomicSnapshot memory s
    ) internal {
        emit log_string(string.concat("hyp25.snapshot.", label));
        emit log_named_uint("pool_assets", s.poolAssets);
        emit log_named_uint("junior_nav", s.juniorNav);
        emit log_named_uint("junior_total_supply", s.juniorSupply);
        emit log_named_uint("junior_assets_per_share", s.juniorAssetsPerShare);
        emit log_named_uint("shares_burned_at_event", s.sharesBurnedAtEvent);
        emit log_named_uint("shares_burned_cumulative", s.sharesBurnedCumulative);
        emit log_named_uint("charlie_usdg_received_cumulative", s.charlieUsdGReceived);
        emit log_named_uint("claimable_usdg_due", s.claimableUsdG);
        emit log_named_uint("pending_or_refundable_redeem_shares", s.pendingRedeemShares);
        emit log_named_uint("claimable_redeem_shares", s.claimableRedeemShares);
        emit log_named_uint("bob_share_value", s.bobShareValue);
        emit log_named_uint("free_assets", s.freeAssets);
        emit log_named_uint("max_liability", s.maxLiability);
        emit log_named_uint("timestamp", s.timestamp);
        emit log_named_uint("oracle_publish_time", s.oraclePublishTime);
        emit log_named_uint("oracle_confidence", s.oracleConfidence);
        emit log_named_int("oracle_price_raw", s.oraclePriceRaw);
        emit log_named_int("oracle_exponent", s.oracleExponent);
    }

}
```
