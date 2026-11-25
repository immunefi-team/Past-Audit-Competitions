# \#99069 [SC-High] Unbounded Junior share supply overflows deposit activation, so LP epoch settlement reverts and queued LP deposits and redemptions are frozen

**Submitted on Nov 11th 2026 at 18:22:58 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99069
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
Junior share supply has no upper bound, and deposit activation is not overflow-safe. When Junior principal is near zero, each small Junior deposit multiplies the share supply, and the Senior coupon then drains the principal back, so the next small deposit multiplies it again. After about 33 hourly deposits (294.17 USDG in the PoC, at 1,000,000 USDG of Senior) supply is 0.95 × 2^256. From then on, a Junior deposit larger than the remaining mint room (about 5% of Junior principal at that supply) that matures while Junior principal is above zero makes the share quote or the mint overflow inside `HousePool._settleLpEpoch`. While that deposit is queued the live settlement reverts with `Panic(0x11)` (at every weekday keeper hour in the PoC): the depositor's USDG stays in escrow, neither activated nor cancellable, and a Senior redemption funded earlier in the same call is rolled back.

The preconditions are heavy. The starting state is a Junior tranche wiped to near zero with Senior whole; at the release limits that needs a move of at least 50% against LPs with the book at the skew cap, and the PoC's fixture uses 95%. The supply growth then needs about 32 hours with no net open interest on a moving mark and trading revenue well below the Senior coupon. In the PoC nobody loses principal and the actor can withdraw nothing. How long the freeze lasts depends on the market afterwards: 72 hours in the PoC's idle-book branch, more than 7 days in its balanced-book branch (trading revenue 3.06 times the Senior coupon; 1,028 hours when that market is continued in a separate run), and usually 1 to 11 hours on a net-long book in our other runs.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

A matured Junior deposit lot is activated at net asset value whenever Junior principal is above zero (`HousePool.sol:914-920`; only principal exactly zero defers it). The shares are computed by:

```solidity
// TrancheVault.sol:1407-1409, quoteDepositFromState
uint256 netAssets = feeBps == 0 ? assets : Math.mulDiv(assets, BPS - feeBps, BPS, Math.Rounding.Floor);
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

and minted by `finalizeDepositEpochFromPool` (`TrancheVault.sol:1474`, `_mint(address(this), shares)`).

Supply growth. With Junior principal `J` and supply `S`, a deposit of `X` mints about `X · S / (J + 1)` shares, so supply is multiplied by about `X / (J + 1)` when `J` is far below `X`. The Senior coupon is paid from Junior principal, so within the hour `J` falls back toward zero while `S` stays. Repeating a deposit of a few USDG each hour multiplies `S` each time. Nothing bounds `S`; the only statement about it in the code is the NatSpec of `HousePoolRedemptionMathLib.sol:9-11` ("Canonical principal and supply are bounded protocol accounting values"), and no check enforces a bound. The actor does not need to settle anything: in the PoC an honest keeper settles every cycle 7 to 67 seconds after the hour, and the actor only calls `TrancheVault.requestDeposit`.

The overflow. Once `S` is near 2^256, a matured lot of `X` needs `X · S / (J + 1) + S` to fit in 256 bits, i.e. `J` at least `J*` (18,999.58 USDG for a 1,000 USDG lot at the PoC's `S`, 0.949999 × 2^256, which the output prints truncated as 0.9499). Below that:

- while `J` is small the `mulDiv` at `:1409` overflows;
- once `J` is about 950 USDG or more (`X · S / 2^256`; first seen in the PoC at `J` = 980.80, hour 34) the quote fits and `_mint` at `:1474` overflows (OpenZeppelin `_totalSupply += value`).

Both raise `Panic(0x11)`, which nothing in `_settleLpEpoch` catches. Every other reason for not activating a deposit is handled without reverting: a paused pool, a closed entry gate and a wiped Junior tranche return after the redemption phase with `entriesDeferred` (`HousePool.sol:903-920`, `:938-939`), and a lot whose quote is zero is rejected and made refundable (`HousePool.sol:1050-1068`). The overflow is the one case that reverts the whole settlement, including the redemption funding that ran before it.

The depositor cannot leave: a matured lot cannot be cancelled (`cancelPendingDeposit` reverts `DepositEpochAlreadyActive`; this is the normal rule for any matured lot, in the control too). The depositor is not warned either: `maxRequestDeposit` returns 2^256 − 1 while `estimateDepositShares(1,000 USDG)` already reverts with `Panic(0x11)`.

## Impact Details
All figures are from the PoC (Senior 1,000,000 USDG; three branches from one snapshot; USDG).

| | Control (actor does nothing) | Attack, idle book | Attack, balanced book |
|---|---|---|---|
| Actor's deposits / spent | 0 / 0 | 33 / 294.17 | 33 / 294.17 |
| Junior share supply | 5.0 × 10^14 | 0.9499 × 2^256 | 0.9499 × 2^256 |
| Keeper's settlement, weekday hours | succeeds (first hour; the control ends there) | `Panic(0x11)` in 71 of 71 | `Panic(0x11)` in 120 of 120 |
| Honest 1,000 USDG Junior deposit | activated at the next hour | neither activated nor cancellable for 72 h, then refundable | still locked after 168 h |
| Honest 33,333 USDG Senior redemption | funded at the next hour, 33,333.00 | funded after 71 h, 33,249.83 | funded after 71 h, 33,249.83 |

"Idle book": a balanced book is opened once and nobody trades again. "Balanced book": the same book plus round trips giving Junior revenue 3.06 times the Senior coupon. In both, the Senior redemption is funded only by the Friday oracle-frozen settlement, which defers deposit activation and therefore does not reach the overflow; it pays 83.17 less than in the control (the 25 bps frozen-settlement fee, net of 0.16 of coupon the actor's deposits added).

What is frozen: the depositor's USDG in vault escrow, and any LP redemption queued while that lot is at the head of the deposit queue. What is not, in the PoC: no principal is lost (the lot is refunded, or still in escrow after 168 hours) and the actor withdraws nothing (maximum Junior withdrawal 0). Trading is unaffected: in a separate run, trader-claim settlement, liquidation, open, increase, reduce, close and margin withdrawal all succeeded both with and without the inflated supply (not in the gist).

Two things are not claimed as this bug's impact.

- Ownership. The actor's lots carry almost all Junior shares, so whatever later accrues to Junior is theirs: 2,524.51 USDG after 7 days in the balanced branch, not withdrawable while Junior is below a quarter of Senior. In separate runs it became withdrawable in 1 of 16 exit probes, after four weeks with the honest book closed and a 99% Senior exit (net +10,410.44; the same shares without the freeze: +12,448.82), and after a deficit, an owner recapitalization and a price reversal (293,620.60 received for the 294.17 spent; without the actor the honest 1,000 USDG lot held 98.62% of Junior and received 160,976.54 in the same exit). That is ownership of a near-wiped tranche bought at dust, the subject of the public H-02 test cited below. The freeze lowered it in the first run; in the second it had kept the honest lot from activating beside those shares.
- Delay. In one separate run a loss larger than Junior arrived while a 100,000 USDG Senior exit was held back: the 2,611.05 that the control had paid out before the loss was impaired with the tranche, like the unpaid rest in both.

The supply does not come down. `TrancheVault` burns shares only when a redemption is funded (`:1498`) and for claim-escrow dust (`:1143`); the holder of the inflated shares cannot redeem while Junior is below a quarter of Senior (when it did redeem, in the 99% Senior-exit run above, supply fell to 0.25 × 2^256 and the lot activated) and no owner function reduces supply (code reading; not in the gist). The tranche therefore keeps accepting deposit requests while any lot above roughly `(1 − f) / f × J` (with `f` = supply / 2^256) freezes settlement again.

Limits of the result, measured:

- With no position and no trade at all after the actor's deposits there is no freeze: the coupon drains `J` to zero before the lot matures and it is refundable at the first hour (asserted in the PoC). The freeze needs Junior principal above zero at maturity; opening a book is enough.
- A Senior holder has an exit of about 16 hours. Once more than 16 matured Senior redemption epochs are queued and the redemption budget is not exhausted, the redemption phase reaches its per-call cap (`MAX_LP_EPOCHS_PER_PHASE`, `HousePool.sol:180`, `:869-876`; the listed Cecuro MED-24 concerns this cap) and the settlement returns before the deposit phase. With an honest 10 USDG Senior exit filed every hour in the balanced market, the weekday settlement succeeded in 28 of 482 keeper hours (every 16 hours), each success funding 16 epochs of queued Senior exits; the Junior lot stayed locked (separate run; not in the gist).
- A loss that takes Junior principal to zero ends the freeze: activation is then deferred and the lot is refundable (separate runs, after the owner had recapitalized and cleared degraded mode; not in the gist).
- Senior coupon. Against the control above, Senior was 632.41 behind after one week in the idle market (the control pays the coupon out of the activated honest lot's 1,000 USDG; the attack refunds that lot) and ahead in the balanced market (202.23 after one week, 1,328.16 after four). Against a control in which the actor holds the same shares and no honest lot is queued, Senior was level or ahead, or a one-off 141 behind with an honest Senior exit filed every hour (not in the gist).
- On a net-long book with a moving mark, a Junior wipe usually reopens the cancel window within hours (first window after 1 to 11 hours in 13 of 15 runs; not in the gist).
- Owner `pause()` followed by a settlement funds the Senior redemption at once in both branches. In the idle branch it also frees the lot an hour later (not at 5 and 9 bps on six feeds, below); in the balanced branch the lot stays locked and the `Panic` returns after `unpause()` (asserted in the PoC).
- Six feeds, four inverted, on a Pyth double (separate runs, previous PoC version; not in the gist): the test passes unchanged at 0 and 1 bps confidence per feed. At 5 and 9 bps (10 is refused) it passes only with the pause and 71-hour assertions relaxed: the supply loop (33 and 34 hours, 298.10 and 300.87 spent), the `Panic(0x11)` in every weekday keeper hour, the Friday-only Senior funding (after 70 and 69 hours) and the lot's lock are the same.
- An owner donation of at least `J*` through `accountExcess` activates the lot (run with 19,099.58 USDG for the 1,000 USDG lot; not in the gist). The donation becomes Junior principal, almost all of it held by the actor's shares, and supply stays near 2^256.

Preconditions:

- Junior principal near zero with Senior whole and Junior shares outstanding. The fixture reaches it with a 95% move against a long of 37.8% of pool depth and a recovery mark chosen by search; this state is given, not produced by the actor. From a Junior principal of exactly zero the block on new requests is left by one minimum round-trip trade of the actor's own (0.81 USDG in a separate run at Senior 30,000; not in the gist).
- During the 32 hours of deposits: no net open interest exposed to a moving mark, and trading revenue well below the Senior coupon. One minimum net position (about 1,000 USDG) stopped the PoC's loop at every pool size we ran, 30,000 to 40,000,000 USDG of Senior; with a ten times wider margin the loop still completed against a net position of 5,000 USDG at 30,000,000 and 40,000,000, and not at 1,000,000 (not in the gist). A keeper, or the actor, settles within the margin the actor's lot sizes leave: about 70 seconds after each hour in the PoC.
- Afterwards: a Junior deposit larger than the remaining room, maturing while Junior principal is above zero.

Closest exclusions.

- Known issues. We found no entry for a share-supply bound, an activation overflow or a deposit that reverts settlement in the listed audits, the AI scan or `SECURITY.md`. A public commit of the repository (`16d0e2be`, 2026-03-11, "Add failing tests for third audit findings (C-01, C-03, H-01, H-02, M-01, M-02)") adds a test on the same near-zero state. Its comment says that there "ERC4626 mints massive shares", and it asserts "H-02: $1 deposit must not grant majority ownership of wiped tranche". That large mint is the first step of the supply growth described here. The test is about ownership of the tranche; it does not show repetition, a missing supply bound, an overflow or a reverting settlement. The exact-zero block that followed it (`44657646`) is in place at the audited commit (`TrancheVault.sol:1585-1586`, `HousePool.sol:917`), where this PoC runs.
- Severity. Both PoC branches exceed 24 hours, so the 24-hour row is met in the PoC's conditions. We file the 1-hour row because the freeze itself gives the actor nothing and costs 294.17 to hold a 1,000 USDG deposit (the programme's griefing rule for High findings), and because the code is not yet deployed on Arbitrum One.
- Not claimed, not in the gist: at a non-zero Junior maintenance fee (0 at release), a supply much closer to 2^256 than this PoC's overflows `accruedTotalSupply()`; every LP-pricing path reverts, switching the fee off included (separate tests; not run at this PoC's supply).

## Recommendation
Bound Junior share supply, or equivalently require a minimum Junior principal for deposit activation, so that supply cannot be multiplied at dust principal. Independently, make activation overflow-safe: compute the quote and the post-mint supply with checked bounds and, when they do not fit, reject the lot, leaving it refundable, as the zero-quote branch already does, so that one lot can never revert a settlement.

## References
`HousePool.sol:903-920`, `:1046-1068`; `TrancheVault.sol:1398-1410`, `:1445-1476`; `libraries/HousePoolRedemptionMathLib.sol:9-11`; `DEPLOYMENT.md` release parameters.

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/6cd4c953041458276fe5e02787a0e329

## Proof of Concept

## Proof of Concept
One Foundry test for the repository's suite at the in-scope commit: `N3V2.t.sol` with its base `N3V2Base.sol`.

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc/n3-final-v2
cp N3V2.t.sol N3V2Base.sol packages/perps/test/poc/n3-final-v2/
cd packages/perps
forge test --root . --match-path 'test/poc/n3-final-v2/N3V2.t.sol' --gas-limit 9000000000000000000 -vv
```

What the test uses:

- No in-scope contract is modified. No storage writes, no mocked calls, no impersonation of protocol contracts.
- Every state change after the starting state is an external call to a public entrypoint from its caller's own address. The actor only calls `TrancheVault.requestDeposit`; the keeper, the LPs and the traders are separate honest addresses. The owner acts only in `setUp()` (the timelocked config installs below) and in two throw-away branches that test `pause()` / `unpause()`.
- Deployment is the repository's `BasePerpTest`. `setUp()` reads the release table from the deployed contracts and asserts every row except the Junior fee recipient, `positionSizeQuantum` (code constant `CfdTypes.SIZE_QUANTUM`) and the close execution bounty (no getter). The Senior limits (40,000,000 USDG, 8,000 bps), the Router's minimum open notional (1,000 USDG) and the oracle's `adverseConfidenceMultiplierBps` (2,500) are installed through the owner's timelocked config paths. The Senior rate (800 bps) is the contract's default. Not the release configuration: two mock feeds (release: six); a Pyth mock with zero confidence and zero fee, so the multiplier has no effect here; a zero-address Junior fee recipient (release: the protocol treasury; no effect at a fee of 0); mock USDG; the repository's Router test harness `LegacyOrderRouterHarness`, a subclass of `OrderRouter`; the test calls only inherited functions.
- `--gas-limit` only raises the test call's own gas.
- Cheatcodes: time, `prank` for the honest addresses, state snapshots for the three branches and for throw-away probes, state-diff recording to read the calls inside the reverting settlement, mock price setters, mock USDG mints to wallets.

Output, excerpt (other lines are omitted; ATTACK-QUIET is the idle-book branch; THIRD is the honest depositor, SEN the honest Senior LP, J Junior principal):

```
[PASS] test_1M_PoisonedSupply_ThreeBranches_FromOneSnapshot() (gas: 1939469982)
  SNAPSHOT Mon 12:10 UTC (GIVEN, exogenous near-wipe): stored J 0.405426, Senior 1001534.772745, Junior supply 500005500036642, no open position
    33 lots, actor USDG spent 294.174520, 32 h, last activation Tue 21:01 -> Junior supply 0.9499 x 2^256
    THIRD's view before requesting: maxRequestDeposit = 2^256-1; estimateDepositShares(1000 USDG) Panic(0x11); convertToShares(1000 USDG) Panic(0x11)
    overflow site (recorded calls): quoteDepositFromState called, finalizeDepositEpochFromPool NOT called -> TrancheVault.sol:1409 (mulDiv)
    live route OrderRouter.settleLpEpoch: Panic(0x11) in 71 of 71 weekday keeper hours; overflow site 1 (:1409) in all; J pinned at 0.000060 USDG at the last locked hour
    still locked at +7 days, J = 2524.509338 of J* = 18999.582768; Panic(0x11) in 120 of 120 weekday hours (site 1 in 34, site 2 in 86)
                                         CONTROL                       ATTACK-QUIET                  ATTACK-BALANCED
    SEN exit funded                      +0 h, 33332.999999            +71 h frozen, 33249.831596    +71 h frozen, 33249.831596
    THIRD's 1,000 USDG lot               activated +0 h                cancel opens +72 h            locked at +168 h
```

The test also asserts the rolled-back Senior funding: inside the reverting call `fundRedeemEpoch` for 33,333.164508 USDG is recorded and reverted, and the same call without the Junior lot funds that amount. Two answers printed in the output are the same in the control and are not effects of the bug: `HousePool__Unauthorized` on the direct settlement route while positions are open, and `DepositEpochAlreadyActive` on cancelling a matured lot.
