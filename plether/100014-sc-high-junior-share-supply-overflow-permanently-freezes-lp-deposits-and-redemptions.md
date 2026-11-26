# \#100014 [SC-High] Junior share-supply overflow permanently freezes LP deposits and redemptions

**Submitted on Nov 16th 2026 at 03:45:45 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100014
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Permanent freezing of funds

## Description

## Brief/Intro
While Junior NAV is small but positive, each Junior deposit mints shares at a near-zero price. An attacker repeats hourly deposits until the Junior share supply is close to `uint256.max`. The next valid deposit then overflows `_mint`, and the whole LP settlement pass reverts:
- matured Junior deposits can no longer activate;
- Senior and Junior redemptions can no longer be funded;
- matured requests cannot be cancelled.

With a 1 bps Junior maintenance fee active, fee accrual overflows as well. The owner's fee-disable finalization reverts, and an existing Senior redemption is still frozen five years later. This is **permanent freezing of funds**.

## Vulnerability Details
1. **No headroom check on the mint.** The Junior deposit quote divides by current Junior assets and does not bound the mint by the remaining supply ([TrancheVault.sol L1398-L1411](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1411)). Finalization calls `_mint` ([L1441-L1475](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1441-L1475)). When `totalSupply + shares` overflows, the entire [HousePool settlement pass](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1090) reverts, including unrelated redemption funding.
2. **The share price can be driven to near zero.** Junior absorbs first loss and pays the Senior coupon ([HousePoolWaterfallAccountingLib.sol L37-L78](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L37-L78)), so Junior assets can fall to a few USDG while old shares stay outstanding. Each attacker deposit then multiplies the supply, and the next hourly epoch again starts from tiny Junior assets.
3. **Matured requests cannot escape.** Cancellation after maturity requires a successful epoch transition ([TrancheVault.sol L826-L861](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L826-L861)).
4. **The fee path locks the owner out.** `accruedTotalSupply` adds pending fee shares to the raw supply without a headroom check ([L341-L358](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L341-L358)). Fee-disable finalization first checkpoints the old fee, which mints fee shares ([L375-L400](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L375-L400), [L417-L449](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L417-L449)), so that call overflows too.

## Impact Details
**Critical: permanent freezing of funds.**

Original-harness PoC, release 80% Senior share and 40M USDG Senior cap:
1. **The owner sets the fee.** Through the normal timelock, the owner sets a 1 bps annual Junior maintenance fee.
2. **The attacker drives Junior to near zero.** Two attacker accounts open opposed 248,400 USDG positions. After a 35 bp move, a keeper liquidates the losing SHORT. The test then replays 48 observations across 334 days of the repository's [ECB-derived Plether basket series](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/whitepaper/generated/ecb_plether_basket.csv) (29 Oct 2021 - 28 Sep 2022) as six-feed marks. Closing the LONG leaves **2.492871 USDG** of Junior assets with the old shares outstanding.
3. **The attacker amplifies the supply.** Hourly public Junior deposits follow; the final one is **19,517.957729 USDG**.

| Victim | Attack | Control |
|---|---|---|
| Senior LP redemption of 99,779,587,112,481 shares, requested before the final mint | Pending with 0 claimable at +24 h, day 120 and +5 years; cancel and refund revert | 107,334.136363 USDG claimable at +24 h |
| Honest 1,000 USDG Junior deposit | Pending and inaccessible; refund reverts at day 120 | 976.521151 USDG accessible |
| Owner fee-disable finalization | Reverts | Succeeds |
| New Senior redemption request at +5 years | Reverts | 10,833.612577 USDG claimable |

Without any fee change (release fee 0), the same historical path freezes the 1,000 USDG Junior deposit: it stays pending and inaccessible, against 953.043073 USDG accessible in the control. The attacker's extra cost over the control is 46.956928 USDG.

A flat-price construction (constant mark, Senior coupon only) produces the same freeze with a 5,525.027565 USDG final deposit.

## Recommendation
- Bound share growth: add a virtual-share offset, or a minimum assets-per-share requirement for deposit activation.
- Reject and refund an epoch whose mint would exceed the remaining supply headroom, and let the other settlement work progress.
- Apply the same headroom check to maintenance-fee accrual.
- Provide a recovery path that can disable an uncheckpointable fee and return requests that cannot activate.

## References
- [TrancheVault.sol L1398-L1475](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1475), [L341-L449](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L341-L449), [L826-L861](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L826-L861)
- [HousePool.sol L1030-L1090](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1090)
- [HousePoolWaterfallAccountingLib.sol L37-L78](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L37-L78)
- [ECB-derived basket series](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/whitepaper/generated/ecb_plether_basket.csv)

## Proof of Concept

## Proof of Concept

The full Foundry tests and their run output are in this gist: https://gist.github.com/tinnohofficial/cbda2465c873b9276200c276207eb50c
- `JuniorShareSupplyOverflow.t.sol`, `JuniorShareSupplyOverflowHistoricalCrisis.t.sol`: the tests
- `*.output.txt`: the output of each suite

Copy the two Solidity files from the gist into `packages/perps/test/poc/` in the pinned repository. Run from the repository root with the original `packages/perps/test/perps/BasePerpTest.sol`:

```sh
forge test --root packages/perps --match-path test/poc/JuniorShareSupplyOverflow.t.sol --threads 4 -vv
forge test --root packages/perps --match-path test/poc/JuniorShareSupplyOverflowHistoricalCrisis.t.sol --threads 4 -vv
```

Both commands passed twice on the original harness with identical output (`*.output.txt` in the gist). The tests call public entry points only, at release parameters, and modify no in-scope contract.

## Step by step (`test_HistoricalBasketActiveFeeFreezesJuniorAndSenior`)
1. Release pool settings: 80% Senior share and a 40M USDG Senior cap. The owner sets a legal 1-bps Junior maintenance fee through the normal timelock.
2. Two attacker accounts open opposed 248,400 USDG LONG and SHORT positions.
3. A 35 bp move makes the thin SHORT liquidatable, and a permissionless keeper liquidates it.
4. 48 observations over 334 days of the repository's ECB-derived basket series are applied as six-feed marks. The attacker closes the LONG, leaving 2.492871 USDG of Junior assets with the old shares outstanding.
5. An honest Senior LP requests a redemption, and an honest Junior LP requests a 1,000 USDG deposit.
6. The attacker makes hourly public Junior deposits. Each epoch mints shares at a near-zero price; the final deposit is 19,517.957729 USDG, which leaves the supply near `uint256.max`.
7. The next settlement reverts on the victim's mint. At +24 h, day 120 and +5 years, the Senior shares stay pending with 0 claimable. The cancel and refund calls revert, the owner's fee-disable finalization reverts, and a new Senior request reverts.
8. The control runs the same steps without the attacker's amplification deposits, and settles every request.

## Historical basket data check

The Solidity file embeds 48 selected relative marks. This command reads the repository CSV and verifies every embedded mark against its dated ECB-derived basket observation; the conversion uses integer truncation to match the test fixture.

```sh
python3 - <<'PYVERIFY'
import csv, datetime, re
from decimal import Decimal, ROUND_DOWN
from pathlib import Path
source = Path('packages/perps/test/poc/JuniorShareSupplyOverflowHistoricalCrisis.t.sol').read_text()
raw_days = source.split('uint16[48] memory daysFromPeak = [', 1)[1].split('];', 1)[0].replace('uint16', '')
raw_marks = source.split('uint32[48] memory basketPrices = [', 1)[1].split('];', 1)[0].replace('uint32', '')
days = [int(x) for x in re.findall(r'\d+', raw_days)]
marks = [int(x) for x in re.findall(r'\d+', raw_marks)]
rows = {r['date']: Decimal(r['basket']) for r in csv.DictReader(Path('packages/perps/whitepaper/generated/ecb_plether_basket.csv').open())}
start = datetime.date(2021, 10, 29)
assert len(days) == len(marks) == 48
for day, mark in zip(days, marks):
    observation = rows[str(start + datetime.timedelta(days=day))]
    expected = int((Decimal(99_650_000) * observation / rows[str(start)]).quantize(Decimal('1'), rounding=ROUND_DOWN))
    assert mark == expected, (day, mark, expected)
print('Verified 48 dated basket marks')
PYVERIFY
```

## Output: active fee and existing Senior redemption

From both historical-crisis logs:

```text
[PASS] test_HistoricalBasketActiveFeeFreezesJuniorAndSenior()
ECB replay Junior NAV after close: 2492871
ECB replay Senior impairment: 0
ECB replay Junior gate: 1
crisis attack final deposit: 19517957729
crisis control victim pending: 0
crisis attack victim pending: 1000000000
crisis control victim accessible: 976521151
crisis attack victim accessible: 0
crisis control Senior pending shares: 0
crisis attack Senior pending shares: 99779587112481
crisis control Senior claimable assets: 107334136363
crisis attack Senior claimable assets: 0
crisis attack Senior cancel at 24h succeeds: 0
crisis attack Senior refund at 24h succeeds: 0
crisis attack Senior pending at day120: 99779587112481
crisis attack Senior cancel at day120 succeeds: 0
crisis attack Senior refund at day120 succeeds: 0
crisis attack Senior pending at five years: 99779587112481
fee crisis control day120 victim pending: 0
fee crisis attack day120 victim pending: 1000000000
fee crisis attack day120 refund succeeds: 0
fee crisis control owner fee-disable succeeds: 1
fee crisis attack owner fee-disable succeeds: 0
fee crisis control five-year Senior request succeeds: 1
fee crisis attack five-year Senior request succeeds: 0
fee crisis control five-year Senior claimable: 10833612577
fee crisis attack five-year Senior claimable: 0
```

USDG values have six decimal places: for example, `107334136363` atoms are 107,334.136363 USDG.

## Output: no-fee and flat-price controls

```text
[PASS] test_HistoricalBasketNoFeeFreezesJuniorDeposit()
ECB replay Junior NAV after close: 2494517
crisis control victim pending: 0
crisis attack victim pending: 1000000000
crisis control victim accessible: 953043073
crisis attack victim accessible: 0
crisis attack final deposit: 19597448294
crisis attacker net attack minus control USDG atoms: -46956928

[PASS] test_JuniorDepositRemainsPendingAfterSupplyOverflow()
control victim pending USDG atoms after 24h: 0
attack victim pending USDG atoms after 24h: 1000000000
attack victim claimable shares after 24h: 0
attack final Junior deposit USDG atoms: 5525027565
```

The no-fee historical attacker net difference of `-46956928` atoms is **−46.956928 USDG** after subtracting its incremental final funding. The flat-price test uses coupon erosion with no trader or price change during the attack/control branch.
