# \#97892 [SC-Low] TrancheVault.requestRedeem accepts any controller, so a stranger can plant a refund-pending head on an LP's redeem queue

**Submitted on Nov 3rd 2026 at 19:13:13 UTC by @humbleservant for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97892
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`TrancheVault.requestRedeem(shares, controller, owner)` authorizes the share owner and never asks the controller, so any holder of a single share atom can append a request to another LP's redeem queue. When that one-atom request is alone in its hourly epoch it quotes zero assets, the pool marks the epoch refundable, and the request becomes the LP's redeem head. From then on the LP's ERC-7540 / ERC-4626 claim path (`maxRedeem`, `maxWithdraw`, `redeem`, `withdraw`) reads zero and reverts for the LP's own funded requests. The only way to clear the head is `claimRedeemRefund`, which hands the LP the stranger's atom and restarts the LP's one-hour cooldown on share transfers and new redeem requests. The stranger pays one share atom (about 1e-9 USDG) and two transactions per epoch and can pick any LP, including an integration contract that only speaks ERC-7540. I found this while checking whether the receiver rule added after the Cecuro review also holds on the request side.

## Vulnerability Details

### 1. The request side never asks the controller

```solidity
// TrancheVault.sol L606-632
function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId) {
    if (controller == address(0) || owner == address(0)) {
        revert TrancheVault__ZeroAddress();
    }
    if (shares == 0) {
        revert TrancheVault__WithdrawalTooSmall();
    }
    if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
        revert TrancheVault__DepositCooldown();
    }
    uint256 maxShares = maxRequestRedeem(owner);
    if (shares > maxShares) {
        revert TrancheVault__ExceededMaxRequestRedeem(owner, shares, maxShares);
    }
    _requireMinimumRedeemSize(shares, maxShares);
    if (msg.sender != owner && !isOperator[owner][msg.sender]) {
        _spendAllowance(owner, msg.sender, shares);
    }
    _transfer(owner, address(this), shares);

    requestId = _enqueueRedeemRequest(shares, controller);
    emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
}
```

Every check is about `owner`: the cooldown (L617), the request capacity (L620), the minimum size (L624) and the allowance (L625-627). `controller` is only compared with `address(0)` (L611). `_enqueueRedeemRequest` then appends the request to the controller's own queue (`_appendControllerRedeem`, L1783-1797). If the controller has no other outstanding redeem request, the planted request becomes `controllerRedeemHead[controller]` at once; otherwise it becomes the head as soon as the requests in front of it are claimed.

### 2. Meeting the owner checks costs one share atom

The stranger needs a wallet that holds shares and is out of cooldown. One ordinary deposit into a source wallet, followed by one hour, provides that permanently. From then on the source sends one share atom to a fresh wallet:

- `_update` (L325-331) gives the receiver the sender's `lastDepositTime` when it is newer than the receiver's. The source's stamp has already elapsed, so the fresh wallet is out of cooldown immediately, and sending does not touch the source's own stamp.
- One atom is the fresh wallet's whole balance, so the request is a full-balance request, which `_requireMinimumRedeemSize` (L1611-1618) lets through even though the atom has no asset value.

`maxRequestRedeem(freshWallet)` returns 1, and `requestRedeem(1, lp, freshWallet)` succeeds with the LP as controller.

### 3. Settlement turns the atom into a refund-pending head

At the epoch boundary, `HousePool._fundRedemptionPhase` prices the matured head (HousePool.sol L957-959):

```solidity
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);
if (fullHeadAssets == 0) {
    if (phase.fundedShares != 0) {
        break;
    }
    ...
    if (vault.refundRedeemEpochRemainder(epochId, remainingShares) != remainingShares) {
```

The vault prices shares with a 1,000 virtual-share offset (`VIRTUAL_SHARES`, TrancheVault.sol L36), so one share atom is worth about 0.001 asset atoms at launch pricing (999,987 asset atoms per 1e9 share atoms in my test) and its quote floors to zero. With nothing else funded in the phase, the pool calls `refundRedeemEpochRemainder` (TrancheVault.sol L1505-1524), which sets `refundEnabled` on the epoch. The quote only becomes nonzero once a single share atom is worth a whole asset atom, roughly a 1000x rise in the share price. I checked two realistic price moves on the real contracts: after the owner's `accountExcess` path admitted 6,000 USDG and the share price doubled (1,999,645 asset atoms per 1e9 share atoms), and after an engine payout of 4,000 USDG cut it to 333,546, the one-atom epoch is still refunded.

`settleLpEpoch` is permissionless while no positions are open and runs through the router with a fresh mark otherwise (HousePool.sol L797-821), so settlement happens on the normal keeper cadence without any action by the LP.

### 4. The standard claim path reads the planted head

`maxRedeem` and `maxWithdraw` read `controllerRedeemHead[controller]` (TrancheVault.sol L1299-1311), and `redeem` and `withdraw` claim that same head (L998-1012). A refund-pending head has zero claimable shares and assets, so both views return 0 and both claims revert with `ERC4626ExceededMaxRedeem` / `ERC4626ExceededMaxWithdraw` (L1161, L1182), even when the LP's own request behind the head is fully funded.

### 5. Every way out restarts the LP's cooldown

- After settlement, `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable` (L894-896).
- `claimRedeemRefund` (L916-947) requires `receiver == controller` (L931, `_requireShareReceiver` at L1620-1628). For an atom-only epoch the refundable amount is the whole atom, so `shares != 0` and the vault sets `lastDepositTime[lp] = block.timestamp` before it transfers the atom (L942-945).
- An approved operator can acknowledge the refund for the LP, but the receiver must still be the LP, so the stamp still lands on the LP.
- Before maturity the LP can cancel the planted request, but cancellation also delivers the atom to the LP and stamps it (L903, L911).

The stamp blocks share transfers (L326) and new redeem requests (L617, and `maxRequestRedeem` at L1331) for `DEPOSIT_COOLDOWN = 1 hours` (L38).

### 6. Why the existing receiver rule does not cover this

The Cecuro review reported the receiver-side form of this problem (LOW-39 and LOW-40): a third party could deliver shares to an account and impose its cooldown. The fix requires `receiver == controller` for every share delivery, and the interface states the goal in `IAsyncTrancheVault.sol` L12-14:

> "Deposit claims and redemption cancellations/refunds require receiver == controller, including zero-share acknowledgements, so share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance."

The receiver check itself carries the comment "require the controller's authority over cooldowns" (TrancheVault.sol L1624).

The request side never received the same treatment. A third party still chooses the controller, and the vault's own refund rules then force the share delivery, and with it the cooldown, onto that controller without its authority. ERC-7540 does let an owner name any controller, but a plain ERC-7540 vault has no refundable head that blocks the standard claim path. The head rule in `IAsyncTrancheVault.sol` L8-10 ("A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request") combined with the mandatory self-delivery is what turns the standard permission into a handle on another account's cooldown. This is the LOW-39/LOW-40 fix left incomplete on the request side.

### 7. What this report does not claim

- Not CECURO-LOW-67 (a full-balance dust exit can burn the requester's own shares for zero assets). Here the dust exit is only the vehicle, and the harmed account is a different controller.
- Not CECURO-MED-24 (zero-payout refund epochs consume bounded settlement work). I do not count the one-pass delay a refund epoch can add to later epochs.
- Not CECURO-MED-29 (per-controller flooring on shared epochs). On a shared epoch the planted atom is funded at zero assets and the LP clears it with a zero-asset `redeem(1)` and no cooldown; my fifth test shows that limit.

### 8. Suggested fix

- Require the controller's consent in `requestRedeem`, for example `controller == owner || isOperator[controller][msg.sender]`. `requestDeposit` (L569-603) accepts any controller in the same way and should get the same check, even though planting a deposit head costs the requester real assets that the controller keeps.
- Alternatively, let a controller dismiss a foreign refund head with the shares returned to the requesting owner and no cooldown stamp, or have the standard views and claims skip refund-pending heads.

## Impact Details

Impact: Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".

- **Target:** any LP or integration contract with redeem requests in either tranche, chosen by the attacker.
- **While the head is pending:** the target's ERC-7540 / ERC-4626 claim path reads zero and reverts for every funded request behind the head. An integration that only implements the standard surface has no way to claim its funded USDG through the interface it supports.
- **Clearing the head:** the target receives one share atom and a 3,600-second lock on share transfers and new redeem requests for its whole wallet. Every exit path imposes the lock.
- **Repetition:** one planted head per hourly request epoch from one re-funded dust wallet. Planted heads stack, and each needs its own acknowledgement, each restarting the cooldown (my fourth test plants two).
- **Attacker cost:** one share atom, about 1e-9 USDG at launch pricing, plus two transactions (a transfer and a request) per epoch, after a one-off deposit and one hour of cooldown for the source wallet. The attacker gains nothing, which matches the program's griefing definition.

Limits, stated plainly:

- No USDG is lost or frozen. `claimRedeem(requestId, shares, receiver, controller)` pays funded requests by explicit id at any time, and a pending planted head does not block new redeem requests.
- The plant only becomes a refund-pending head when no other request for that tranche lands in the same epoch. On a shared epoch the atom is funded at zero assets and a zero-asset `redeem(1)` clears it with no cooldown. The attacker cannot force an empty epoch, but on a quiet vault, and at launch, most hourly epochs are empty.
- A larger plant is a gift rather than a block: after the share price doubled, a 501-atom request already quoted a nonzero amount, was funded, and the target claimed it through the standard path with no cooldown change.

## References

- Frozen competition commit (the code is not deployed; every line below is at this commit): https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- `requestRedeem`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632
- Cooldown on transfer and stamp inheritance: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L325-L331
- Minimum-size exemption for full-balance requests: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618
- Controller queue append: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1797
- Zero-quote refund branch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L950-L987 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1505-L1524
- Settlement authorization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L821
- Head-reading views and claims: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1148-L1192
- Cancel and refund acknowledgement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L884-L947
- Receiver rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1620-L1628
- Cooldown constant and request capacity: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1328-L1335
- Interface statement of the receiver rule and the head rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L8-L14
- `packages/perps/SECURITY.md` L1399: "Redemption cancellation or terminal refund still begins a new cooldown when shares return to a wallet."
- Cecuro review items checked: LOW-39 and LOW-40 (receiver rule, fixed on the claim side only), LOW-67 (dust full-balance exits), MED-24 (zero-payout refund epochs), MED-29 (shared-epoch flooring).

## Proof of Concept

## Proof of Concept

Save the file below as `packages/perps/test/poc/ForeignControllerRefundHead.t.sol` in a checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It uses the repo's own `HousePoolTestBase` fixture, so the HousePool, both TrancheVaults, CfdEngine, OrderRouter and PletherOracle are the real contracts, with MockUSDG and MockPyth as in the repo's tests. The only shortcuts are the fixture's own helpers: `_refreshMarkForAsyncSettlement` installs a mark through a router prank (a keeper's Pyth update in production), and `_settleLpEpochForTest` calls `pool.settleLpEpoch(0, 0)`, which is permissionless while no positions are open. Every actor is an unprivileged address and nothing writes storage directly.

The five tests:

1. `test_control_lpWithoutPlantedHeadRedeemsThroughStandardSurface`: without a planted head the LP redeems through `redeem` with no cooldown change.
2. `test_strangerPlantsRefundHead_lpStandardSurfaceDeadUntilAcknowledged`: a one-atom wallet plants a request on the LP; settlement refunds it; the LP's fully funded request is invisible to `maxRedeem` / `maxWithdraw` and `redeem` / `withdraw` revert with exact arguments; the explicit-id claim still pays (balance delta); the only clear hands over the atom and restarts the cooldown; at +3,599 s transfers and new requests revert, at +3,600 s both work.
3. `test_exits_everyClearDeliversTheAtomToTheLpAndRestartsItsCooldown`: pre-maturity cancel stamps the LP; post-settlement cancel reverts; a refund to another receiver reverts; an approved operator cannot redirect the refund, and its acknowledgement stamps the LP.
4. `test_twoPlantedEpochsStackAndNeedTwoAcknowledgements`: two planted epochs stack and need two acknowledgements.
5. `test_sharedEpoch_atomFundsAtZeroAssets_headClearsWithoutCooldown`: the limit; on a shared epoch the atom is funded at zero assets and clears with no cooldown.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

/// @notice TrancheVault.requestRedeem authorizes only the share owner and accepts any controller. A holder of one
///         share atom can therefore enqueue a full-balance request onto another LP's controller queue. Alone in its
///         hourly epoch the atom quotes zero assets, settlement marks the epoch refundable, and the LP's controller
///         redeem head becomes a refund-pending request. The ERC-7540 / ERC-4626 standard claim surface (maxRedeem,
///         maxWithdraw, redeem, withdraw) then reads zero and reverts for the LP's own later funded request. The only
///         post-settlement clear is claimRedeemRefund, which must deliver the atom to the LP and restarts the LP's
///         one-hour cooldown on share transfers and new redeem requests.
/// @dev    Real HousePool, both TrancheVaults, CfdEngine, OrderRouter and PletherOracle from the repo fixture
///         (HousePoolTestBase); MockUSDG and MockPyth as in the repo's own tests. No storage writes, no seeded
///         vault or pool state; every actor is unprivileged.
contract ForeignControllerRefundHeadTest is HousePoolTestBase {

    address internal lp = address(0x1111);
    address internal otherLp = address(0x2222);
    address internal delegate = address(0x3333);
    /// @dev The stranger's funded wallet: one 2 USDG deposit, cooled once, then the source of every atom.
    address internal source = address(0xA11);
    /// @dev Fresh wallets that hold one share atom each.
    address internal dust = address(0xD05);
    address internal dustTwo = address(0xD06);

    /// @dev One-off setup for the stranger: a 2 USDG deposit into the junior vault and one hour of cooldown.
    function _prepareSource(
        TrancheVault vault
    ) internal returns (uint256 shares) {
        (, shares) = _depositAsync(vault, source, 2e6);
        assertGt(shares, 1);
        _finishAsyncCooldown(vault, source);
    }

    /// @dev Moves `atoms` share atoms from the cooled source wallet to `wallet`, then requests redeem of all of them
    ///      with `controller` as the request controller. The receiver inherits the elapsed source stamp, so the
    ///      full-balance request passes the cooldown and the minimum-size checks at once.
    function _plant(
        TrancheVault vault,
        address wallet,
        uint256 atoms,
        address controller
    ) internal returns (uint256 requestId) {
        vm.prank(source);
        vault.transfer(wallet, atoms);
        assertEq(vault.maxRequestRedeem(wallet), atoms, "full-balance request from the fresh wallet is admissible");
        vm.prank(wallet);
        requestId = vault.requestRedeem(atoms, controller, wallet);
        assertEq(vault.controllerRedeemTail(controller), requestId, "request is appended to the controller's queue");
    }

    /// @dev Negative control: an LP without a planted head claims its funded request through the standard surface
    ///      with no acknowledgement and no cooldown change.
    function test_control_lpWithoutPlantedHeadRedeemsThroughStandardSurface() public {
        TrancheVault vault = juniorVault;
        (, uint256 shares) = _depositAsync(vault, lp, 5_000e6);
        _finishAsyncCooldown(vault, lp);
        vm.prank(lp);
        uint256 rOwn = vault.requestRedeem(shares / 2, lp, lp);
        _settleAsyncRequest(rOwn, true);
        uint256 funded = vault.claimableRedeemRequest(rOwn, lp);
        assertEq(funded, shares / 2, "the request is fully funded");
        assertEq(vault.maxRedeem(lp), funded);
        uint256 stampBefore = vault.lastDepositTime(lp);
        uint256 balBefore = settlementToken.balanceOf(lp);
        vm.prank(lp);
        uint256 assets = vault.redeem(funded, lp, lp);
        assertGt(assets, 0);
        assertEq(settlementToken.balanceOf(lp) - balBefore, assets);
        assertEq(vault.lastDepositTime(lp), stampBefore, "no cooldown change on a funded standard claim");
        assertEq(vault.controllerRedeemHead(lp), 0);
    }

    function test_strangerPlantsRefundHead_lpStandardSurfaceDeadUntilAcknowledged() public {
        TrancheVault vault = juniorVault;
        (, uint256 lpShares) = _depositAsync(vault, lp, 5_000e6);
        _prepareSource(vault);

        // 1. the stranger's one-atom wallet requests redeem with the LP as controller; nobody else requests this epoch
        uint256 rA = _plant(vault, dust, 1, lp);
        assertEq(vault.controllerRedeemHead(lp), rA, "the LP's controller head is now the stranger's request");
        assertEq(vault.estimateRedeemAssets(1), 0, "one share atom quotes zero assets");
        emit log_named_uint("asset atoms per 1e9 share atoms", vault.estimateRedeemAssets(1e9));

        // 2. settlement refunds the zero-quote epoch; the LP's head is a refund-pending request
        _settleAsyncRequest(rA, true);
        assertTrue(vault.redeemRefundPending(rA, lp), "head is refund-pending for the LP");
        assertEq(vault.claimableRedeemRequest(rA, lp), 0);
        assertEq(vault.refundableRedeemRequest(rA, lp), 1);

        // 3. the LP's own later request is funded in full behind the planted head
        _finishAsyncCooldown(vault, lp);
        vm.prank(lp);
        uint256 rB = vault.requestRedeem(lpShares / 2, lp, lp);
        assertGt(rB, rA, "the LP's request lands behind the planted head");
        _settleAsyncRequest(rB, true);
        uint256 funded = vault.claimableRedeemRequest(rB, lp);
        assertEq(funded, lpShares / 2, "the LP's request is fully funded");
        uint256 fundedAssets = vault.claimableRedeemAssets(rB, lp);
        assertGt(fundedAssets, 0);

        // 4. standard surface: zero capacity and reverts, although funded value is waiting behind the head
        assertEq(vault.maxRedeem(lp), 0, "maxRedeem reads the planted head");
        assertEq(vault.maxWithdraw(lp), 0, "maxWithdraw reads the planted head");
        vm.prank(lp);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, lp, funded, 0));
        vault.redeem(funded, lp, lp);
        vm.prank(lp);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, lp, fundedAssets, 0));
        vault.withdraw(fundedAssets, lp, lp);

        // 5. the explicit-id claim still settles assets to the LP's wallet while the head stays pending
        uint256 half = funded / 2;
        uint256 balBefore = settlementToken.balanceOf(lp);
        vm.prank(lp);
        uint256 paid = vault.claimRedeem(rB, half, lp, lp);
        assertGt(paid, 0);
        assertEq(settlementToken.balanceOf(lp) - balBefore, paid, "explicit-id claim pays with the head pending");
        assertTrue(vault.redeemRefundPending(rA, lp), "planted head still pending after the explicit claim");
        assertEq(vault.maxRedeem(lp), 0);

        // 6. the only clear: acknowledge the stranger's refund, which hands the LP the atom and restarts its cooldown
        assertGt(vault.maxRequestRedeem(lp), 0, "the LP could request before acknowledging");
        uint256 t = block.timestamp;
        vm.prank(lp);
        uint256 refunded = vault.claimRedeemRefund(rA, lp, lp);
        assertEq(refunded, 1, "the LP is handed the stranger's atom");
        assertEq(vault.lastDepositTime(lp), t, "wallet-wide cooldown restarted");
        assertEq(vault.controllerRedeemHead(lp), rB);
        assertEq(vault.maxRedeem(lp), funded - half, "standard surface restored only after the acknowledgement");

        // 7. the lock lasts exactly DEPOSIT_COOLDOWN: transfers and new requests are blocked until one hour has passed
        assertEq(vault.DEPOSIT_COOLDOWN(), 1 hours);
        vm.warp(t + 1 hours - 1);
        assertEq(vault.maxRequestRedeem(lp), 0, "no new redeem request inside the hour");
        vm.prank(lp);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        vault.transfer(otherLp, 1);
        vm.prank(lp);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        vault.requestRedeem(2e9, lp, lp);

        vm.warp(t + 1 hours);
        assertEq(vault.maxRequestRedeem(lp), vault.balanceOf(lp), "request capacity recovers at exactly one hour");
        vm.prank(lp);
        assertTrue(vault.transfer(otherLp, 1));
        vm.prank(lp);
        assertGt(vault.requestRedeem(2e9, lp, lp), 0, "a new redeem request is accepted at exactly one hour");
    }

    function test_exits_everyClearDeliversTheAtomToTheLpAndRestartsItsCooldown() public {
        TrancheVault vault = juniorVault;
        (, uint256 lpShares) = _depositAsync(vault, lp, 5_000e6);
        _prepareSource(vault);
        _finishAsyncCooldown(vault, lp);

        // (a) before maturity the LP can cancel the planted request, only to itself, and that restarts its cooldown
        uint256 rA = _plant(vault, dust, 1, lp);
        assertEq(vault.controllerRedeemHead(lp), rA);
        vm.prank(lp);
        vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
        vault.cancelRedeemRequest(rA, otherLp, lp);
        vm.prank(lp);
        uint256 cancelled = vault.cancelRedeemRequest(rA, lp, lp);
        assertEq(cancelled, 1, "the LP receives the atom on cancel");
        assertEq(vault.lastDepositTime(lp), block.timestamp, "pre-maturity cancel restarts the LP cooldown");
        assertEq(vault.controllerRedeemHead(lp), 0);
        _finishAsyncCooldown(vault, lp);

        // (b) after settlement: cancel is unavailable, the refund receiver must be the controller, and an approved
        //     operator cannot redirect the refund either
        uint256 rB = _plant(vault, dustTwo, 1, lp);
        _settleAsyncRequest(rB, true);
        assertTrue(vault.redeemRefundPending(rB, lp));
        vm.prank(lp);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        vault.cancelRedeemRequest(rB, lp, lp);
        vm.prank(lp);
        vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
        vault.claimRedeemRefund(rB, otherLp, lp);
        vm.prank(lp);
        vault.setOperator(delegate, true);
        vm.prank(delegate);
        vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
        vault.claimRedeemRefund(rB, delegate, lp);

        // (c) with the planted head pending the LP is not blocked from new requests or from explicit-id claims
        _finishAsyncCooldown(vault, lp);
        assertGt(vault.maxRequestRedeem(lp), 0, "a pending planted head does not block new requests");
        vm.prank(lp);
        uint256 rC = vault.requestRedeem(lpShares / 2, lp, lp);
        _settleAsyncRequest(rC, true);
        uint256 funded = vault.claimableRedeemRequest(rC, lp);
        assertGt(funded, 0);
        assertEq(vault.maxRedeem(lp), 0, "standard surface reads the planted head");
        uint256 balBefore = settlementToken.balanceOf(lp);
        vm.prank(lp);
        uint256 assets = vault.claimRedeem(rC, funded, lp, lp);
        assertGt(assets, 0);
        assertEq(settlementToken.balanceOf(lp) - balBefore, assets, "explicit-id claim settles with the head pending");
        assertTrue(vault.redeemRefundPending(rB, lp));

        // (d) the operator acknowledgement lands the atom and the cooldown on the LP, not on the operator
        uint256 t = block.timestamp;
        vm.prank(delegate);
        uint256 got = vault.claimRedeemRefund(rB, lp, lp);
        assertEq(got, 1);
        assertEq(vault.lastDepositTime(lp), t, "operator path still restarts the LP's cooldown");
        assertEq(vault.lastDepositTime(delegate), 0);
        assertEq(vault.controllerRedeemHead(lp), 0);
    }

    function test_twoPlantedEpochsStackAndNeedTwoAcknowledgements() public {
        TrancheVault vault = juniorVault;
        (, uint256 lpShares) = _depositAsync(vault, lp, 5_000e6);
        _prepareSource(vault);
        uint256 rA = _plant(vault, dust, 1, lp);
        _settleAsyncRequest(rA, true);
        uint256 rB = _plant(vault, dust, 1, lp);
        assertGt(rB, rA, "the same wallet, re-funded with one atom, lands in the next epoch");
        _settleAsyncRequest(rB, true);
        assertTrue(vault.redeemRefundPending(rA, lp));
        assertTrue(vault.redeemRefundPending(rB, lp));

        _finishAsyncCooldown(vault, lp);
        vm.prank(lp);
        uint256 rC = vault.requestRedeem(lpShares / 2, lp, lp);
        _settleAsyncRequest(rC, true);
        uint256 funded = vault.claimableRedeemRequest(rC, lp);
        assertGt(funded, 0);
        assertEq(vault.maxRedeem(lp), 0);

        vm.prank(lp);
        vault.claimRedeemRefund(rA, lp, lp);
        assertEq(vault.controllerRedeemHead(lp), rB);
        assertEq(vault.maxRedeem(lp), 0, "the second planted head still masks the funded request");
        vm.prank(lp);
        vault.claimRedeemRefund(rB, lp, lp);
        assertEq(vault.controllerRedeemHead(lp), rC);
        assertEq(vault.maxRedeem(lp), funded);
        assertEq(vault.lastDepositTime(lp), block.timestamp);
    }

    /// @dev The honest limit: when any other request lands in the same epoch, the atom is funded at zero assets and a
    ///      zero-asset standard redeem clears the head with no cooldown change.
    function test_sharedEpoch_atomFundsAtZeroAssets_headClearsWithoutCooldown() public {
        TrancheVault vault = juniorVault;
        _depositAsync(vault, lp, 5_000e6);
        (, uint256 otherShares) = _depositAsync(vault, otherLp, 5_000e6);
        _prepareSource(vault);
        _finishAsyncCooldown(vault, lp);
        _finishAsyncCooldown(vault, otherLp);

        uint256 rA = _plant(vault, dust, 1, lp);
        vm.prank(otherLp);
        uint256 rOther = vault.requestRedeem(otherShares / 2, otherLp, otherLp);
        assertEq(rOther, rA, "same request window: the atom shares its epoch with a real request");
        _settleAsyncRequest(rA, true);

        assertFalse(vault.redeemRefundPending(rA, lp), "a shared epoch is funded, not refunded");
        assertEq(vault.claimableRedeemRequest(rA, lp), 1);
        assertEq(vault.claimableRedeemAssets(rA, lp), 0);
        assertEq(vault.maxRedeem(lp), 1, "standard surface shows one share, not the LP's real entitlement");
        assertEq(vault.maxWithdraw(lp), 0);
        uint256 stampBefore = vault.lastDepositTime(lp);
        vm.prank(lp);
        uint256 assets = vault.redeem(1, lp, lp);
        assertEq(assets, 0);
        assertEq(vault.controllerRedeemHead(lp), 0, "a zero-asset standard redeem clears the shared-epoch head");
        assertEq(vault.lastDepositTime(lp), stampBefore, "no cooldown change on the shared-epoch path");
    }

}
```

Run from `packages/perps` (the repo compiles with via_ir, so the first run takes a few minutes):

```bash
FOUNDRY_PROFILE=quick forge test --match-path test/poc/ForeignControllerRefundHead.t.sol -vv
```

Output:

```
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 39.52s
Compiler run successful with warnings:
Ran 5 tests for test/poc/ForeignControllerRefundHead.t.sol:ForeignControllerRefundHeadTest
[PASS] test_control_lpWithoutPlantedHeadRedeemsThroughStandardSurface() (gas: 1446936)
[PASS] test_exits_everyClearDeliversTheAtomToTheLpAndRestartsItsCooldown() (gas: 2992137)
[PASS] test_sharedEpoch_atomFundsAtZeroAssets_headClearsWithoutCooldown() (gas: 2965980)
[PASS] test_strangerPlantsRefundHead_lpStandardSurfaceDeadUntilAcknowledged() (gas: 3045589)
Logs:
  asset atoms per 1e9 share atoms: 999987
[PASS] test_twoPlantedEpochsStackAndNeedTwoAcknowledgements() (gas: 2979748)
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 14.27ms (18.42ms CPU time)
Ran 1 test suite in 20.29ms (14.27ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```
