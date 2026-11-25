# \#99277 [SC-Critical] New Junior LPs capture the VPI rebate clawback owed to incumbents

**Submitted on Nov 12th 2026 at 20:14:46 UTC by @nomorecaffeine for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99277
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

# 2. Description

A Junior depositor who opens a skew-flattening position just before the LP epoch boundary activates at a NAV that excludes the rebate that position received, then takes a pro-rata share of the rebate back on close. In the PoC, at a flat price, the depositor nets 3,193.454457 USDG and the incumbent Junior LP loses 6,392.829790 USDG.

A skew-reducing open earns a negative VPI rebate, which leaves pool cash into the trader's nonwithdrawable `vpiRebateReserveAssets`. The close clamp guarantees it comes back:

[CloseAccountingLib.sol#L112-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118)

```solidity
        // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
        // where an attacker opens at high depth, reduces pool depth, then closes to extract a
        // net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
        // stale-price risk is priced separately through frozenSpreadAssets.
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

The NAV that prices deposit activation takes lots, entry cost and a cap of pledge plus trader claims from each account. The reserve is not an input:

[TerminalNavBookV2.sol#L430-L434](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L430-L434)

```solidity
        uint256 lots = size / SIZE_QUANTUM;
        uint256 entryCostAssetAtoms = engine.positionEntryCostAssetAtoms(account);
        IMarginClearinghouse clearinghouse = IMarginClearinghouse(engine.clearinghouse());
        uint256 candidateCapAssetAtoms =
            clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
```

NAV therefore understates the incumbents by the whole reserve, which the clamp or the liquidation clawback in `planLiquidation` must return. It becomes a transfer through `OrderRouter.settleLpEpoch`, which anyone can call and which activates pending deposits at that NAV; the 5 minute `LP_REQUEST_CUTOFF_DURATION` restricts filing requests, not trading.

With the `DEPLOYMENT.md` risk parameters:

- Alice is the only Junior LP, with 10,000,000 USDG.
- Bob opens a 4,000,000 USDG long, putting skew at the 0.4 cap and paying VPI to the pool.
- Mallory requests a 40,000,000 USDG Junior deposit.
- 20 seconds before the boundary, Mallory opens a 4,000,000 USDG short that flattens the skew:
  - a rebate of 7,992.009269 USDG leaves pool cash into her reserve.
- Mallory calls `settleLpEpoch`. Her deposit activates at the NAV net of the reserve, giving her 79.998% of Junior supply.
- Mallory closes. The clamp returns 7,992.009269 USDG to the pool, of which 6,393.478730 accrues to her shares.
- Mallory redeems everything and withdraws her margin: +3,193.454457 USDG after two 4 bps execution fees. Alice's shares are worth 6,392.829790 USDG less than in the same run without the short.

Profit needs entrant share times skew ratio above 0.08 and tops out near 4.8 bps of pool depth per cycle; self-made skew does not pay. Extended runs net +2,116.06 USDG at 9 bp oracle confidence and +15,192.81 USDG beside a 40M Senior tranche (about 0.16% of redeemed shares queue under the 80% covenant until it clears).

`SECURITY.md:871-872` excludes the reserve from NAV, and WHITEPAPER 5.1 gives an exit-side reason, "LPs cannot withdraw it as marked equity before the matching position obligation settles"; neither sits in the four `SECURITY.md` sections the program lists as known issues. On entry, `SECURITY.md:859-860` says new LPs "do not inherit old trader obligations at a higher deposit NAV", the symmetry `ACCOUNTING_SPEC.md:172-173` promises, and `ACCOUNTING_SPEC.md:737-740` accepts timing transfers for carry only.

**Impact.** Incumbent Junior LPs permanently lose the entrant's pro-rata share of every outstanding VPI rebate, funded by the VPI the skew creator paid to LPs. The loss survives redemption as a fall in the incumbents' `convertToAssets`. Attackers profit from it when organic skew is near the cap; ordinary deposits take it with no attacker.

## Caller authorization

No role is used: Mallory commits and executes her own orders and settles the epoch through the permissionless `OrderRouter.settleLpEpoch`; the program states "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections".

## Recommendation

Price Junior deposit activation with the outstanding negative-VPI reserve added back, since the clamp and the liquidation clawback guarantee its return. Redemption pricing keeps excluding it, which preserves the withdrawal protection WHITEPAPER 5.1 describes. A minimal patch tracks the reserve total beside the four writes to `vpiRebateReserveBalances` (plus a getter on `IMarginClearinghouse`) and adds it in `HousePool._settleLpEpoch`:

```diff
 // MarginClearinghouse.sol
+    uint256 public totalVpiRebateReserveAssets;
 ...
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
 ...
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;

 // HousePool.sol, _settleLpEpoch
-        juniorDepositPhase.pricingAssets = juniorPrincipal;
+        juniorDepositPhase.pricingAssets =
+            juniorPrincipal + IMarginClearinghouse(ENGINE.clearinghouse()).totalVpiRebateReserveAssets();
```

With it, Mallory ends at -3,200.02 USDG (fees only), Alice is unharmed, and the 324 sponsor tests under `spec/lp`, the VPI suites and `MarginClearinghouse.t.sol` pass.

## Link to Proof of Concept

https://gist.github.com/nguyen-xuan-quoc/4e7f37faa12253b83ec624a836b58b44

## Proof of Concept

# 3. Proof of Concept

The PoC runs in the repository's Foundry test suite at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, without modifying any contract in scope.

https://gist.github.com/nguyen-xuan-quoc/4e7f37faa12253b83ec624a836b58b44

1. Copy `01-VpiReserveEntryNav.t.sol` from the Gist to `packages/perps/test/perps/poc/VpiReserveEntryNav.t.sol`.
2. From `packages/perps`, run:

```sh
forge test --match-path test/perps/poc/VpiReserveEntryNav.t.sol -vv
```

USDG and Pyth are the suite's mocks, the price stays at 1.0 with zero confidence, and orders go through the suite's `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` and adds one scalar `commitOrder` wrapper. The full log is `02-run-output.txt` in the Gist.

Controls: the same short opened after activation (`test_control_sameShortAfterActivation`) leaves Mallory at -3,200.024952 USDG and Alice unharmed (+0.010341), and in `test_passive_newLpCapturesReserveFromIncumbent` an ordinary 10,000,000 USDG deposit activating beside an honest trader's rebate realizes +3,996.548540 USDG.

**Expected:** a deposit activates at a price that includes the outstanding reserve, and the incumbent keeps the whole clawback.

**Actual:** all three tests pass; the attack test logs:

```
Ran 3 tests for test/perps/poc/VpiReserveEntryNav.t.sol:VpiReserveEntryNav
[PASS] test_attack_rebateCapturedAtEntry() (gas: 12327930)
Logs:
  ATTACK: short opened before the boundary, closed after activation
    rebate reserve held by mallory's short (USDG atoms) 7992009269
    mallory share of Junior supply after activation (1e18 = 100%) 799983898359509053
    mallory realized USDG out minus in (atoms)
  3193454457
    alice Junior value, deposit only / this run 10007997587780 10001604757990
    alice change versus deposit only (atoms)
  -6392829790
    mallory share of the reserve (atoms) 6393478730
    alice loss versus deposit only (atoms) 6392829790
```
