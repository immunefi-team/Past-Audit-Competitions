# \#98961 [SC-Low] Senior redemption at a few atoms of Senior principal removes more high-water mark than the redeemed share: an exit of half the shares cancels 2/3 of the recovery protection (99,903 USDG beyond its share in the test)

**Submitted on Nov 11th 2026 at 07:15:58 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98961
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When a Senior redemption is funded, `HousePool` reduces the Senior high-water mark (HWM) by the fraction "funded assets / Senior principal". After an almost complete wipe, Senior principal is a few atoms while the HWM still records the full pre-loss value. Funded assets are then an integer number of atoms that includes the +1 virtual asset of the share quote, so the fraction removed from the HWM is much larger than the fraction of shares that left. In the test, holders of 50.01 % of the Senior shares redeem at a principal of 3 atoms: 2 atoms are paid and 2/3 of the HWM is removed. The holders who stay keep one third of the recovery protection for half of the shares: 99,903 USDG less than their share.

## Vulnerability Details

`HousePool._settleLpEpoch` (`HousePool.sol:845-863`) funds matured Senior redemptions and then calls:

```solidity
HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
```

`scaleSeniorOnWithdraw` (`HousePoolWaterfallAccountingLib.sol:119-131`; its NatSpec: "Removes senior principal and the same economic fraction of its high-water-mark entitlement"):

```solidity
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

`fundedAssets` comes from the redemption quote `floor(shares * (principal + 1) / (supply + 1000))` (`HousePoolRedemptionMathLib.sol:31-47`, virtual offsets 1 asset / 1000 shares passed at `HousePool.sol:957-959`).

With principal `P`, HWM `H`, share supply `S` and redeemed shares `x`, the HWM removed is `floor(H * A / P)` with `A = floor(x * (P + 1) / (S + 1000))`. At ordinary scale `A / P` equals `x / S` to within one part in `P`. At a principal of a few atoms it does not: with `P = 3` and `x / S = 50.01 %`, `A = floor(0.5001 * 4) = 2`, so `A / P = 2/3`. The HWM is not a few atoms: a loss never lowers it (`absorbLoss`, same library, leaves it "deliberately unchanged"), so one atom of principal carries `H / P` of protection, here about 200,000 USDG.

The HWM is what later recovery restores Senior to: a recapitalization restores Senior principal "only up to its high-water mark" and the remainder becomes unassigned (`HousePoolPendingPreviewLib.sol:103-134`). Removing more HWM than the exiting share takes recovery away from the holders who stay.

Measured in the PoC (USDG):

| | value |
|---|---|
| Senior principal / HWM before the exit | 0.000003 / 600,009.221390 |
| Shares redeemed | 50.01 % of the supply |
| Paid to the exiting holder | 0.000002 |
| HWM after the exit | 200,003.073797 |
| HWM after a share-proportional reduction | 299,906.479990 |
| Staying holder if the owner then recapitalizes the pre-exit HWM: without the exit / with it | 299,905.478410 / 200,002.405859 |
| `unassignedAssets` after that recapitalization | 400,006.147594 |

Same route, other remainders, exits of 83 to 85 % of shares (extended test): removed beyond the exiting share 45,455.45 USDG at 11 atoms, 5,008.84 at 101, 892.82 at 571, 509.08 at 1,001, 0.51 at 1 USDG. The error has both signs: 16 small exits in one settlement at `P = 101`, each quoted 1.99 atoms and paid 1, kept 92,246 USDG of HWM above the share-proportional value.

### How the state is reached

The state is rare: both tranches wiped to a remainder of a few atoms, no position open. In the PoC every step is a public call. A LONG and a SHORT of 720,000 tokens each open at 1.00; the basket goes to 0.50 and the LONG takes its 360,000 USDG gain (degraded mode latches); the basket goes to about 1.61 and the SHORT takes the rest. The remainder is decided by the winning trader's last close: with a final 1-lot close, one 1e-8 price tick moves the pool remainder by one atom, and the trader chooses the moment of the order and the signed print used. The fixture's feeds give 1e-8 control (an upper bound); on a 1e-5 grid the closest remainder was 571 atoms.

The owner then calls `clearDegradedMode()`. The documents allow it as soon as raw `E >= L` (`EMERGENCY_RESPONSE_GUIDE.md:45-47`; `SECURITY.md:1509-1511`: "Recapitalize or clear positions", then clear), which holds once no position is open. The next settlement books the loss (Junior 0, Senior 3 atoms, HWM unchanged); in the PoC the redemption is requested after it and funded by the following settlement. If the owner recapitalizes before clearing, the recapitalization is applied in the same reconcile as the loss and the window does not open.

## Impact Details

- Who loses: the Senior holders who do not redeem. Their recovery protection is cut by up to `H / (P + 1)` more than the exiting share: 99,903 USDG of a 600,009 USDG HWM at 3 atoms.
- Where the value goes: nowhere at the exit (the redeemer is paid atoms). If the owner later recapitalizes the pre-exit HWM, the part above the reduced HWM is booked as `unassignedAssets` (400,006 USDG in the PoC).
- Who can trigger it: Senior redemptions of one epoch whose shares together quote at least one atom (at 3 atoms of principal: from 25 % of the supply; smaller requests quote zero and are refunded; from 75 % the whole principal is funded and the HWM is cleared, the documented full-funding rule). The redeemer does not profit.
- Remedy: the staying holders have no call that prevents or reverses it. The owner's `assignUnassignedAssets` (`HousePool.sol:660-704`) cannot return the surplus to them as Senior: towards Senior it reverts `HousePool__ExceedsSeniorDepositCapacity` while Junior principal is zero, and towards Junior it mints the whole amount to one receiver (both executed, separate test).
- Limits: at 1 USDG of principal or more the effect is below 1 USDG. The large figures need a remainder of a few atoms, in practice a winning trader who chooses the last print.

Proposed severity: Low. No funds move at the exit and the caller does not profit; the remaining holders lose recovery protection, realised when the owner recapitalizes.

### What the documents say

- `ACCOUNTING_SPEC.md:580-583`: "settlement sets `P' = P - A` and `H' = H - floor(H * A / P)`. [...] retains impaired protection proportionally [...] Rounding favors the remaining protected claim". The formula is implemented as written, and the repository tests (`testFuzz_ImpairedWithdrawalPreservesRecoveryRatio`; `test_ImpairedSeniorFrozenExitScalesHwmByFundedPrincipal` at 100,000 USDG) assert it against principal. Neither the text nor the tests cover ownership at a few atoms, where 2/3 of the principal is funded for 50.01 % of the shares and the protection per remaining share falls by one third.
- CECURO MED-11 (fixed in 9ab65f41) replaced the earlier rule (HWM reduced by funded shares over pre-burn supply) by this formula: with the virtual offsets, shares and assets are different fractions, and a healthy exit could understate the HWM (`SECURITY.md:1082-1083`: "healthy withdrawals cannot create impairment through virtual-offset pricing"). For the impaired case the response promises "proportional protection with conservative rounding". At a few atoms the same divergence acts the other way; the earlier rule (plether-core a27cf98a, `AsyncLpEpochSettlement.t.sol:617`: "HWM must burn by funded-share ownership") gives the share-proportional 299,906.48 here.
- CECURO LOW-54 and `SECURITY.md:1417-1419` accept that an impaired withdrawal "may retain less than one extra USDG atom of high-water-mark protection per funded pass". That residual is the floor in `floor(H * A / P)`. Here the deviation comes from `A` itself, is in the removing direction, and is not bounded by one atom. CECURO LOW-63 accepts virtual-offset rounding "at normal operating scales".

## Recommendation

While Senior is impaired and principal remains after the exit, do not remove a larger fraction of the HWM than the fraction of the pre-burn share supply that was funded (`seniorPhase.fundedShares`, `seniorPhase.pricingSupply` in `_settleLpEpoch`):

```solidity
uint256 hwmBefore = seniorHighWaterMark;
bool impaired = hwmBefore > seniorPrincipal;
nextState = HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
if (impaired && nextState.seniorPrincipal > 0) {
    uint256 hwmFloor = hwmBefore - hwmBefore * seniorPhase.fundedShares / seniorPhase.pricingSupply;
    if (nextState.seniorHighWaterMark < hwmFloor) nextState.seniorHighWaterMark = hwmFloor;
}
```

On a patched `HousePool` at the pool's address, the PoC's exit leaves the HWM at 299,906.479990 and the staying holder at 299,905.478410, equal to the control; a healthy exit still gives `H' = P'`; an impaired exit at ordinary scale gives `H - floor(H * A / P)` as before; an exit that funds the whole principal still clears the HWM. It covers the removing direction only (not the 92,246 USDG retaining case). The repository's LP test contracts (53 contracts, 284 tests of `spec/lp`, `accounting/waterfall` and `accounting/cash-priority`) pass on the patched pool; the rest of the suite was not run.

## References

- `packages/perps/src/HousePool.sol:845-863`, `:957-959`, `:660-704`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:119-131`
- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:31-47`
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-134`
- `packages/perps/ACCOUNTING_SPEC.md:580-583`, `SECURITY.md:1082-1083`, `:1417-1419`, `:1509-1511`
- CECURO audit: MED-11 (fix commit 9ab65f41), LOW-54, LOW-63

## Proof of Concept

## Proof of Concept

Uses the repository's `BasePerpTest` (protocol contracts, release risk parameters; `MockPyth` prices, test USDG). No in-scope contract is modified.

1. Save the file below as `packages/perps/test/perps/SeniorHwmAtomExit.t.sol` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
2. From the repository root: `forge test --root packages/perps --match-contract SeniorHwmAtomExit -vv` (`FOUNDRY_VIA_IR=false` compiles faster; both pipelines pass).
3. Expected: 1 passed, with this output (USDG atoms, 6 decimals):

```
HWM before / after the exit / share-proportional 600009221390 200003073797 299906479990
STAY after recovery: control / after the exit 299905478410 200002405859
unassignedAssets after recovery 400006147594
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Plether perps @ a82d4fe, release parameters. Trades: OrderRouter.commitOrder + executeOrder by a keeper. LP actions:
// TrancheVault requests, OrderRouter.settleLpEpoch. Owner: the documented clearDegradedMode and recapitalizePool.
// Mocks: MockPyth payloads and USDG. No in-scope contract is modified.

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract SeniorHwmAtomExit is BasePerpTest {
  address constant KEEPER = address(0xC0FFEE);
  address constant LONG_T = address(0x10A6);
  address constant SHORT_T = address(0x5407);
  address constant STAY = address(0x57A1); // Senior, stays: 299,900 USDG
  address constant EXIT = address(0xE817); // Senior, exits: 300,100 USDG
  address constant JA = address(0x7A); // Junior holders
  address constant JB = address(0x7B);
  uint256 lastPub;
  uint256 px = 1e8;
  OrderRouter r;

  function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
    return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
  }

  function _initialJuniorDeposit() internal pure override returns (uint256) {
    return 0;
  }

  function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
    return 1e6;
  }

  function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
    return 1e6;
  }

  function setUp() public virtual override {
    super.setUp();
    r = OrderRouter(address(router));
    IOrderRouterAdminHost.RouterConfig memory c = _routerConfig();
    c.adverseConfidenceMultiplierBps = 2500;
    c.minOpenNotionalAssets = 1000e6;
    _setRouterConfig(c);
    _setSeniorCapacity(40_000_000e6, 8000);
    baseMockPyth.setSynchronizeLegacyUniquePrices(false);
    uint256 d = vm.getBlockTimestamp() / 1 days;
    uint256 a = (8 - (d + 4) % 7) % 7;
    vm.warp((d + (a == 0 ? 7 : a)) * 1 days + 6 hours); // Monday 06:00 UTC
  }

  function test_halfOfTheSharesExit_twoThirdsOfTheHwmGone() public {
    _reachThreeAtomsOfSeniorPrincipal();
    uint256 h0 = pool.seniorHighWaterMark();
    assertEq(pool.seniorPrincipal(), 3, "Senior principal: 3 atoms");
    assertGt(h0, 600_000e6); // the HWM still records the deposits
    uint256 snap = vm.snapshotState();

    // Control: nobody exits; the owner recapitalizes the HWM and one settlement books it.
    uint256 stayControl = _recapitalizeAndValueOf(STAY, h0);
    assertEq(pool.seniorPrincipal(), h0, "control: restored to the HWM");
    vm.revertToState(snap);

    // EXIT redeems all its Senior shares.
    uint256 x = seniorVault.balanceOf(EXIT);
    uint256 supply = seniorVault.totalSupply();
    vm.prank(EXIT);
    uint256 id = seniorVault.requestRedeem(x, EXIT, EXIT);
    _settle(id);
    uint256 h1 = pool.seniorHighWaterMark();
    assertEq(x * 10_000 / supply, 5001, "50.01 % of the shares");
    assertEq(pool.seniorPrincipal(), 1, "2 of the 3 atoms were paid");
    assertEq(h1, h0 - h0 * 2 / 3, "HWM cut by 2/3");
    uint256 proportional = h0 - h0 * x / supply;

    uint256 stayAfter = _recapitalizeAndValueOf(STAY, h0);
    assertEq(pool.seniorPrincipal(), h1, "Senior restored to the reduced HWM only");
    console2.log("HWM before / after the exit / share-proportional", h0, h1, proportional);
    console2.log("STAY after recovery: control / after the exit", stayControl, stayAfter);
    console2.log("unassignedAssets after recovery", pool.unassignedAssets());
    assertGt(proportional - h1, 99_000e6, "HWM removed beyond the exiting share");
    assertGt(stayControl - stayAfter, 99_000e6, "STAY restored over 99,000 USDG short");
  }

  /// Hedged pair at 1.00; LONG takes its gain at 0.50; SHORT closes near 1.61 and its last 1-lot print leaves 3 atoms.
  function _reachThreeAtomsOfSeniorPrincipal() internal {
    _mark(1e8);
    _lp(juniorVault, JA, 100_000e6);
    _lp(juniorVault, JB, 100_000e6);
    _lp(seniorVault, STAY, 299_900e6);
    _lp(seniorVault, EXIT, 300_100e6);
    _fundTrader(LONG_T, 300_000e6);
    _fundTrader(SHORT_T, 800_000e6);
    for (uint256 i; i < 3; ++i) {
      _trade(LONG_T, false, CfdTypes.Side.LONG, 2400, 80_000e6, 1e8);
      _trade(SHORT_T, false, CfdTypes.Side.SHORT, 2400, 240_000e6, 1e8);
    }
    _trade(LONG_T, true, CfdTypes.Side.LONG, 7200, 0, 0.5e8);
    _closeShortLeaving(7172, 1000e6);
    _closeShortLeaving(20, 200e6);
    _closeShortLeaving(7, 60e6);
    // 1 lot = 100 tokens: one 1e-8 tick of the last close moves the pool remainder by 1 atom. The closing trader
    // picks the print; the loop only finds the tick that leaves 3 atoms.
    uint256 p = 1e8 + _equity() - 3;
    for (uint256 i; i < 16; ++i) {
      uint256 s = vm.snapshotState();
      _trade(SHORT_T, true, CfdTypes.Side.SHORT, 1, 0, p);
      uint256 left = _equity();
      if (left == 3) break;
      vm.revertToState(s);
      p = left > 3 ? p + (left - 3) : p - (3 - left);
    }
    assertEq(_equity(), 3, "remainder set by the last print");
    engine.clearDegradedMode(); // owner, documented: no position is open
    _settleWith(JA); // books the loss: Junior 0, Senior 3 atoms, HWM unchanged
  }

  function _recapitalizeAndValueOf(address lp, uint256 amount) internal returns (uint256) {
    settlementToken.mint(address(this), amount);
    settlementToken.approve(address(engine), amount);
    engine.recapitalizePool(amount);
    _settleWith(JB);
    return seniorVault.balanceOf(lp) * (pool.seniorPrincipal() + 1) / (seniorVault.totalSupply() + 1000);
  }

  /// A worthless Junior exit gives the settlement queue an item to process.
  function _settleWith(address j) internal {
    uint256 shares = juniorVault.balanceOf(j);
    vm.prank(j);
    _settle(juniorVault.requestRedeem(shares, j, j));
  }

  function _equity() internal view returns (uint256) {
    return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
  }

  function _closeShortLeaving(uint256 lots, uint256 aim) internal {
    _trade(SHORT_T, true, CfdTypes.Side.SHORT, lots, 0, 1e8 + (_equity() - aim) / lots);
  }

  function _settle(uint256 epochId) internal {
    if (vm.getBlockTimestamp() < pool.lpEpochStart(epochId)) vm.warp(pool.lpEpochStart(epochId));
    bytes[] memory data = _pay(px);
    vm.prank(KEEPER);
    r.settleLpEpoch(data);
  }

  function _pay(uint256 p) internal returns (bytes[] memory data) {
    vm.warp(vm.getBlockTimestamp() + 1);
    vm.roll(vm.getBlockNumber() + 1);
    uint256 t = vm.getBlockTimestamp();
    MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
    u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, MockPyth.MockPrice(int64(int256(p)), 0, -8, t, lastPub));
    u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, MockPyth.MockPrice(int64(int256(p)), 0, -8, t, lastPub));
    lastPub = t;
    px = p;
    data = new bytes[](1);
    data[0] = abi.encode(u);
  }

  function _mark(uint256 p) internal {
    bytes[] memory data = _pay(p);
    vm.prank(KEEPER);
    r.updateMarkPrice(data);
  }

  function _trade(address acct, bool close, CfdTypes.Side side, uint256 lots, uint256 margin, uint256 p) internal {
    OrderV3Types.OrderRequest memory q;
    q.clientOrderId = keccak256(abi.encode(acct, r.nextCommitId()));
    q.side = side;
    q.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
    q.marginDelta = margin;
    q.isClose = close;
    q.targetPrice = (side == CfdTypes.Side.LONG) != close ? 1 : CAP_PRICE;
    uint256 m = type(uint256).max; // no bound
    bytes32 h = r.lifecycleBook().currentExecutionConfigHash();
    q.bounds = OrderV3Types.ExecutionBounds(uint64(vm.getBlockTimestamp() + 60), 60, 7, h, m, m, m, m, m, m, 0, 0, type(uint32).max);
    vm.prank(acct);
    uint64 id = r.commitOrder(q);
    bytes[] memory data = _pay(p);
    vm.prank(KEEPER);
    OrderV3Types.ExecutionResult memory res = r.executeOrder(id, data);
    require(res.status == OrderV3Types.LifecycleStatus.Executed, "order not executed");
  }

  function _lp(TrancheVault v, address lp, uint256 amt) internal {
    settlementToken.mint(lp, amt);
    vm.startPrank(lp);
    settlementToken.approve(address(v), amt);
    uint256 id = v.requestDeposit(amt, lp, lp);
    vm.stopPrank();
    vm.warp(pool.lpEpochStart(id));
    pool.settleLpEpoch(0, 0); // no position yet: permissionless
    vm.prank(lp);
    v.claimDeposit(id, amt, lp, lp);
  }
}
```
