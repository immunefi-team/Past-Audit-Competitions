# \#98902 [SC-Critical] Junior LP mints cheap shares by parking a refundable VPI rebate in vpiRebateReserve (excluded from the NAV snapshot), stealing incumbent Junior principal — deposit-side, independently-fixable mirror of the reported close/redeem VPI-NAV finding (H01)

**Submitted on Nov 10th 2026 at 20:34:09 UTC by @dxhod for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98902
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The canonical LP NAV snapshot (`HousePoolAccountingLib.buildReconcileSnapshot`) does not account for the VPI rebate that a skew-reducing open pays out of the HousePool and parks in the clearinghouse `vpiRebateReserve`. While such a position is open, Junior NAV is understated by the full rebate, although closing the position returns the reserve to the pool. An attacker who is both a trader and a Junior depositor can open a skew-reducing position, settle its own matured deposit at the understated NAV, and then close the position to restore NAV. The excess shares minted to the attacker are funded by incumbent Junior LP principal; no price movement is required.

## Vulnerability Details

In `executeOpen`, a skew-reducing open pays its VPI rebate from pool cash and locks the same amount in `vpiRebateReserve`, but `buildReconcileSnapshot`, which prices both LP entry and exit, never reads that reserve. Pool physical assets fall with no offsetting asset recognized, so any Junior deposit settled while the position is open is priced against a depressed `juniorPrincipal`.

`CfdEngineSettlementSidecar.sol:442-444` (`executeOpen`, reached when `delta.tradeCostAssets < 0`; `poolRebatePayoutAssets = -tradeCostAssets` is set at `CfdEnginePlanLib.sol:333`):

```solidity
        if (delta.poolRebatePayoutAssets > 0) {
            IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
        }
```

The same amount is then locked for the account via `lockVpiRebateReserve` / `reclassifyPnlPledgeToVpiRebateReserve` (`CfdEngineSettlementSidecar.sol:466-477`).

`HousePoolAccountingLib.sol:63-65` (inside `buildReconcileSnapshot`, :60-78):

```solidity
        uint256 terminalAssets = engineSnapshot.physicalAssets;
        uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
        int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
```

Lines 66-77 combine only these three inputs into `distributable` / `deficit`. The reserve balance is not an input.

Attack sequence (an honest trader holds a standing LONG near the skew cap):

1. The attacker requests a Junior deposit with `TrancheVault.requestDeposit`; it matures at the next epoch.
2. At the activation epoch, the attacker opens an equal-size SHORT through `OrderRouter.commitOrder` / `executeOrder`. The open reduces skew, so the pool pays the rebate out (:442-444).
3. The rebate is locked in `vpiRebateReserve` (:466-477), which the NAV snapshot ignores. Junior NAV per share falls.
4. The attacker calls the permissionless `OrderRouter.settleLpEpoch`. The Junior deposit phase is priced with `pricingAssets = juniorPrincipal` (`HousePool.sol:914-920`), and `quoteDepositFromState` mints shares against it (`HousePool.sol:1046-1049`).
5. The attacker claims the shares with `claimDeposit`, then closes the SHORT. `_settleCloseActionCharge` calls `consumeVpiRebateReserve` (`CfdEngineSettlementSidecar.sol:690-698`), returning the reserve to the pool and restoring NAV.
6. The excess shares now represent value transferred from incumbent Junior LPs. On the trade leg, the lifetime VPI clamp charges the rebate back on close, so the attacker's trading cost is the two execution fees.

Relation to #98069: #98069 concerns the `tradeCostAssets > 0` branch of `executeOpen` (`CfdEngineSettlementSidecar.sol:454-463`), where a skew-increasing open's VPI charge is booked as pool revenue and captured on the redeem side. This report concerns the mutually exclusive `tradeCostAssets < 0` branch (:442-444), where the rebate payout is excluded from NAV and captured on the deposit side. The fixes are independent: removing the revenue booking does not add `vpiRebateReserve` to NAV, and adding the reserve to NAV does not remove the revenue booking.

## Impact Details

Severity: Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

PoC results at commit a82d4fe with release parameters (vpiFactor 0.01e18, maxSkewRatio 0.4, execution fee 4 bps). Setup: Senior 8,000,000 USDG, incumbent Junior 9,000,000 USDG, honest LONG of 6,600,000 notional (skew 0.388), attacker deposit 30,000,000 USDG.

- Attacker net: +4,566.38 USDG.
- Incumbent Junior net: -9,918.34 USDG, measured after the SHORT is closed and NAV is restored.
- Pool NAV drop from the rebate payout: 12,800.61 USDG.
- Junior NAV per 1e6 shares: 1000 -> 999 while the SHORT is open.
- Control (same deposit, no SHORT): attacker -1 atom, incumbent -73.02 USDG (redemption rounding).
- Price constant at 1.0 in both runs. The gap between the incumbent loss and the attacker gain is mostly the two execution fees (about 5,280 USDG).

The rebate payout is linear in pool depth at the skew cap, and the extraction happens through a deposit, so it is not limited by the `freeAssets` budget that caps redemptions.

All attacker steps (`requestDeposit`, `claimDeposit`, `commitOrder`, `executeOrder`, `settleLpEpoch`) are permissionless public calls; no trusted role, owner action or parameter change is involved, and owner helpers in the PoC are used only to seed the pool.

Known issues: the behavior is not listed in the Known Limitations section of `packages/perps/SECURITY.md` (:1328). The VPI limitations (:1434-1443) describe the trader-side reserve and lifetime clamp, and the LP accounting limitations (:1380-1399) cover mark-currency NAV, deferred activation, senior funding deferral and cooldowns. Neither section states that `vpiRebateReserve` is excluded from LP NAV or that deposits can mint against the reduced NAV.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:442-444` — rebate paid out of the pool in `executeOpen`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:466-477` — rebate locked in `vpiRebateReserve`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` — `tradeCostAssets > 0` revenue branch (#98069), not reached by this path
- `packages/perps/src/CfdEngineSettlementSidecar.sol:690-698` — `consumeVpiRebateReserve` on close
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` — `buildReconcileSnapshot`
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:333` — `poolRebatePayoutAssets = -tradeCostAssets`
- `packages/perps/src/HousePool.sol:914-920` — Junior deposit phase priced at `juniorPrincipal`
- `packages/perps/src/HousePool.sol:1046-1049` — `quoteDepositFromState`
- `packages/perps/SECURITY.md:1328`, `:1380-1399`, `:1434-1443` — Known Limitations
- Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Link to Proof of Concept

https://gist.github.com/dxhod/1b993052d6ee50c5382fb0ecf1012f0c

## Proof of Concept

Foundry test on commit a82d4fe using the repository's `BasePerpTest` harness at release parameters; only USDG and Pyth are mocked, and every attacker step goes through a public entry point.

### Steps

1. Seed Senior 8,000,000 USDG and incumbent Junior 9,000,000 USDG. An honest trader opens a 6,600,000-notional LONG (skew 0.388, cap 0.4). Record the incumbent's redeemable value and Junior NAV per 1e6 shares.
2. The attacker calls `juniorVault.requestDeposit` for 30,000,000 USDG.
3. Warp to the activation epoch. The attacker opens a 6,600,000-notional SHORT via `router.commitOrder` + `executeOrder`; Junior NAV per 1e6 shares falls from 1000 to 999.
4. The attacker calls `router.settleLpEpoch{value: 1}` and `juniorVault.claimDeposit`; shares are minted at the reduced NAV.
5. The attacker closes the SHORT; `vpiRebateReserve` is consumed back into the pool and NAV is restored.
6. Measure the attacker's net (final external value minus committed) and the incumbent's net. The control repeats the run without step 3.

### Command

```sh
cd /Users/dmytrokhodov/Documents/Projects/SCA/tmp/plether-competition
export PATH="$HOME/.foundry/bin:$PATH"
forge test --root packages/perps --match-path test/sca/R8Entry.t.sol -vv \
  --out out-sca-R8Entry --cache-path cache-sca-R8Entry
```

### Result

`2 passed`: `test_R8Entry_cheapMint_dilutesIncumbentJunior` and the control `test_R8Entry_control_noShort_noTheft`.

- Exploit: attacker net +4,566.375592 USDG; incumbent net -9,918.342789 USDG; NAV drop 12,800.612919 USDG.
- Control: attacker net -1 atom; incumbent net -73.019799 USDG (redemption rounding).

### Key assertions

- `navDrop > 0`: the rebate payout reduces pool physical assets.
- `navPerShareAfterShort < navPerShareBefore`: Junior NAV per share falls while the SHORT is open, price constant at 1.0.
- `victimNet < 0`: the incumbent loses value, measured after the SHORT is closed and NAV is restored.
- `actorNet > 0`: the attacker profits from the deposit minted at the reduced NAV.
- Control: `actorNet <= 0` and `victimNet > -100e6`.

**Secret Gist with the full runnable test file (`R8Entry.t.sol`):** https://gist.github.com/dxhod/1b993052d6ee50c5382fb0ecf1012f0c
