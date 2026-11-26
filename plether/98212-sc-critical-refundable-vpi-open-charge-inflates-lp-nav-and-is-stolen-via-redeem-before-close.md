# \#98212 [SC-Critical] Refundable VPI open charge inflates LP NAV and is stolen via redeem-before-close

**Submitted on Nov 6th 2026 at 00:26:47 UTC by @Martelle88622 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98212
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader opens a position that pushes the pool away from balance, the trader pays an extra charge. This charge is called Virtual Price Impact (VPI). The pool records that cash as income right away, which raises the value of liquidity provider shares.

When the same trader later closes the full position, the math gives the same amount back as a refund. The trade history ends at zero: charge paid in, same amount refunded out.

The problem is timing. Nothing is set aside for the future refund between the open and the close. A person who is both a trader and a large share holder can do the steps in this order: open, withdraw shares at the raised value, then close and collect the refund. The extra withdrawal value comes from the other share holders, who are left with less.

A test that runs the full chain shows a gain of about $151 for the attacker and a loss of about $311 for the other share holder, on a pool of about $1,000,000. The trade history ends at zero, so the whole charge was refundable.

## Vulnerability Details

Some words first, so this reads on its own:

- The pool holds cash in a token called USDG. Amounts here use 6 decimals, so `1_000_000` means $1.
- Junior shares are the risky liquidity provider shares. Their cash value (share value) is pool cash and claims, divided by shares.
- Share withdrawals run in hourly rounds. A holder first asks to withdraw, waits for the round to close, a settlement step prices the shares, and the holder then claims cash. Anyone can run the settlement step; it does not need special rights.
- VPI is a charge that depends on imbalance. Let `S` be the pool imbalance in cash terms, `D` be pool depth in cash terms, and `k` be a setting named `vpiFactor`. The cost of an imbalance is `0.5 * k * S * S / D`. Opening in a way that increases imbalance pays the difference as a charge. Closing in a way that reduces imbalance gets the difference back as a refund. The live setting is `k = 0.01`, and one side may be up to 40% of the pool.
- The test uses a flat price (no price move), so profit and loss from price moves is zero. The only moving part is the VPI charge and refund, plus small trading fees.

What is wrong, step by step:

1. Opening: `OpenAccountingLib.sol:126-135` works out the VPI charge. `CfdEngineSettlementSidecar.sol:454-463` records that cash as pool income. `HousePool.sol:426-429, 835-856` includes it in the pool total, so share value rises.
2. Share pricing: `HousePoolAccountingLib.sol:60-78` values shares from physical cash minus trader claims, plus or minus price profit and loss. There is no line that subtracts the future VPI refund. The only VPI reserve in the system covers refunds owed to traders whose history is negative, which is zero here because the history is positive.
3. Closing: `CloseAccountingLib.sol:103-118` works out the refund with a lifetime limit that allows exactly `-V` against `+V`, ending at zero. `CfdEnginePlanLib.sol:1041-1061` and `CfdEngineSettlementSidecar.sol:621-623` pay that refund from pool cash.
4. The result: between open and close, the pool treats refundable cash as if it were final profit.

## Impact Details

- A normal trader plus share holder, with no owner or keeper rights, takes cash from other share holders and exits. This matches the selected Critical impact above (direct theft of non-yield user funds), and no other impact is claimed.
- At live settings the gain per round is small: about 0.015% of pool value (about $151 on a $1,000,000 pool; about $1,500 per round at $10,000,000; about $8,000 at a $50,000,000 pool, limited by how much free cash a round can pay). The round can repeat every one to two hourly rounds, and the test needs no price move.
- What the attacker needs: enough cash to be a large Junior holder (about 40% or more of the Junior pool at largest imbalance, not a strict majority), a position that stays healthy for one to two rounds, and a withdrawal that fits the round payout limit. High required cash is a question of likelihood, not of kind: the competition rules state that high required capital alone does not lower impact, and the profit here is net positive with no loss risk beyond normal market and fee exposure.
- The lifetime limit works as written but does not stop this: it only blocks a negative history. Here the history ends at zero. The missing piece is that the positive part was treated as final pool income before the offsetting refund.
- The attacker's gross share of the raised value is about `0.59 * 800 = 472`. Open and close fees plus yearly carry cost use the rest, leaving +$151. The victim side falls by $311. The gap between the two numbers stays in the pool and treasury as fees; only the $151 / $311 pair is claimed as theft and loss. The point for severity is the end state: the attacker left the share pool entirely with more cash than put in, the victim's remaining claim is worth less, and the charge that caused it was fully refunded.

## References

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/src/HousePool.sol` (share value includes the VPI cash)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` (open records VPI as income), `:621-623` (close pays the refund)
- `packages/perps/src/libraries/OpenAccountingLib.sol:126-135`, `packages/perps/src/libraries/CloseAccountingLib.sol:103-118`, `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1061`, `packages/perps/src/CfdMath.sol:150-161`
- `packages/perps/SECURITY.md`, "VPI limitations" section ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips") — covers traders whose history goes negative. This history ends at zero; the gain comes from owning shares for the positive part between open and close. No existing test in `packages/perps/test/perps/accounting/` withdraws shares at the raised value between the two legs.
- Scan report included with the repo at `audit-comp-plether/audits/scan-plether-fi-plether-core-d4541b49.md` — its close-liquidity items are about which depth number the closing math uses. This report does not change cash around the close: the test proves at one fixed depth that the refund already equals the charge at open depth, so any close-depth fix leaves the share-value capture in place.
- Pre-audit report included with the repo at `audit-comp-plether/audits/` (HIGH severity item on Arbitrum / price-feed clock gap) — about stale price fills. The test uses one flat price throughout with zero price profit and loss, so no stale price is involved.


## Link to Proof of Concept

https://gist.github.com/a-sklyarov2/da23c8b767372b7e5d335f2e0a4c32fb

## Proof of Concept

## Proof of Concept

Runnable Foundry test at pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, no in-scope changes, mocks only for Pyth updates and the USDG token. Full file in the secret gist linked in the gist field below (filename `PocVpiLpNav.t.sol`).

To run: in a checkout at that commit, place the gist file at `packages/perps/test/perps/PocVpiLpNav.t.sol`, then from the repo root run:

```bash
forge test --root packages/perps --match-path test/perps/PocVpiLpNav.t.sol -vv
# [PASS] test_ReversibleVpiInflatesLpNav()
```

The test uses only live trading and pool steps, plus stand-ins for outside services (price updates and the USDG token). It changes no pool or trading code. It checks the live settings on chain (execution fee 4 basis points, where 1 basis point is 0.01%; charge factor 0.01; imbalance cap 40%). No setup beyond the repo checkout is needed; the file is self-contained and brings its own victim and attacker accounts.

1. Setup: one victim deposits $410,000 for Junior shares. One attacker deposits $590,000 for Junior shares (about 59% of the Junior pool) and $100,000 as trading money. Total pool is about $1,000,000.
2. The attacker opens a long position of size `400_000e18` at price `1e8`, which is 40% of the pool, the largest allowed imbalance. The measured VPI charge is about $800. Pool cash rises by about that amount. A check at fixed depth (`CfdMath.calculateVPI` with imbalance in cash terms as `size * price / 1e20`) confirms the full-close refund is exactly the mirror of the charge, so the later gain cannot be blamed on a different depth setting at close time.
3. The attacker asks to withdraw all Junior shares, waits one round, an unrelated address runs `router.settleLpEpoch`, and the attacker claims about $590,472 — more than the $590,000 deposited — while the VPI history is still at +$800 (refund not yet paid).
4. The attacker closes the full position at the same flat price. The trade history returns to exactly 0 and no VPI reserve is left.
5. The victim withdraws everything through the same steps and receives about $409,689 against a $410,000 deposit. The test reads the share value before the request and checks the paid cash matches that value within 10 units (the smallest units), so the loss is real cash paid out.

Measured (USDG units with 6 decimals):

```text
attacker profit: 151.094370e6   (test checks more than 100e6)
victim realized loss: 311.293648e6  (test checks more than 200e6)
trade history after close: 0
attacker Junior shares after: 0
```

Limits stated openly: size is small per round at the live charge factor of 0.01 (about $151 net on $1M); larger numbers need a non-live setting and are not claimed. The attacker must be a large share holder for one to two rounds with a healthy position — a feasibility condition, which per competition rules does not lower severity, and profit here is net positive.
