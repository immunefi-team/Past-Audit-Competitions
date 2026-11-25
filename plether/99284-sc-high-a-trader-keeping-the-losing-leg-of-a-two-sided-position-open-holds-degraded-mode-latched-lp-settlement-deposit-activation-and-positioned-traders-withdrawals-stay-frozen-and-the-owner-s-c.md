# \#99284 [SC-High] A trader keeping the losing leg of a two-sided position open holds degraded mode latched: LP settlement, deposit activation and positioned traders' withdrawals stay frozen, and the owner's clear reverts unless the owner recapitalizes

**Submitted on Nov 12th 2026 at 21:31:40 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99284
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
`CfdEngine` latches `degradedMode` when a close leaves the pool's cash below the larger side's maximum liability (raw `E < L`); only the owner can clear it, and only once `E >= L` again. A trader with no role, holding a long and a short at the admission limit, reaches that state after an index move of under 1 % by closing the winning leg first, and then holds it by keeping the other. While the trader keeps the losing leg open, the owner's `clearDegradedMode` reverts, LP epoch settlement reverts (no redemption funding, no deposit activation) and traders with a live position cannot withdraw free balance. In the PoC, at a 10M pool, 0 of 103 hourly settlement attempts succeed in the 103 hours the leg survives at its existing margin. Before the owner's 72-hour risk timelock can act, the owner can end it only by recapitalizing the pool with 160,525.73 USDG, and settlement still does not run after that.

Stated plainly: every blocked action is on the specification's list of what degraded mode blocks, and `SECURITY.md`'s runbook names recapitalization and stricter risk settings as the owner's responses. No line of code deviates from a document. What this report shows is narrower. (a) The freeze is the latch's, not full utilisation's: with the same book, prices and times and no latch, settlement runs, deposits activate, a redemption is funded and the positioned trader withdraws. (b) Until the owner's 72-hour risk timelock elapses, the party that decides how long the mode lasts is a trader, not the owner: the owner's clear reverts while the leg is held, and no document we read says a trader can hold it. Nothing is lost; claims are delayed. The trader makes no profit and carries real cost and market exposure.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`. `E` is pool assets less trader claims, `L` the larger side's maximum profit, `B` the settlement buffer (25 bps of `L`).

The latch, called after every close (`CfdEngine.sol:1320`) and liquidation (`:1344`):

```solidity
// CfdEngine.sol:1351-1369, _enterDegradedModeIfInsolvent
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
```

The only way out:

```solidity
// CfdEngine.sol:658-667
function clearDegradedMode() external onlyOwner {
    ...
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
```

What reads the flag:

- LP settlement: `HousePool._settleLpEpoch` calls `_requireWithdrawalsLive` first (`HousePool.sol:832`), which reverts `HousePool__DegradedMode` (`:1864-1870`). Redemption funding and deposit activation both happen inside this call.
- Trader withdrawal: for an account with a position, `validateWithdraw` reverts `CfdEngine__DegradedMode` (`CfdEngineSettlementSidecar.sol:274`).

The path. Each step is the trader's own call to a public entrypoint, from two EOAs:

1. Open a long and a short of equal size until admission (`E >= L + B`) refuses the next chunk. At a pool of 9,919,999.54 USDG that is 9,895,200.00 USDG of notional per side, margin 80,470.26 + 484,642.40, and `E − L` = 24,800.28.
2. Wait for the index to move. The trader does not cause the move. The smallest move after which step 3 latches is 78.13 bps (fall) or 79.77 bps (rise) at 10M; 83.32 / 85.06 at 1M; 78.27 / 79.91 at 50M.
3. Close the winning leg. Its payout leaves the pool; `E` drops below `L` and the mode latches.
4. Keep the losing leg open. `L` is still set by that leg's maximum profit, so `E < L` holds and the owner's clear reverts `CfdEngine__StillInsolvent`. The leg is solvent, so a keeper's `executeLiquidation` reverts `PositionIsSolvent`.

What the documents say:

- `ACCOUNTING_SPEC.md:1224-1229` lists as blocked "trader withdrawals while the account has a live position" and "new LP epoch settlement, including redemption funding and deposit activation". The allowed list (`:1216-1222`) has "recapitalization" and "owner action to clear the mode after solvency is genuinely restored".
- `EMERGENCY_RESPONSE_GUIDE.md:45-47`: the mode "blocks new risk, LP entry, and all LP epoch settlement"; the owner "can call `clearDegradedMode()` once raw `E >= L`".
- `SECURITY.md:1508-1509` (runbook): "Consider stricter risk settings to accelerate cleanup", "Recapitalize or clear positions."
- `WHITEPAPER.md:36-43` concedes the entry route: the liability bound "does not fully reserve every possible sequence in which Long and Short positions close" at different prices.

So entering the mode, and each block, is documented. None of the texts we read says that a trader can hold the condition open, or what the owner must pay to leave it while the trader does.

## Impact Details
Three worlds from one snapshot, 10M pool, release configuration; the same prices, times and hourly keeper schedule. ATTACK: winner closed at a 250 bps fall, loser kept. CONTROL-A: winner left open. CONTROL-B: winner closed at 50 bps, below the threshold, so no latch; the index then goes to the same 250 bps.

| 24 hours, hourly keeper | ATTACK | CONTROL-A | CONTROL-B |
|---|---|---|---|
| `E − L` | −160,525.73 | 31,203.39 | 39,205.71 |
| Pool free assets | 0.00 | 6,443.89 | 14,471.21 |
| `settleLpEpoch` succeeds | 0 of 24 (24 `DegradedMode`) | 2 of 24 | 2 of 24 |
| Matured Junior lot, 10,000.00: activated | 0.00 | 10,000.00 | 10,000.00 |
| Senior lot, 5,000.00: claimable | 0.00 | 5,000.00 | 5,000.00 |
| Redemption of 10,000 USDG of Junior shares: received | 0.00 | 9,996.95 | 11,224.02 |
| Positioned trader, balance 29,995.74 including position margin (the log's "free balance"): `withdraw(1,000)` | reverts `CfdEngine__DegradedMode` | ok | ok |

The other 22 attempts in each control revert `HousePool__NoLpEpochProgress` (nothing left to settle). The 1M and 50M suites give the same 0 of 24 against 2 of 24.

Duration. Latched on a Monday at 14:10:33 UTC after a 250 bps move; kept leg 9,893,800.00 notional with margin 484,587.45 (4.897 %). Every 30 minutes a keeper tries to liquidate the leg and the owner tries to clear; every hour a keeper tries to settle. At the existing margin and a flat index every liquidation attempt reverts until Friday 21:29:58 UTC (+103 h); until then the owner's clear reverts at every checkpoint and settlement succeeds 0 of 103 times. What ends it on Friday is the frozen-market margin step-up: the keeper liquidates the leg, `E − L` becomes 9,985,538.75 and the clear passes with no money. With 254,211.71 USDG more margin (2.569 % of notional, locked) the leg holds all 327 checkpoints to the next Monday 10:00 UTC, also with the index a further 200 bps against it: no liquidation, the clear reverts every time, settlement 0 of 163.

Remedy. The owner's clear alone reverts `CfdEngine__StillInsolvent`. The smallest recapitalization that lets it pass is 160,525.73 USDG (1.622 % of the notional; one atom less reverts). After it and the clear, free assets are 0.00, `settleLpEpoch` reverts `HousePool__NoLpEpochProgress` and neither lot activates: the cash is queued as pending recapitalization. Adding a further 24,734.50 or 494,690.00 changes none of that. `HousePool.pause()` and `OrderRouterAdmin.pause()` release nothing; the owner's settlement hold releases the Senior lot only (5,000.00 refunded). What restores settlement is the kept leg closing. The owner's money is not lost: after the leg closes and one settlement it sits as 160,525.73 of unassigned assets, and `assignUnassignedAssets` mints the owner Junior shares worth 160,525.73; only then do the two lots activate.

Attacker cost. Sunk round trip, both legs closed together at the 250 bps level: 67,187.11 USDG (0.678 % of notional). Carry on the kept leg: 1,247.60 per day. Margin locked: 565,112.66 at the limit, 484,642.40 after the winner closes. Exposure: the kept leg is an unhedged short of 9,895,200.00 USDG, 98,952.00 per 100 bps either way. Re-latch cycle (scripted: index back to entry, then down 250 bps again): the owner pays the clear amount plus 69,909.21, the smallest extra that admits the re-open, and the trader re-opens and closes the winner. Over three cycles the owner pays 241,714.23, 239,507.43 and 239,507.43; each cycle costs the trader 7,872.76 (the re-opened long's 247,380.00 price gain excluded), an outlay ratio of 30 : 1.

Limits of the result, measured:

- Nothing is lost. The redemption stays queued and is claimable (10,272.90 USDG of shares) once the leg closes; both lots activate later.
- The lots and the redemption request cannot be cancelled in ATTACK (`DepositEpochAlreadyActive`, `SeniorDepositCommitted`, `RedeemCancellationUnavailable`). Those are the vault's documented cancellation rules, and the cancels revert in the controls too. We claim only what differs: settlement, the two activations, redemption funding from incoming cash, and the positioned trader's withdrawal.
- The positioned trader has an exit: close, then withdraw. It costs 100.48 USDG on a 10,000.00 notional in ATTACK against 3.89 in CONTROL-A; the book is one-sided after the attacker's close, and the test does not separate that from the flag.
- For deposit activation the flag cannot be separated from raw `E < L`: no close produces `E < L` without the flag.
- At a 250 bps latch the kept leg is liquidated on Friday unless margin is added.
- Not in the gist: the owner's free lever, the 72-hour-timelocked risk configuration (`CfdEngineAdmin.sol:109-139`). Separate run: proposed at the latch, finalized at +72 h with maintenance 300 bps (engine-wide), a keeper liquidates the kept leg, the clear passes with no money and settlement runs; adding 100,000 USDG of margin first defeats it (900,000 against 1,000 bps). The 103-hour and weekend figures assume unchanged risk settings.
- Capital and book. The pair is about one pool of notional per side, and the PoC's book holds nobody else apart from one 10,000 USDG third-party long. With third-party open interest filling the book the position needed is smaller but the move needed is larger; and a latch whose losing leg is underwater ends for free once anyone liquidates it and the owner clears (separate runs, not in the gist).
- The trigger move is scripted. We make no claim that it occurs, or how often. The fixture gives the kept leg 5 % margin and the other 1 % in each direction; other splits were not run.
- Oracle and token are the repository's mocks: Pyth with a two-feed, equal-weight basket and confidence 0, not the release six-feed basket.
- Duration, remedy and cost were run at 10M only.

Closest exclusions.

- By design. The main objection, as in the intro: the blocks are listed, the entry route is conceded, recapitalization is the named response. Our answer is (a) and (b) there.
- "Lack of liquidity impacts". The pool is short of `L` by 160,525.73, and a redemption's funding from free cash is capped by the reserve with or without the latch (`SECURITY.md:342`); that share is a liquidity limit and we do not claim it. Deposit activation adds cash, and a trader's free balance is not pool liquidity; both work in the controls at the same utilisation.
- "Impacts requiring basic economic and governance attacks". The capital is stated above: 565,112.66 of margin and an unhedged leg of about one pool of notional. The program says "A high capital requirement is generally not a valid reason to downgrade a report" and that feasibility limitations "do not change the impact or severity of a finding, only the amount paid".
- Known issues. `SECURITY.md:661-663`, "raw post-settlement insolvency is contained by degraded mode", and `:340` cover the latch itself. `:1384`, "share NAV can exceed immediately redeemable liquidity", covers waiting redeemers. `:1394-1395` (Known Limitations) names "an explicit settlement hold" as escrow recovery for accepted Senior reservations; measured, it releases the Senior lot and nothing else. None names a trader-held latch or the owner's payment.
- Trust. No privileged role acts in the attack. The owner, who can "recapitalize the pool, clear degraded mode" immediately and change risk settings "after a 72-hour delay", appears only as the party whose clear reverts; the program holds that a report "is not out of scope only because a privileged address is affected".
- Griefing. There is no profit motive. The program's baseline is "spending USD 1 to cause USD 10 or less in damage is Griefing". Measured: owner outlay to trader cost is 30 : 1 per re-latch cycle; for the first latch, 160,525.73 against 67,187.11 sunk. The outlay is recoverable, and the damage to LPs and traders is delay.
- Severity. We select the 24-hour freezing row on duration alone: 0 of 24 hourly settlements asserted, and 72 hours before the owner's timelocked lever can act. Against it: documented mode, no loss, no profit. The Medium "Griefing" row is the other reading. We leave the row to the program.

## Recommendation
Keep the containment and take its duration out of a trader's hands. Options:

- Let LP settlement run the steps that do not reduce raw solvency while degraded; deposit activation raises `E`.
- After a grace period, let the owner or a keeper close or reduce the position that keeps `E < L`.

We have not built or tested a patch.

## References
`CfdEngine.sol:658-667`, `:1320`, `:1344`, `:1351-1369`; `HousePool.sol:832`, `:1864-1870`; `CfdEngineSettlementSidecar.sol:274`; `ACCOUNTING_SPEC.md:1216-1229`; `EMERGENCY_RESPONSE_GUIDE.md:45-47`; `SECURITY.md:340`, `:342`, `:661-663`, `:1384`, `:1394-1395`, `:1508-1509`; `CfdEngineAdmin.sol:109-139`; `WHITEPAPER.md:36-43`.

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/b87d249b15bfebde596936dd1400708f

## Proof of Concept

## Proof of Concept
Two files for the repository's Foundry suite at the in-scope commit: `HeldLatchBase.sol` (fixture) and `HeldLatchPoC.t.sol` (tests). No in-scope contract is modified. Expected: `14 tests passed, 0 failed` in three suites (1M, 50M, 10M).

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc/held-latch
cp HeldLatchBase.sol HeldLatchPoC.t.sol packages/perps/test/poc/held-latch/
forge test --root packages/perps --match-path 'test/poc/held-latch/HeldLatchPoC.t.sol' --gas-limit 9000000000000000000 -vv
```

`--gas-limit` only raises the test call's own gas. Tested with forge 1.7.1.

What the test uses:

- Release configuration, asserted from the deployed contracts in `setUp`: Senior limits 40,000,000 USDG and 8,000 bps, minimum open notional 1,000 USDG, execution fee 4 bps, settlement buffer 25 bps, frozen close spread 50 bps, and the ten risk parameters of `DEPLOYMENT.md` (among them maintenance 10 bps, initial 20 bps, frozen-market 300 bps), read back from `engine.riskParams()`.
- Not release values. Pool sizes (1M / 10M / 50M; no release value); the pair at the whole admission limit; margins 5 % and 1 %; the scripted index path and the cadence of keeper and owner attempts.
- Fixtures. Pyth and USDG are the repository's mocks (two-feed basket, confidence 0). The Router is the repository's `LegacyOrderRouterHarness`, a subclass of `OrderRouter`; only inherited functions are called.
- Pranks: `prank` / `startPrank` target externally owned addresses only (keeper, the two attacker legs, a stranger, third-party depositors, traders and LPs), each sending its own call. No contract is impersonated. The test contract is the owner and calls the owner functions directly, without a prank.
- Cheatcodes (census of a full trace of the 10M suite): `prank`, `startPrank`, `stopPrank`, `warp`, `roll`, `snapshotState`, `revertToState` (branching one history into ATTACK and the controls, and rolling back every probe), `getBlockTimestamp`, `getBlockNumber`, `toString`, `getNonce`, `computeCreateAddress`. No `store`, `stdstore`, `deal`, `etch` or `mockCall`. The test also calls the mock token's `mint` (funding test wallets) and the mock Pyth's price setters.

Output, the labelled lines of the 10M suite (shortened at "..."):

```
[TRIGGER] fall (close the long): pool 9,919,999.54 USDG; pair 9,895,200.00 USDG per side, margin 80,470.26 + 484,642.40, open cost 7,923.29 (0.080 % of N)
[TRIGGER] fall (close the long): E-L at the limit 24,800.28; smallest latching move 78.13 bps = 781326 units of 1e-8 (781250 units does not latch; ...)
[TRIGGER] rise (close the short): ...; smallest latching move 79.77 bps = 797728 units of 1e-8 (797651 units does not latch; ...)
[FREEZE] ATTACK (winner closed at -250 bps, loser open, 24 h hourly keeper): settleLpEpoch 0/24 ok (24 revert DegradedMode; ...), E-L -160,525.73, free assets 0.00
[FREEZE] CONTROL-A (winner open, same prices/times): settleLpEpoch 2/24 ok (0 revert DegradedMode; other attempts: HousePool__NoLpEpochProgress), E-L 31,203.39, free assets 6,443.89
[FREEZE] CONTROL-B (winner closed at -50 bps, no latch): settleLpEpoch 2/24 ok (...), E-L 39,205.71, free assets 14,471.21
[FREEZE] Junior depositor 10,000.00 USDG matured lot: activated amount CONTROL 10,000.00 (...) vs ATTACK 0.00; ...
[FREEZE] Senior depositor 5,000.00 USDG lot: CONTROL claimable 5,000.00 vs ATTACK 0.00; ...
[FREEZE] LP redemption (10,000 USDG of Junior shares) received in 24 h: CONTROL-A 9,996.95, CONTROL-B 11,224.02 vs ATTACK 0.00
[FREEZE] positioned trader free balance 29,995.74: withdraw(1,000) CONTROL ok vs ATTACK reverts CfdEngine__DegradedMode
[DURATION] fall: latch after a 250 bps move at Mon 14:10:33 UTC; kept leg 9,893,800.00 notional, margin 484,587.45 (4.897 % N), E-L -160,525.73
[DURATION] existing margin, flat index: every keeper attempt reverts until Fri 21:29:58 UTC (+103 h, FAD); before that owner clear and settleLpEpoch revert at every checkpoint (settle 0/103 ok)
[DURATION] keep-alive margin that holds every checkpoint through the weekend and a further 200 bps: add 254,211.71 USDG = 2.569 % of N, total 7.467 % (locked, not spent)
[DURATION] keep-alive margin posted, index a further 200 bps beyond the latch: 327 checkpoints to Mon 10:00:00 UTC: no keeper liquidation, owner clear reverts every time, settleLpEpoch 0/163 ok
[REMEDY] clear alone: CfdEngine__StillInsolvent; smallest recapitalization that lets it pass: 160,525.73 USDG (1.622 % of N, 1.622 % of L; x_min-1 reverts); ...
[REMEDY] after x_min + clear: free assets 0.00, settleLpEpoch reverts HousePool__NoLpEpochProgress, Junior/Senior lots activated 0/0, pending recapitalisation 160,525.73 (the cash is queued, not free)
[COST] pair 9,895,200.00 each; margin locked 565,112.66 (5.710 % N) at the limit; sunk round trip (both closed together at -250 bps, CONTROL): 67,187.11 = 0.678 % of N
[COST] re-latch cycle 1: owner pays 241,714.23 (x_min + 69,909.21 = 282.598 % of B); attacker's cost of the cycle (...; the 250 bps price gain of 247,380.00 excluded) 7,872.76 => owner : attacker = 30 : 1
```

In this engine the long gains when the index falls (asserted).
