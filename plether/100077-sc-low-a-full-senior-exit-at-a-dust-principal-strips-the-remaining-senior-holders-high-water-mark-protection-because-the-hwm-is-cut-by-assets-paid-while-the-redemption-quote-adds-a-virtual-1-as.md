# \#100077 [SC-Low] A full Senior exit at a dust principal strips the remaining Senior holders' high-water-mark protection, because the HWM is cut by assets paid while the redemption quote adds a virtual +1 asset

**Submitted on Nov 16th 2026 at 07:38:41 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100077
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The Senior high-water mark (HWM) is the target that future revenue must refill before Junior gets anything. When a Senior LP exits, HousePool cuts the HWM by `HWM * fundedAssets / seniorPrincipal`. The payout itself is priced with the vault's virtual offsets: `shares * (P + 1) / (S + 1000)`.

After a near-total Senior loss (principal `P` of a few thousand atoms), that `+1` virtual asset pays the exiter slightly more than its share fraction, less than one atom. The HWM cut multiplies this sub-atom excess by `HWM / P`, which is in the tens of millions. In the PoC a 97.55% holder exits and the HWM drops by 97.60%. **The holders who stay lose 16.62 USDG of recovery target**, out of the 1,000.68 USDG they owned. The exiter gains under one atom. The lost target goes to Junior on the next revenue.

`SECURITY.md:1082-1083` and `:1417-1418` promise the opposite: rounding is "conservative" and keeps less than one extra atom of protection per pass.

### Plain Summary

- Senior LPs have a register line that says "refill us up to 41,000 USDG before Junior gets paid". That line is the HWM.
- A crash leaves only 2,125 atoms (0.002 USDG) of Senior principal, but the register still says 41,000.
- Raju owns 97.55% of Senior shares and exits. The payout formula has a "ghost" holder (+1 atom, +1000 shares). Because the price per share is now below 1/1000 atom, the ghost's atom leaks to Raju: he gets 97.60% of the principal. That is less than one extra atom, so it looks harmless.
- But the register line is cut by 97.60% of 41,000 instead of 97.55%. That 0.05% gap is **16.62 USDG**.
- Gabbar, the locked seed holder who cannot leave, and other passive LPs now have a 984.06 USDG target instead of 1,000.68. When traders lose money later, those 16.62 USDG go to Junior.

### Vulnerability Details

**1. The payout includes the +1 virtual asset**, priced at the pre-burn principal and supply.

```solidity
// packages/perps/src/HousePool.sol:845-848
RedemptionPhase memory seniorPhase = RedemptionPhase({
@>  pricingPrincipal: seniorPrincipal,
    pricingSupply: _seniorShareSupply(),
    budget: _min(freeAssets, seniorPrincipal),
```

```solidity
// packages/perps/src/HousePool.sol:992-1000 (same at :957-959)
(uint256 fundedShares, uint256 fundedAssets) = REDEMPTION_MATH_SIDECAR.maxSharesForNetBudget(
    phase.budget - phase.fundedAssets, remainingShares,
    phase.pricingPrincipal, phase.pricingSupply,
@>  1,
@>  1000,
    feeBps
);
```

```solidity
// packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:45-46
uint256 grossAssets =
@>  Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
```

For an exit of fraction `f = shares / S`, the payout is `f * S * (P + 1) / (S + 1000)`. This is larger than `f * P` whenever `S > 1000 * P`.

**2. The HWM is cut by the asset ratio, so the excess is amplified by `HWM / P`.**

```solidity
// packages/perps/src/HousePool.sol:857-859
if (seniorPhase.fundedAssets > 0) {
    HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
@>      HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
```

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:127-130
@> nextState.seniorHighWaterMark -= Math.mulDiv(
@>     state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
@> );
   nextState.seniorPrincipal -= fundedAssets;
```

The fraction removed is about `f * (P + 1) / P`, not `f`. The HWM removed beyond pro-rata is up to `HWM * f / P` atoms. At normal impairment `HWM / P` is around 1-2, so the error is a few atoms. At a dust principal it is tens of millions of atoms.

**3. The lost target is never given back to the stayers.**
- Revenue refills Senior only up to the reduced HWM, and the rest goes to Junior (`HousePoolWaterfallAccountingLib.sol:147-160`).
- A recap fills only the reduced gap, and the rest becomes `unassignedAssets` (`HousePoolPendingPreviewLib.sol:123-135`).
- `assignUnassignedAssets` (`HousePool.sol:660`) mints new shares to a receiver. It does not top up existing holders.

**Preconditions.**
- Junior is wiped and Senior principal is dust while the HWM is still large. In the PoC this comes from an ordinary LP-adverse outcome: a hedged LONG and SHORT both close in profit. The only owner action is the documented `clearDegradedMode` runbook.
- At least one Senior holder stays. The seed receiver always does, because its floor is locked (`TrancheVault.sol:322`, `:1630-1637`).
- A full-balance exit is allowed below the 1 USDG minimum (`TrancheVault.sol:1611-1617`).

**Why not a known issue.**
- Cecuro **MED-11** (fixed) moved HWM scaling from the share ratio to this asset ratio. Its fix note says impaired withdrawals keep "proportional protection with conservative rounding". This report shows the fix breaks at a dust principal.
- Cecuro **LOW-54** (accepted) and `SECURITY.md:1417-1418` cover only the conservative direction: under one atom *kept* per pass, which helps stayers. This report is the opposite direction, with an error of `HWM / P` atoms that hurts stayers.

### Impact Details

Measured with release risk params:

| Case | Senior P (atoms) | HWM (USDG) | Exiter share | Stayers' fair HWM | HWM left | Protection lost |
|---|---|---|---|---|---|---|
| PoC (41k Senior) | 2,125 | 41,000.68 | 97.55% | 1,000.68 | 984.06 | **16.62 USDG** |
| +300k Senior LP | 6,973 | 341,001.06 | 99.71% | 1,000.69 | 978.11 | **22.59 USDG** |

The bound per funded pass is `min((1 - f) * HWM, f * HWM / P_atoms)`. At the release Senior cap (HWM = 40M USDG), P = 2,000 atoms and f = 97.5%, about **19,500 USDG** of Senior recovery target moves to Junior. With P ≤ 39 atoms, the stayers' entire 1M USDG target can go.

The loss is permanent and is not recoverable. It is paid out as cash on the next recovery inflow (revenue or recap) without any further action, and it goes to Junior instead of the Senior holders who stayed. No cash at rest is taken at the moment of the exit, and the exiter does not profit.

### Mitigation

Cut the HWM by the burned share fraction, measured against the same virtual-offset supply the quote uses, and keep the asset ratio as a ceiling:

```solidity
uint256 byAssets = Math.mulDiv(H, fundedAssets, P, Math.Rounding.Floor);
uint256 byShares = Math.mulDiv(H, fundedShares, S + 1000, Math.Rounding.Floor);
nextState.seniorHighWaterMark -= byAssets < byShares ? byAssets : byShares;
```

This keeps the MED-11 coupon case, where assets fall below the share ratio, and removes the dust-principal overshoot.

### References

- `packages/perps/src/HousePool.sol:845-848`: Senior phase priced at pre-burn principal and supply
- `packages/perps/src/HousePool.sol:957-959`, `:992-1000`: `virtualAssets = 1`, `virtualShares = 1000`
- `packages/perps/src/HousePool.sol:857-859`: `scaleSeniorOnWithdraw` called with `fundedAssets`
- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:45-46`: `shares * (P + 1) / (S + 1000)`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:127-130`: HWM cut by `fundedAssets / P`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:147-160`: revenue refills only up to the HWM, the rest goes to Junior
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:123-135`: recap fills only the gap, the rest goes to `unassignedAssets`
- `packages/perps/src/HousePool.sol:660`: `assignUnassignedAssets` mints new shares
- `packages/perps/src/TrancheVault.sol:322`, `:1630-1637`: seed floor is locked
- `packages/perps/src/TrancheVault.sol:1611-1617`: full-balance exit allowed below the minimum
- `SECURITY.md:1082-1083`, `:1417-1418`: "conservative rounding" guarantee


## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/poc/SeniorDustExitHwmErosion.t.sol` and run:

```
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path test/poc/SeniorDustExitHwmErosion.t.sol -vv
```

It uses the repo's `BasePerpTest` fixture (full stack: engine, HousePool, Senior/Junior TrancheVaults, router) with release risk params, and only real router, vault and pool flows. Pyth is mocked because it is out of scope. A binary search finds the SHORT exit price that leaves Senior with a dust principal. The non-seed Senior LP then exits with its whole balance, and the test compares the stayers' HWM against their pro-rata share.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Standalone PoC (repo Foundry suite, release risk params): at a dust Senior principal, a full Senior exit
///         removes more high-water-mark protection than its share fraction, because redemption pricing pays
///         shares * (P + 1) / (S + 1000) (HousePoolRedemptionMathLib.sol:45-47) while scaleSeniorOnWithdraw removes
///         HWM * fundedAssets / P (HousePoolWaterfallAccountingLib.sol:127-129). The +1 virtual asset is
///         amplified by HWM / P. The protection removed beyond pro-rata belongs to the Senior holders who stay.
///         Save as packages/perps/test/poc/SeniorDustExitHwmErosion.t.sol and run:
///         forge test --root packages/perps --match-path test/poc/SeniorDustExitHwmErosion.t.sol -vv
contract SeniorDustExitHwmErosionPoC is BasePerpTest {

    address internal constant A = address(0xA11CE); // winning LONG
    address internal constant B = address(0xB0B); // winning SHORT
    address internal constant SLP = address(0x1B0003); // non-seed Senior LP (exits)
    address internal constant JLP = address(0x1B0001);

    uint256 internal currentPrice;

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
        return 60_000e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _refresh(1e8);
        _fundTrader(A, 200_000e6);
        _fundTrader(B, 200_000e6);
        _fundJunior(JLP, 30_000e6);
        _fundSenior(SLP, 40_000e6);
        _refresh(1e8);
    }

    function _live(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        if (previous >= vm.getBlockTimestamp()) previous = vm.getBlockTimestamp() - 1;
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price
    ) internal {
        _mockPythUpdateData(price);
        router.updateMarkPrice(_live(price));
        currentPrice = price;
    }

    function _trade(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(who);
        router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(currentPrice);
        router.executeOrder(router.nextExecuteId(), data);
    }

    function _size(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    /// @dev Hedged A LONG / B SHORT (~90% of pool assets each). Price falls -> A closes in profit; price rises to p2
    ///      -> B closes in profit. Both are paid from pool cash. Owner clears degraded mode once the book is flat.
    function _doubleWin(
        uint256 p2
    ) internal {
        uint256 ta = pool.totalAssets();
        uint256 chunk = (ta * 1e12 * 9000 / 10_000) / 3 / 1e20 * 1e20;
        for (uint256 k; k < 3; ++k) {
            _trade(A, CfdTypes.Side.LONG, chunk, chunk / 1e12 / 4, false);
            _trade(B, CfdTypes.Side.SHORT, chunk, chunk / 1e12 / 4, false);
        }
        _refresh(0.62e8);
        _trade(A, CfdTypes.Side.LONG, _size(A), 0, true);
        _refresh(p2);
        _trade(B, CfdTypes.Side.SHORT, _size(B), 0, true);
        if (engine.degradedMode()) engine.clearDegradedMode();
        _refresh(currentPrice);
    }

    function doubleWinExt(
        uint256 p2
    ) external {
        require(msg.sender == address(this));
        _doubleWin(p2);
    }

    function test_SeniorDustExitErodesStayerHwm() public {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _refresh(1e8);

        // 1. Find the exit price that leaves Senior with a dust principal (binary search on p2, in snapshots).
        uint256 lo = 1.3e8;
        uint256 hi = 1.99e8;
        uint256 snap = vm.snapshotState();
        while (hi > lo + 1) {
            uint256 mid = (lo + hi) / 2;
            uint256 ps;
            try this.doubleWinExt(mid) {
                (ps,) = pool.getPendingDepositTrancheState();
            } catch {}
            vm.revertToState(snap);
            snap = vm.snapshotState();
            if (ps > 0) lo = mid;
            else hi = mid;
        }
        _doubleWin(lo);

        (uint256 P, uint256 J) = pool.getPendingDepositTrancheState();
        uint256 H = pool.seniorHighWaterMark();
        uint256 S = seniorVault.totalSupply();
        console2.log("pending Senior principal (atoms) / Junior principal / Senior HWM (atoms)", P, J, H);
        assertGt(P, 0);
        assertEq(J, 0);
        assertFalse(engine.degradedMode());

        // 2. The non-seed Senior LP exits with its whole balance (allowed below the 1 USDG minimum).
        uint256 shares = seniorVault.maxRequestRedeem(SLP);
        vm.prank(SLP);
        seniorVault.requestRedeem(shares, SLP, SLP);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _refresh(currentPrice);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _mockPythUpdateData(currentPrice);
        router.settleLpEpoch(_live(currentPrice));

        uint256 P1 = pool.seniorPrincipal();
        uint256 H1 = pool.seniorHighWaterMark();
        uint256 S1 = seniorVault.totalSupply();
        uint256 fairH1 = H * S1 / S; // HWM that the remaining shares owned before the exit
        console2.log("exiter share fraction (bps)", shares * 10_000 / S);
        console2.log("funded to exiter (atoms) / exiter fraction of principal (bps)", P - P1, (P - P1) * 10_000 / P);
        console2.log("stayers' HWM: owned before exit / left after exit", fairH1, H1);
        console2.log("HWM protection removed from stayers beyond pro-rata (atoms)", fairH1 - H1);

        // Stayers' restoration claim must not shrink by more than rounding dust (SECURITY.md:1082, :1417).
        assertLe(fairH1 - H1, 1e6, "stayers lost > 1 USDG of HWM protection to another LP's exit");
    }

}
```

Result (actually run, `FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path test/poc/SeniorDustExitHwmErosion.t.sol -vv`):

```
Ran 1 test for test/poc/SeniorDustExitHwmErosion.t.sol:SeniorDustExitHwmErosionPoC
[FAIL: stayers lost > 1 USDG of HWM protection to another LP's exit: 16623530 > 1000000] test_SeniorDustExitErodesStayerHwm() (gas: 512149876)
Logs:
  pending Senior principal (atoms) / Junior principal / Senior HWM (atoms) 2125 0 41000684942
  exiter share fraction (bps) 9755
  funded to exiter (atoms) / exiter fraction of principal (bps) 2074 9760
  stayers' HWM: owned before exit / left after exit 1000684941 984061411
  HWM protection removed from stayers beyond pro-rata (atoms) 16623530
```

The test fails on purpose. The assertion is the intended invariant (stayers lose at most 1 USDG of HWM to someone else's exit), and the code breaks it by 16.62 USDG.

