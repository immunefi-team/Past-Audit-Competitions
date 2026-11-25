# \#99825 [SC-Medium] Incorrect price base in frozen-oracle liquidation (capped Engine mark shifted instead of the stored uncapped basket) leads to griefing: any keeper can liquidate solvent SHORTs at the cap once the weekly freeze starts.

**Submitted on Nov 15th 2026 at 12:29:02 UTC by @maptool for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99825
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - A user position could get liquidated while still solvent

## Description

## Brief/Intro
Liquidation pricing is one rule implemented in two branches. The live branch, after the Cecuro MED-10 fix, prices a SHORT as `min(rawBasket - shift, CAP_PRICE)`, where `rawBasket` is the uncapped accepted basket the Oracle keeps in its own storage and recomputes from the stored feeds even for an empty payload; the frozen branch applies the same shift to `engine.lastMarkPrice()`, the Engine's capped copy of the same observation. With the basket at or above the cap, live gives `CAP_PRICE` and frozen gives `CAP_PRICE - shift`, so a SHORT that `executeLiquidation(trader, [])` finds solvent at Friday 21:59:59 is liquidated by the identical call at 22:00:00 with no new observation, identical confidence and the same margin regime. In production any permissionless caller is paid the keeper share of the liquidation charge out of the solvent trader's margin, the trader's armed stop-loss (non-triggerable while frozen) is forfeited, and the sweep repeats every weekly freeze for every near-cap SHORT; the regime requires the basket to sit at the cap (a roughly 100% rise against USD), so it is an end-of-life condition in which the flip is deterministic.

## Vulnerability Details
Both liquidation entrypoints shift a `rawPrice` and then clamp it (`PletherOracle.sol:308-345`). The helper that
supplies `rawPrice` is documented as pre-cap:

```solidity
// PletherOracle.sol:531-543
/// @dev Retains the pre-cap price for adverse liquidation shifts while keeping the neutral snapshot capped.
function _updateLiquidationSnapshot(bytes[] calldata pythUpdateData) internal returns (..., uint256 rawPrice, ...) {
    BasketPrice memory basket = _resolveCurrentBasket(pythUpdateData, PriceMode.Liquidation, policy);
    rawPrice = basket.price;   // pre-cap in the live branch only
}
```

In the frozen branch `_resolveCurrentBasket` returns `_frozenBasket`, whose price is the Engine mark:

```solidity
// PletherOracle.sol:729-743
basket.price = engine.lastMarkPrice();   // the Engine only ever receives _clampToCap(basket.price), :887-888
// Liquidations receive a deterministic, conservative confidence shift while Pyth is closed.
basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
```

The uncapped value is not lost; it is in the same contract. `_acceptedBasket` is written with the pre-cap basket
(`:798`) and clamped only on read (`getAcceptedBasketMark`, `:479`), and the live empty-payload path
`_acceptedLiveBasket` recomputes the raw basket from the stored feeds (`:706-727`). The frozen branch simply reads the capped copy instead. At release settings (`basketMaxConfidenceRatioBps = 10`, `adverseConfidenceMultiplierBps = 2500`):

```text
rawBasket  = 2.00100000           confidence = 2.00 * 10 bps = 0.00200000      shift = 0.0005
live   SHORT liquidation price = min(2.001 - 0.0005, 2.00) = 2.00000000
frozen SHORT liquidation price = 2.00 - 0.0005               = 1.99950000
```

### Why this is not the documented conservative frozen policy

- **The documented conservatism is the confidence width, not the price base.** `_frozenBasket` comments that  liquidations receive a "conservative confidence shift" (`:741`); `ORACLE_SYNCHRONIZATION.md:70-76` and  `WHITEPAPER.md:1111` say the same. The PoC holds the width identical on both sides of the boundary, so the entire  difference is the base. No document says the frozen base is the capped mark, or that frozen liquidation is stricter  than live liquidation for the same observation.
- **MED-10 defines the liquidation rule with no frozen exception.** The disposition says liquidation applies the
  adjustment "to the raw, uncapped basket, then clamp[s]" and that this "removes the artificial below-cap SHORT liquidation price caused by subtracting confidence from an already capped mark". The frozen branch produces exactly that price, through a helper documented as retaining the pre-cap value.
- **There is no information loss to be conservative about.** The Oracle holds the uncapped basket for the very tick the Engine mark was written from and uses it one second earlier: the PoC's live empty-payload call finds the account solvent, and `getAcceptedBasketMark()` reports the same publish time as `engine.lastMarkTime()` before and after the boundary.
- **The fix's own regression never covered it.** The MED-10 test (`PletherOracle.t.sol:690-735`) has a frozen branch but pins the stored mark at 1.00, where capped and raw coincide.

### Preconditions

1. The accepted basket is at or above `CAP_PRICE` by at least the frozen shift (0.0005 at release settings).
2. A SHORT whose equity clears the FAD maintenance requirement at `CAP_PRICE` but not at `CAP_PRICE - shift` (a 5 USDG band for a 10,000-token position, entered by ordinary carry accrual or price drift).
3. An oracle-frozen window begins while the stored mark is within `fadMaxStaleness`.
4. Any address calls `executeLiquidation(account, [])` or `executeLiquidationBatch(accounts, [])`.

No privileged role, storage write or malformed payload is involved. Below the cap the stored mark equals the raw basket and both branches agree (control test).

## Impact Details
For a public 10,000-token SHORT (20,000 USDG notional) at release settings, the frozen liquidation that the live rule rejects produces:

| Effect | Amount | Recipient |
| --- | ---: | --- |
| Erroneous SHORT price loss (`lots * shift`) | 5.000000 USDG | HousePool |
| Liquidation charge (10 bps of notional) | 19.995000 USDG | 50% caller, 50% HousePool |
| Forfeited stop-loss trigger bounty | 0.200000 USDG | protocol treasury |
| Trader total loss | 25.195000 USDG | |
| Position | deleted at its maximum-profit price, stop-loss destroyed | |

**Primary impact: griefing.** The caller earns only the ordinary keeper share, so the attack has no meaningful profit motive: one empty-payload transaction, zero Pyth fee, deterministic harm to a solvent user. The value scales with position size and with every near-cap SHORT one batch call sweeps, and recurs each weekly freeze.

**Conditional impacts.**

- *Direct loss of user funds to the caller.* The 9.9975 USDG keeper credit is paid out of the solvent trader's margin through a liquidation that the protocol's own live rule rejects one second earlier, so by the letter of the Immunefi classification the mechanism is also a transfer of user funds to the attacker. It is listed here, not claimed as the severity, because the amount is the standard keeper share and cannot be amplified by the attacker.
- *Forced termination of a solvent position with no timely or rational remedy.* The trader receives no signal before the flip: the position is solvent under the only documented rule, and nothing discloses that eligibility changes at the calendar second. The flip and the liquidation can land in the same first frozen block, as the PoC does at 22:00:00. The armed stop-loss cannot trigger while frozen (`ConditionalTriggerFrozen`) and is forfeited instead of honored. The two tools that remain are not usable: adding margin (`CfdEngine.addMargin`, no frozen gate) requires anticipating an undocumented rule and out-racing keepers at the boundary, and a voluntary frozen close pays the fixed 0.50% spread (`frozenCloseSpreadBps = 50`), 100 USDG on this position, four times the loss the liquidation inflicts.
- *LP and treasury accounting on a mispriced event.* HousePool books 14.9975 USDG and the treasury 0.20 USDG from a liquidation that should not have occurred; this is gain rather than loss, so it is noted for completeness only.

## References
- [Frozen basket built from the capped Engine mark](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743)
- [Helper documented as retaining the pre-cap price](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L531-L543)
- [Liquidation shift-then-clamp on that value](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L308-L345)
- [Uncapped accepted basket stored (`:798`) and clamped only on read (`:479`)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L471-L481), (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L786-L799)
- [Live empty-payload replay recomputing the raw basket](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L706-L727)
- [MED-10 regression whose frozen branch pins the mark at 1.00](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L690-L735)
- [Frozen operation description](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L70-L76)
- [Cecuro AI audit, MED-10 disposition](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Link to Proof of Concept

https://gist.github.com/mrKaplan-cg/c67c91bfa2c21010d931f031872a0cc8

## Proof of Concept

## Proof of Concept
Paste the file provided in the gist at `packages/perps/test/audit/Lead5FrozenCapLiquidationPoC.t.sol` and run:

```bash
cd packages/perps
forge test --match-path test/audit/Lead5FrozenCapLiquidationPoC.t.sol -vv
```

Observed:

```text
[PASS] test_SolventShortAtCapIsLiquidatedAtFreezeBoundary()
  liquidation price (8dp): 199950000
  erroneous SHORT price loss (6dp): 5000000
  liquidation charge (6dp): 19995000
  keeper credit (6dp): 9997500
  forfeited protection bounty (6dp): 200000
  trader total loss (6dp): 25195000
  HousePool gain (6dp): 14997500
[PASS] test_BatchSweepsMultipleSolventShortsAtFreezeBoundary()
  keeper credit for two accounts (6dp): 19995000
[PASS] test_Control_BelowCapLiveAndFrozenAgree()
Suite result: ok. 3 passed; 0 failed
```

The test extends the repository's `BasePerpTest` with release risk parameters (0.1% maintenance, 0.2% initial, 3% FAD, `adverseConfidenceMultiplierBps = 2500`, `basketMaxConfidenceRatioBps = 10`, `CAP_PRICE = 2e8`). Only Pyth and the settlement token are mocked; every action goes through public production entrypoints.

### Test 1: `test_SolventShortAtCapIsLiquidatedAtFreezeBoundary`

1. Friday 21:29:50 UTC. A live `updateMarkPrice` installs a raw basket of 2.001. The Engine stores 2.000; asserted.
2. The trader opens a 10,000-token SHORT with 633 USDG margin at the cap through `commitOrder` / `executeOrder`.
3. Friday 21:59:59 UTC, FAD margin active. A final live refresh installs the same raw basket with confidence 0.002, the exact value the frozen branch will synthesize, so only the price base can differ. The trader arms a stop-loss at 1.90; its trigger bounty reservation is asserted non-zero.
4. A keeper calls `executeLiquidation` with the signed payload, then with an empty payload (replay of the Oracle's stored accepted basket). Both revert `CfdEngine__PositionIsSolvent`: the live rule finds the account solvent from stored data alone. `getAcceptedBasketMark()` is asserted to carry the same publish time as `engine.lastMarkTime()`.
5. Friday 22:00:00 UTC. `isOracleFrozen` is true, `isFadWindow` is still true. Position size, margin, settlement
   balance, pledge, liquidation reserve, claim, stored mark price and mark time are asserted identical to step 4, and the accepted basket still carries the same publish time. A stop-loss trigger attempt reverts
   `OrderRouter__ConditionalTriggerFrozen`.
6. The keeper calls `executeLiquidation(trader, [])`. The `PositionLiquidated` event price is asserted to be
   1.9995 (`CAP - shift`) and the position size to be zero.
7. Accounting is asserted: price loss 5 USDG, keeper credit equal to half the 19.995 USDG charge and to the event bounty, treasury credited the forfeited 0.20 USDG reservation (state `Settled`), protection status `Liquidated`, trader total loss 25.195 USDG, HousePool gain 14.9975 USDG.

### Test 2: `test_BatchSweepsMultipleSolventShortsAtFreezeBoundary`

1. Two SHORTs are opened at the cap and their pledges normalized to 604 USDG through the public `addMargin` path.
2. At 21:59:59 a live `executeLiquidationBatch` with the signed payload leaves both positions intact; asserted.
3. At 22:00:00 one empty-payload `executeLiquidationBatch` deletes both and credits the caller both keeper shares (19.995 USDG); asserted.

### Test 3: `test_Control_BelowCapLiveAndFrozenAgree`

1. Same setup with the raw basket at 1.999. The stored mark is asserted equal to the raw basket.
2. The live liquidation attempt at 21:59:59 and the frozen replay at 22:00:00 are asserted to return the same
   eligibility. The defect is isolated to the cap.

