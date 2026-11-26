# \#97669 [SC-Insight] Harmonise the free-cash floor across the four `HousePool.payOut` callers: the open-path rebate is the only one that does not apply it

**Submitted on Nov 2nd 2026 at 16:01:52 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97669
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

**No vulnerability is identified in this report and none is claimed.** It is an informational consistency observation, filed as a Security Best Practices Insight. The measurements in the Impact Details section are included specifically to show that the current code is **not** exploitable under the release parameters, and to document the margin by which that holds.

`HousePool.payOut` is the only route by which LP cash reaches a trader, and it carries no internal reserve check. Every bound lives in the caller. There are four callers. Three of them cap the payout at the pool's free cash. The fourth, the open-path rebate, does not.

The planner already computes the reserved figure that the other callers respect, and applies it two code paths away to the fee top-up. The field's own NatSpec scopes the protection to fee top-ups in as many words, so the current behaviour is self-consistent with its documentation. The recommendation is to extend the same floor to the rebate branch, which costs one helper call and makes all four egress sites agree.

## Vulnerability Details

This section describes code structure, not an exploit path.

### The egress carries no internal check

`HousePool.sol:604-614`:

```solidity
function payOut(address recipient, uint256 amount) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) revert HousePool__Unauthorized();
    accountedAssets -= amount;
    settlementToken.safeTransfer(recipient, amount);
}
```

Four callers, three bounds:

| Caller | Bound |
| --- | --- |
| `CfdEngineSettlementSidecar:924`, fee top-up | `min(poolFunded, pool.getSettlementFreeAssets())`, the complete floor |
| `CfdEngineSettlementSidecar:622`, close rebate | `min(actionRebate, freePoolCash)` via `CashPriorityLib.reserveFreshPayouts`, excess waived |
| `CfdEngine:238`, claim payment | `pool.totalAssets() >= totalTraderClaimBalance`, gated on total assets |
| `CfdEngineSettlementSidecar:443`, open rebate | no free-cash cap |

The open amount is set in `CfdEnginePlanLib:333` with no bound beyond sign:

```solidity
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

The close path caps and waives the excess. The open path does neither.

### The planner already holds the figure

The planner builds the reserved term (`CfdEngineSettlementSidecar:193`, identically at `CfdOrderPolicyEvaluatorBase:231`):

```solidity
snap.poolAdditionalReservedAssets =
    pool.pendingRecapitalizationAssets() + pool.pendingTradingRevenueAssets() + pool.unassignedAssets();
```

and feeds it into `_freePoolCashAfterLiabilityReserves` (`CfdEnginePlanLib:1022-1033`):

```solidity
uint256 reservedAssets = maxLiabilityAssets + traderClaimsAssets
    + SolvencyAccountingLib.settlementBufferTargetAssets(maxLiabilityAssets, snap.settlementBufferBps)
    + snap.poolAdditionalReservedAssets + newRevenueReserveAssets;
return poolCashAssets > reservedAssets ? poolCashAssets - reservedAssets : 0;
```

That helper is consulted at one place in the open plan, line 742, to cap the fee top-up. The rebate branch, twenty lines earlier at `:729-733`, tests against total pool cash only:

```solidity
uint256 rebateAssets = uint256(-cashDeltaBeforeTopUp);
if (rebateAssets > snap.poolCashAssets) {
    return false;
}
poolCashBeforeTopUp = snap.poolCashAssets - rebateAssets;
```

The field's NatSpec, `CfdEnginePlanTypes.sol:90`, states the scope:

> "@param poolAdditionalReservedAssets Pending claimant inflows and unassigned assets protected from **fee top-ups**."

The code matches its documentation. The observation is that the narrower scope is unnecessary, and that widening it removes a discrepancy between two views of the same floor.

### The two views of the floor differ

`HousePool.getSettlementFreeAssets()` (`:1242-1255`) reserves `unassignedAssets` and `_pendingClaimantBucketAssets()`:

```solidity
return _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;
```

The post-operation solvency gate reserves `L`, the buffer and trader claims, and neither of the two above. So the pool view and the open plan disagree on the floor by `unassignedAssets + pendingRecapitalizationAssets + pendingTradingRevenueAssets + pendingClaimantBucket`.

## Impact Details

**There is no impact.** The measurements below exist to establish that, and to quantify the margin.

Release parameters matter: the default test harness uses `vpiFactor: 0`, which cancels every rebate and would make any measurement vacuous. All figures use `vpiFactor 0.01e18` and `maxSkewRatio 0.4e18` from `DEPLOYMENT.md`.

Three states, each pushing the rebate harder. The second and third use the fact that `L = max(longMaxProfit, shortMaxProfit)` is a max rather than a sum, so opening the non-dominant side does not raise `L` and clears `E >= L + B` even at maximal utilisation, while that same open flattens skew and so earns the rebate.

| State | `L` | Free cash | Paid by the pool | Ratio |
| --- | --- | --- | --- | --- |
| Plain contra-skew open | 200,500 | 801,699 | 199.56 | 1 : 4,018 |
| Fresh position, 250k contra | 300,000 | n/a | 436.43 | measured in the cycle test |
| Filled to both guards, 300k non-dominant open admitted | 975,000 | 25,343 | 734.46 | 1 : 34 |

All figures in USDG. In the third state the open was admitted, `L` did not move, and the pool paid 734.46 USDG with 24,608 USDG of protected margin still standing. **The floor was never breached in any state.**

The open and close legs net out. Three consecutive cycles on a fresh position give `-436.43` at open and `+436.62` at close, so `+0.19` per cycle **to** the pool. The lifetime VPI clamp holds: the rebate is a timing effect, not new money.

The ceiling is structural. Maximum rebate is approximately `0.5 * vpiFactor * S^2 / D` with skew `S` bounded by `maxSkewRatio * D`, which at release values is `8e-4 * D`, roughly 800 USDG on a 1,000,000 USDG pool. The measured 734.46 is close to that ceiling, so the third row is the mechanism at its maximum rather than an incomplete attempt.

Each quantity the two views disagree about is also out of an unprivileged caller's reach in a live pool, which is the second reason there is no impact:

| Quantity | Writer | Reachable without a trusted role |
| --- | --- | --- |
| `pendingRecapitalizationAssets` | `CfdEngine.recapitalizePool`, `onlyOwner` | no |
| `pendingTradingRevenueAssets` | queued only when `seniorPrincipal + juniorPrincipal == 0` (`HousePool:646`) | bootstrap pool only |
| `unassignedAssets` | `juniorRevenueWithoutOwners` (`HousePool:1594`, `:1668`) | junior principal with no vault holders |

### The counter-argument, stated and answered

A reviewer may observe that the open rebate is not unbounded at all: it is bounded indirectly, by the lifetime VPI clamp in `CloseAccountingLib` which guarantees no position extracts net VPI over its life, and by the post-operation solvency gate which covers the liability side. **That observation is correct**, it is what the measurements below confirm, and it is the reason this is filed as an Insight and not as a bug. The missing cap is not a hole.

The observation worth recording is narrower. The other three callers have a **direct** bound, expressed at the call site in terms of the same quantity `getSettlementFreeAssets()` reports. The open rebate has an **indirect** one, distributed across two other contracts and conditional on two governable parameters. Those are not equivalent properties to maintain, even when they currently coincide, and the discrepancy means `getSettlementFreeAssets()` and the open plan do not describe the same floor.

### Recommendation

Route `delta.poolRebatePayoutAssets` through `_freePoolCashAfterLiabilityReserves`, the helper the fee top-up already uses two code paths away, and waive the excess the way `CashPriorityLib.reserveFreshPayouts` does on close.

The value of doing so is defence in depth and uniformity rather than closing a live hole. Three of four egress sites enforce one floor; a fourth that does not is a pattern that invites being copied by the next caller added, and the ceiling that currently keeps the gap harmless is a function of two governable parameters (`vpiFactor`, `maxSkewRatio`) rather than of the code. `HousePool.activateTrading` (`:532-551`) already requires `unassignedAssets == 0` at first activation, so the quantity is understood elsewhere in the codebase to be worth protecting. Harmonising the four callers makes that protection hold regardless of future parameter choices, and removes the need to reason about the discrepancy between `getSettlementFreeAssets()` and the open plan.

## References

- `packages/perps/src/HousePool.sol#L604-L614`, `payOut`, no internal reserve check
- `packages/perps/src/HousePool.sol#L1242-L1255`, `getSettlementFreeAssets`, the protected floor
- `packages/perps/src/HousePool.sol#L532-L551`, `activateTrading` requires `unassignedAssets == 0`
- `packages/perps/src/HousePool.sol#L646`, trading-revenue queueing condition
- `packages/perps/src/HousePool.sol#L1594`, `#L1668`, `unassignedAssets` writers
- `packages/perps/src/libraries/CfdEnginePlanLib.sol#L333`, the uncapped rebate amount
- `packages/perps/src/libraries/CfdEnginePlanLib.sol#L729-L733`, the rebate tested against total cash
- `packages/perps/src/libraries/CfdEnginePlanLib.sol#L742`, the free-cash cap applied to the fee top-up
- `packages/perps/src/libraries/CfdEnginePlanLib.sol#L1022-L1033`, `_freePoolCashAfterLiabilityReserves`
- `packages/perps/src/CfdEnginePlanTypes.sol#L90`, the NatSpec scoping the protection to fee top-ups
- `packages/perps/src/CfdEngineSettlementSidecar.sol#L193`, `#L443`, `#L622`, `#L924`, the four bounds
- `packages/perps/src/CfdEngine.sol#L238`, claim payment gate
- `packages/perps/DEPLOYMENT.md`, release `vpiFactor 0.01e18` and `maxSkewRatio 0.4e18`

## Proof of Concept

## Proof of Concept

Runnable code is not required for an Insight, but it is supplied, because the point of this report is the measured margin rather than a claim. The file below is complete: one new test file, nothing attached, no in-scope contract modified. It extends `BasePerpTest`, which already exists at `packages/perps/test/perps/BasePerpTest.sol`. The only mocks are Pyth and USDG, both from that harness.

**When this Insight is valuable.** It is valuable at the moment any of the following is considered, because each one removes the condition that currently makes the gap harmless, and the report says in advance which line to change:

1. A governance change to `vpiFactor` or `maxSkewRatio`. The rebate ceiling scales as `vpiFactor * maxSkewRatio^2`, so the 1 : 34 margin measured below is a parameter-dependent quantity, not a code-guaranteed one.
2. Any change that makes `unassignedAssets` or the pending claimant bucket reachable without a trusted role.
3. The addition of a fifth `payOut` caller, where the open-path pattern is the one most likely to be copied, being the shortest.
4. A review of owner recapitalisation flow, since funds queued by `recapitalizePool` sit in `pendingRecapitalizationAssets`, which the rebate branch does not reserve.

Save at `packages/perps/test/perps/accounting/open/OpenRebateUnderFullUtilisation.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";

import {BasePerpTest} from "../../BasePerpTest.sol";

/// @notice Drives the open-path pool rebate to its structural maximum against the floor that
///         `getSettlementFreeAssets()` protects, and measures the remaining margin. This is a measurement
///         harness: it asserts no violation, because none occurs under release parameters.
///
///  (1) `L = max(longMaxProfit, shortMaxProfit)` is a MAX, not a sum. While the non-dominant side stays strictly
///      below the dominant one, opening it does not increase `L`, so it clears `E >= L + B` even at maximal
///      utilisation.
///  (2) The open-path pool rebate (`poolRebatePayoutAssets`, `CfdEngineSettlementSidecar:443`) leaves pool cash
///      without the free-cash cap that the fee top-up and the close rebate apply.
///
/// Target optimum, with `cap = 2x entry price` (so `maxProfit = notional` on both sides):
///   free cash  ~= D - S_n*(1+b)                  -> 0 as S_n -> 0.9975*D
///   max rebate ~= 0.5*k*S^2/D  with S <= 0.4*D   -> 8e-4*D
/// which calls for a LONG/SHORT ratio of 0.61: skew holds at 39 percent, below the 40 percent cap, while `L`
/// climbs to the solvency guard.
///
/// @dev RELEASE parameters. The default harness uses `vpiFactor: 0`, which would cancel every rebate.
contract OpenRebateUnderFullUtilisationTest is BasePerpTest {

    address internal heavy = address(0x4EA71);
    address internal light = address(0xC0147E);
    address internal lastMover = address(0x1A57);

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

    function _sideNotionals() private view returns (uint256 longMax, uint256 shortMax) {
        (longMax,,,) = engine.sides(0);
        (shortMax,,,) = engine.sides(1);
    }

    function _tryOpen(address who, CfdTypes.Side side, uint256 size) private returns (bool ok) {
        (ok,) = _tryOpenReason(who, side, size);
    }

    /// @dev Also returns the typed failure code, so the binding constraint is read rather than guessed.
    function _tryOpenReason(
        address who,
        CfdTypes.Side side,
        uint256 size
    ) private returns (bool ok, uint8 code) {
        try this.externalOpen(who, side, size) {
            ok = true;
        } catch (bytes memory err) {
            ok = false;
            // CfdEngine__TypedOrderFailure(uint8 kind, uint8 code, bool isClose)
            if (err.length >= 4 + 96) {
                uint256 word;
                assembly {
                    word := mload(add(err, 68))
                }
                code = uint8(word);
            }
        }
    }

    /// @dev External wrapper: lets a failed open be caught without failing the test.
    function externalOpen(address who, CfdTypes.Side side, uint256 size) external {
        require(msg.sender == address(this), "self only");
        _open(who, side, size, 400_000e6, 1e8);
    }

    /// @dev Fills to the contact point of both guards: SHORT dominant, LONG at 61 percent to hold skew at 39.
    function _fillToTheBrim() private {
        uint256 step = 100_000e18;
        for (uint256 i; i < 40; ++i) {
            if (!_tryOpen(heavy, CfdTypes.Side.SHORT, step)) break;
            _tryOpen(light, CfdTypes.Side.LONG, (step * 61) / 100);
        }
        // Fine fill: eat the last granularity step up to the guard.
        uint256 fine = 50_000e18;
        while (fine >= 10e18) {
            for (uint256 j; j < 6; ++j) {
                if (!_tryOpen(heavy, CfdTypes.Side.SHORT, fine)) break;
                _tryOpen(light, CfdTypes.Side.LONG, (fine * 61) / 100);
            }
            fine /= 2;
        }
    }

    function test_OpenRebate_AgainstTheReservedFloor() public {
        _fundTrader(heavy, 60_000_000e6);
        _fundTrader(light, 60_000_000e6);
        _fundTrader(lastMover, 60_000_000e6);

        _fillToTheBrim();

        (uint256 longMax, uint256 shortMax) = _sideNotionals();
        uint256 freeBefore = pool.getSettlementFreeAssets();
        uint256 poolBefore = pool.totalAssets();
        emit log_named_uint("longMaxProfit       ", longMax);
        emit log_named_uint("shortMaxProfit (L)  ", shortMax);
        emit log_named_uint("weak-side headroom  ", shortMax > longMax ? shortMax - longMax : 0);
        emit log_named_uint("pool totalAssets    ", poolBefore);
        emit log_named_uint("FREE CASH           ", freeBefore);

        // Open the non-dominant side: skew flattens, `L` is unchanged, so the solvency guard is untouched.
        CfdTypes.Side weakSide = longMax > shortMax ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;

        // Descending ladder in SIZE units, multiples of the 100-token quantum. Passing a maxProfit figure
        // (6 decimals) as a size (18 decimals) yields INVALID_SIZE_QUANTUM, code 8, and measures nothing.
        uint256[7] memory sizes =
            [uint256(300_000e18), 200_000e18, 100_000e18, 50_000e18, 10_000e18, 1_000e18, 100e18];
        bool opened;
        for (uint256 i; i < sizes.length; ++i) {
            uint8 code;
            (opened, code) = _tryOpenReason(lastMover, weakSide, sizes[i]);
            if (opened) {
                emit log_named_uint("admitted size       ", sizes[i]);
                break;
            }
            emit log_named_uint("refused size        ", sizes[i]);
            emit log_named_uint("  failure code      ", code);
        }

        uint256 poolAfter = pool.totalAssets();
        if (poolAfter < poolBefore) {
            uint256 paid = poolBefore - poolAfter;
            emit log_named_uint("PAID BY THE POOL    ", paid);
            emit log_named_string("BELOW THE FLOOR     ", paid > freeBefore ? "*** YES ***" : "no");
            if (paid > freeBefore) {
                emit log_named_uint("  overshoot         ", paid - freeBefore);
            } else {
                emit log_named_uint("  remaining margin  ", freeBefore - paid);
            }
        } else {
            emit log("the pool paid nothing on this open");
        }
    }

    /// @notice Separate angle: does a FRESH position collect pool cash, and does it give it back on close?
    function test_FreshPositionRebateCycle() public {
        _fundTrader(heavy, 10_000_000e6);
        _fundTrader(lastMover, 10_000_000e6);

        _open(heavy, CfdTypes.Side.SHORT, 300_000e18, 200_000e6, 1e8);

        uint256 poolStart = pool.totalAssets();
        emit log_named_uint("pool at start       ", poolStart);

        for (uint256 cycle; cycle < 3; ++cycle) {
            uint256 poolPre = pool.totalAssets();
            _open(lastMover, CfdTypes.Side.LONG, 250_000e18, 200_000e6, 1e8);
            uint256 poolMid = pool.totalAssets();
            _close(lastMover, CfdTypes.Side.LONG, 250_000e18, 1e8);
            uint256 poolPost = pool.totalAssets();
            emit log_named_int("  delta at open     ", int256(poolMid) - int256(poolPre));
            emit log_named_int("  delta at close    ", int256(poolPost) - int256(poolMid));
            emit log_named_int("  delta of cycle    ", int256(poolPost) - int256(poolPre));
        }

        emit log_named_int("TOTAL POOL DELTA    ", int256(pool.totalAssets()) - int256(poolStart));
    }

}
```

### Run

```
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init lib/forge-std lib/openzeppelin-contracts lib/morpho-blue
# save the file above at packages/perps/test/perps/accounting/open/OpenRebateUnderFullUtilisation.t.sol
forge test --root packages/perps \
  --match-path "test/perps/accounting/open/OpenRebateUnderFullUtilisation.t.sol" -vv
```

`--match-path` is required: in Foundry it filters compilation, whereas `--match-test` only filters execution, and a full via-IR compile of the suite exhausts memory. Initialise the submodules without `--recursive`, since the nested OpenZeppelin submodules exceed the Windows path limit.

### Output

```
[PASS] test_FreshPositionRebateCycle()
  pool at start       : 1002449101796
    delta at open     : -436431135
    delta at close    : 436621225
    delta of cycle    : 190090
    delta at open     : -436431053
    delta at close    : 436621143
    delta of cycle    : 190090
    delta at open     : -436430970
    delta at close    : 436621059
    delta of cycle    : 190089
  TOTAL POOL DELTA    : 570269

[PASS] test_OpenRebate_AgainstTheReservedFloor()
  longMaxProfit       : 579500000000
  shortMaxProfit (L)  : 975000000000
  weak-side headroom  : 395500000000
  pool totalAssets    : 1002780551224
  FREE CASH           : 25343051224
  admitted size       : 300000000000000000000000
  PAID BY THE POOL    : 734457803
  BELOW THE FLOOR     : no
    remaining margin  : 24608593421

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

`BELOW THE FLOOR: no` and `TOTAL POOL DELTA: +570269` are the two results that matter: the floor is not breached, and the pool ends the cycles ahead. The source above was extracted from this report and run as printed.

### Provenance

```
$ git rev-parse HEAD
a82d4fefb02768e5b3bddcc97dee371eee24fea8

$ git status --porcelain -- packages/perps/src
(empty)
```

Nothing under `packages/perps/src` is touched. The evidence is a single new, untracked test file.

