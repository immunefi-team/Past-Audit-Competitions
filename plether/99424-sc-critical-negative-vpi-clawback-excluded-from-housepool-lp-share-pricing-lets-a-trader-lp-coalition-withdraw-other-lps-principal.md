# \#99424 [SC-Critical] Negative-VPI Clawback Excluded from HousePool LP Share Pricing Lets a Trader-LP Coalition Withdraw Other LPs' Principal

**Submitted on Nov 13th 2026 at 16:10:28 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99424
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open receives a negative virtual price impact (VPI) rebate paid from HousePool cash. The position owes an equal clawback, backed by a dedicated nonwithdrawable VPI reserve in the MarginClearinghouse. A close or liquidation recovers the outstanding amount by withholding price gain or consuming the reserve. LP share pricing counts neither the reserve nor the clawback, so between the open and the close the equity that prices Junior shares is lower by the cash the open paid out, the full rebate in every tested configuration.

A Junior deposit activated in that window buys shares at the lowered equity, and the clawback that returns at the close is shared with those new shares. A trader and a Junior LP acting together (the coalition) can open the skew-reducing position just before an LP epoch boundary and close it just after. A passive Junior LP's fully funded withdrawal can then fall below its deposit while the coalition exits with more USDG than it funded.

## Vulnerability Details

**The open pays the rebate from HousePool cash and locks an equal VPI reserve outside the pool.** An open's trade cost is the signed sum of its VPI and execution fee (OpenAccountingLib.sol:135). When the VPI rebate exceeds the fee, the plan sets a HousePool payout of the net rebate to the trader's clearinghouse account, and the VPI reserve target becomes `max(-vpiAccrued, 0)` evaluated on the new lifetime VPI:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L331-L349

```solidity
        delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
        ...
        delta.vpiRebateReserveAfterAssets =
            _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued + openState.vpiAssets);
```

The settlement sidecar transfers that amount out of HousePool, funds the execution fee from free pool cash up to the cash available, and locks the reserve from the trader's free settlement or reclassifies it from the position's PnL pledge:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L478

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L905-L925

```solidity
        if (delta.poolRebatePayoutAssets > 0) {
            IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
        }
        ...
                clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
```

HousePool pays the net rebate and then the execution fee, capped by free pool cash (CfdEnginePlanLib.sol:737-745). When free pool cash covers the fee, as in every tested configuration, HousePool cash falls by the full rebate `r`. The trader's clearinghouse account holds the reserve, equal to `r`, in reserved settlement that it cannot withdraw while the position is open (MarginClearinghouse.sol:843-852; packages/perps/SECURITY.md:866-867).

**LP share pricing omits the reserve.** The engine snapshot that HousePool uses for LP pricing takes physical assets from `HousePool.totalAssets()` and the terminal LP price delta from `TerminalNavBookV2`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L62-L70

```solidity
        uint256 poolAssets = engineContract.pool().totalAssets();
        snapshot.physicalAssets = poolAssets;
        ...
        snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
```

The terminal book caps each account's LP receivable at its PnL pledge plus its trader claim, so the VPI reserve, which sits in the account's reserved settlement, does not enter the terminal delta:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L433-L434

```solidity
        uint256 candidateCapAssetAtoms =
            clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
```

`HousePoolAccountingLib.buildReconcileSnapshot` derives LP distributable equity from the physical pool assets, the terminal LP price delta and the trader claims only, with no term for the reserve or the clawback:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78

The protocol keeps both anticipated VPI and the VPI reserve out of terminal `K` and symmetric LP NAV (packages/perps/SECURITY.md:866-872; packages/perps/ACCOUNTING_SPEC.md:1057-1066). The exclusion keeps LPs from withdrawing the reserve as marked equity before the matching position obligation settles (packages/perps/WHITEPAPER.md:887-888). For solvency and redemption funding it is conservative: the reserved USDG is held in the trader's clearinghouse account and cannot pay LP redemptions until it is collected. For LP share pricing it is not conservative. Symmetric LP NAV prices entry and exit on the same terminal equity so that new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV (packages/perps/SECURITY.md:851-860). A marked price item carries market risk that the new LP then shares. The clawback does not move with the mark: its amount is fixed by the position's lifetime VPI, the dedicated reserve backs it, and the close and liquidation flows below collect it. Leaving it out of share pricing hands a reserve-backed receivable, paid for by the LPs present at the open, to any LP whose deposit is activated before it is collected.

**A full close or a liquidation recovers the outstanding negative lifetime VPI.** A close applies the lifetime clamp, which keeps accrued plus new VPI at or above zero, so closing a position with lifetime VPI of `-r` charges at least `r`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118

```solidity
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

The close plan collects that clawback from withheld price gain or from the reserve, and the sidecar transfers the consumed reserve to HousePool. Liquidation assesses the full `max(-vpiAccrued, 0)` the same way:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1052-L1057

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L691-L693

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1559-L1569

```solidity
            uint256 vpiClawbackAssets =
                _negativeVpiReserveTarget(snap.position.vpiAccrued) - delta.vpiRebateReserveAfterAssets;
            vpiClawbackWithheldAssets = delta.actionChargeWithheldAssets < vpiClawbackAssets
                ? delta.actionChargeWithheldAssets
                : vpiClawbackAssets;
            delta.vpiRebateReserveConsumedAssets = vpiClawbackAssets - vpiClawbackWithheldAssets;
```

**Deposits activated while the reserve is outstanding are priced on the lowered equity.** LP epoch settlement reconciles the pool from the snapshot above and then prices Junior deposits on the resulting Junior principal and share supply:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L919

```solidity
        DepositPhase memory juniorDepositPhase;
        juniorDepositPhase.pricingAssets = juniorPrincipal;
        juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

Shares minted in that settlement are priced on equity lowered by that payout (`r` in the tests). When the position closes, the clawback returns as pool value and is shared pro rata across the enlarged Junior supply, so a new Junior LP holding a fraction `alpha` of Junior supply receives about `alpha * r` that the existing Junior LPs funded. Junior redemptions settled while the reserve is outstanding are priced on the same lowered equity (HousePool.sol:835-841).

**Trigger.** A Junior LP requests a deposit for the next LP epoch before the 5-minute request cutoff. About 1 minute before the epoch boundary, a trader opens a position that reduces the open skew and receives the rebate. At the boundary any address settles the LP epoch through `OrderRouter.settleLpEpoch`, which activates the deposit at the lowered equity. Seconds later the trader closes the position, and the close recovers the clawback for HousePool. After the 1-hour deposit cooldown, the Junior LP redeems at equity that again includes the clawback.

## Impact Details

Severity: Critical, Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

The coalition takes principal from a passive Junior LP. The passive Junior LP makes no trade, its redemption is fully funded with no pending shares, and it withdraws less USDG than in the same history without the coalition trader's open and close. The coalition's Junior LP and trader withdraw their whole balances as USDG and, in the 3 profitable configurations, finish above the capital they funded. In the main configuration the passive Junior LP loses 5,880.57 USDG and withdraws 4,286.45 USDG less than its deposit.

In the release deployment and parameters, each configuration runs twice from one snapshot with the same LP deposits and redemptions, the same independent SHORT, and the same prices, timestamps and blocks; the second run adds only the coalition trader's open and close.

| Configuration | Coalition share of Junior supply | Passive Junior LP deposit | Passive withdrawal without the coalition trades | Passive withdrawal with them | Passive Junior LP loss | Coalition gain over the run without its trades |
|---|---:|---:|---:|---:|---:|---:|
| Junior 10M passive, 30M coalition, LONG 3.96M | 74.99% | 10,000,000.00 | 10,001,594.12 | 9,995,713.55 | 5,880.57 | +2,707.76 |
| Junior 10M passive, 5M coalition, LONG 1.56M | 33.32% | 10,000,000.00 | 10,001,596.32 | 9,999,943.05 | 1,653.27 | +403.78 |
| Junior 10M passive, 2M coalition, LONG 3.96M | 16.66% | 10,000,000.00 | 10,001,597.21 | 10,000,290.52 | 1,306.69 | -1,862.53 |
| Senior 7.5M, Junior 2.5M passive, 2.5M coalition, LONG 3.96M | 50.00% | 2,500,000.00 | 2,501,323.00 | 2,497,402.73 | 3,920.27 | +749.01 |

In the main configuration the rebate is 7,834.66 USDG, pending LP equity falls by exactly that amount at the open, and the close returns 7,834.76 USDG to HousePool. The coalition's Junior LP withdraws 30,005,876.07 USDG for its 30,000,000.00 USDG deposit; with the 1,000,000.00 USDG of trader funding, the coalition gets back 2,707.76 USDG more than it funded, after 3,168.00 USDG of execution fees paid to the protocol treasury. In every configuration the passive loss equals the coalition's Junior share times the rebate within 0.2%, and the USDG held by the custody contracts and actor wallets equals the token supply in both runs.

The loss does not depend on the coalition's profit. With a 2,000,000.00 USDG coalition deposit the coalition loses 1,862.53 USDG, and the passive Junior LP still loses 1,306.69 USDG.

Prerequisites:
- Open skew held by other traders that the coalition's open reduces. The tests use an independent SHORT of 3,960,000.00 USDG notional, 39.57% of the 10,007,842.80 USDG of HousePool assets, within the 40% maximum skew ratio of the release parameters (script/DeployPerpsArbitrumSepolia.s.sol:434-446). The VPI cost of a skew `S` is `0.5 * k * S^2 / D` with `k` = 1% and pool depth `D` (CfdMath.sol:134-139). A skew-reducing open of notional `s` <= `S` therefore earns `r = 0.5 * k * (S^2 - (S - s)^2) / D`, at most 8 bps of `D` at the maximum skew ratio. Without third-party skew the coalition must first create the skew with its own skew-increasing open, which pays VPI and execution fees of its own.
- A coalition share `alpha` of Junior supply after activation that exceeds its trading cost. The coalition's net is about `alpha * r - 2 * 4 bps * s` (execution fee 4 bps per leg, CfdEngine.sol:147). For an open much smaller than `S`, `r` is about `k * S * s / D`. The open is then profitable when `alpha > 2 * 4 bps * D / (k * S)`, about 20% of Junior supply at a skew of 39.57% of depth. The tests are profitable at 33.32%, 50.00% and 74.99%, and unprofitable at 16.66%.
- Capital. The smallest profitable configuration above uses a 2,500,000.00 USDG Junior deposit next to 7,500,000.00 USDG of Senior and 2,500,000.00 USDG of passive Junior capital, plus trader margin; initial margin is 0.2% of notional in the release parameters.
- Timing inside 1 LP epoch. The coalition's Junior LP requests its deposit before the 5-minute cutoff (TrancheVault.sol:39, :545-556). A pending Junior deposit stays cancellable until its epoch is active (TrancheVault.sol:851-859), so the coalition can cancel it if the open does not fill before the boundary. In the tests the open fills 53 seconds before the boundary and the close 17 seconds after it.
- LP epoch settlement at the boundary. Settlement is permissionless through `OrderRouter.settleLpEpoch` (OrderRouter.sol:479-481); while positions are open and the market is live, HousePool accepts only that call with a mark published at or after the boundary (HousePool.sol:808-819). An address with no role settles in the tests. Entry activation is deferred while the oracle is frozen (HousePool.sol:908-912, :1143; packages/perps/SECURITY.md:1387-1388), so the sequence runs during live market hours.
- The 1-hour deposit cooldown (TrancheVault.sol:38) delays only the coalition's Junior exit; the clawback is already back in HousePool by then.
- With Senior capital present, the Junior covenant can limit Junior redemption funding (packages/perps/SECURITY.md:396). In the Senior-heavy configuration the Senior LP redeems in the same LP epoch as the passive Junior LP, both redemptions are fully funded, and the Senior withdrawal is identical in both runs.
- Transaction costs. The coalition result counts the execution fees, carry and every other charge on the trader's balance. It does not include Arbitrum gas or the Pyth update fee, which the local Pyth double does not charge. Fills use zero Pyth confidence. With a nonzero confidence the oracle shifts both coalition fills against the trader (PletherOracle.sol:940); the tests do not measure that case.
- Each configuration measures 1 open and close around a single LP epoch boundary.
- After the release deployment and its owner setup steps (seed positions and trading activation), no privileged role or unusual configuration is involved, and every call in the scenario is a public function. Prices come from the repository's `MockPyth` at the release Pyth address, with all 6 feeds at a fixed 1.20 basket.

## Recommendation

Count the aggregate negative-VPI clawback of open positions, the sum of `max(-vpiAccrued, 0)`, as an LP receivable in the equity that prices LP entry and exit, for example as a positive term in the terminal LP price delta that CfdEngineProtocolLens passes to HousePool. Maintain that aggregate in CfdEngine and update it in the same call that writes or deletes a position's lifetime VPI, so every open, increase, partial close, full close and liquidation that changes the reserve target changes the receivable atomically. Keep the reserve outside withdrawable liquidity, so redemption funding still uses only collected pool cash.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Symmetric LP NAV and the negative-VPI reserve: `packages/perps/SECURITY.md:849-876`, `packages/perps/ACCOUNTING_SPEC.md:1057-1066`
- VPI curve: `packages/perps/src/CfdMath.sol:96-139`
- Release parameters: `packages/perps/DEPLOYMENT.md` (Initial release parameters) and `script/DeployPerpsArbitrumSepolia.s.sol`

## Proof of Concept

## Proof of Concept

The test deploys the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` inside Forge through `ReleaseFixture.sol` (source at the end of this section). In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address. All 6 feeds stay at a 1.20 basket with zero confidence.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

The PoC file `NegativeVpiReserveEntryPricing.t.sol` contains 4 tests. Each test runs one history twice from the same snapshot: a comparison run without the coalition trader's open and close, and the attack run with them. All other LP and trader actions, prices, timestamps and blocks are identical in both runs. Steps in `test_entrantCapturesClawbackFromPassiveJuniorLp`, all through public entry points; orders are executed and LP epochs are settled by an address with no role:
1. The passive Junior LP deposits 10,000,000 USDG and is activated. An independent trader opens a SHORT of 3,960,000 USDG notional, 39.6% of HousePool assets. The coalition trader funds 1,000,000 USDG of margin, and the coalition's Junior LP requests a 30,000,000 USDG deposit for the next LP epoch. The test takes a snapshot.
2. 60 seconds before the LP epoch boundary the coalition trader commits a LONG of the same size, which executes 7 seconds later and receives the VPI rebate. The test reads the pending LP equity from `HousePool.getPendingTrancheState` before and after the open, and the VPI reserve from `MarginClearinghouse.vpiRebateReserveAssets` after it.
3. At the boundary `OrderRouter.settleLpEpoch` runs with a price published at the boundary. The coalition's deposit is activated and claimed.
4. 10 seconds later the coalition trader commits the full close, which executes 7 seconds later. The coalition's deposit has deepened the pool, so the lifetime clamp sets the close's VPI charge to the rebate and the clawback returns to HousePool.
5. After the 1-hour cooldown the coalition's Junior LP redeems all its shares at the next LP epoch. The independent trader closes its SHORT, and 1 hour later the passive Junior LP redeems all its shares. Both traders withdraw their whole clearinghouse balance.
6. The test checks the exact rebate, a VPI reserve equal to the rebate, and a pending LP equity drop equal to the rebate (zero in the comparison run). It checks a zero reserve after the close with at least the rebate returned to HousePool, and a treasury gain equal to the 2 execution fees. It checks full funding of every LP redemption and zero pending shares for the passive Junior LP. It checks a passive loss equal to the coalition's Junior share times the rebate within 0.2%, the exact passive loss, coalition gain and passive shortfall below its deposit, and coalition withdrawals above the capital it funded. In both runs it checks that the USDG held by the custody contracts and actor wallets equals the token supply.

The other 3 tests repeat these steps with different sizes and check the passive loss within 0.2% of share times rebate and the sign of the coalition gain: `test_oneThirdJuniorShareWithSmallerOpen` (5,000,000 USDG coalition deposit, LONG of 1,560,000 USDG notional), `test_passiveJuniorLpLosesWhenCoalitionDoesNotProfit` (2,000,000 USDG coalition deposit), and `test_seniorHeavyPoolHalfJuniorShare` (7,500,000 USDG Senior LP, 2,500,000 USDG passive Junior LP, 2,500,000 USDG coalition deposit; the Senior LP redeems in the same LP epoch as the passive Junior LP).

On a fresh clone of the audit commit, run `git submodule update --init --recursive`. Create `test/poc/` at the repository root, place `ReleaseFixture.sol` and `NegativeVpiReserveEntryPricing.t.sol` (sources at the end of this section) in it, then run from the repository root with Forge 1.5.1:

```bash
forge test --match-path test/poc/NegativeVpiReserveEntryPricing.t.sol -vv
```

Excerpt from the output of the command above for `test_entrantCapturesClawbackFromPassiveJuniorLp` (deployment lines omitted):

```
[PASS] test_entrantCapturesClawbackFromPassiveJuniorLp() (gas: 15743020)
Logs:
  == passive Junior 10M, entrant Junior 30M, skew 39.6% of depth
    passive Junior LP deposit                         10000000.000000 USDG
    coalition Junior LP deposit                       30000000.000000 USDG
    coalition trader funding                          1000000.000000 USDG
    HousePool assets before the open                  10007842.798431 USDG
    independent SHORT notional                        3960000.000000 USDG
    coalition LONG notional                           3960000.000000 USDG
    negative VPI rebate on the open (vpiAccrued)      -7834.655437 USDG
    coalition VPI reserve after the open              7834.655437 USDG
    pending LP equity drop at the open                7834.655437 USDG
    coalition share of Junior supply (bps)            7499
    HousePool USDG change over the coalition close    7834.764313 USDG
    coalition VPI reserve after the close             0.000000 USDG
    passive Junior LP withdrawal, comparison          10001594.119565 USDG
    passive Junior LP withdrawal, attack              9995713.554181 USDG
    passive Junior LP loss                            5880.565384 USDG
    passive Junior LP result vs deposit, attack       -4286.445819 USDG
    coalition Junior LP withdrawal, comparison        29999999.999986 USDG
    coalition Junior LP withdrawal, attack            30005876.072171 USDG
    coalition trader withdrawal, comparison           1000000.000000 USDG
    coalition trader withdrawal, attack               996831.691124 USDG
    coalition cash gain over the comparison           2707.763309 USDG
    coalition result vs funded capital, attack        2707.763295 USDG
    treasury gain (coalition execution fees)          3168.000000 USDG
    USDG in custody contracts + wallets = supply      42000002.000000 USDG
```

Excerpt from the same output for the other 3 tests (deployment lines and other values omitted) and the suite summary:

```
[PASS] test_oneThirdJuniorShareWithSmallerOpen() (gas: 15743436)
Logs:
  == entrant Junior 5M, coalition LONG 1.56M notional
    negative VPI rebate on the open (vpiAccrued)      -4956.912393 USDG
    coalition share of Junior supply (bps)            3332
    passive Junior LP withdrawal, comparison          10001596.323830 USDG
    passive Junior LP withdrawal, attack              9999943.051806 USDG
    passive Junior LP loss                            1653.272024 USDG
    passive Junior LP result vs deposit, attack       -56.948194 USDG
    coalition cash gain over the comparison           403.778753 USDG
    coalition result vs funded capital, attack        403.778747 USDG
[PASS] test_passiveJuniorLpLosesWhenCoalitionDoesNotProfit() (gas: 15740962)
Logs:
  == entrant Junior 2M, coalition unprofitable
    negative VPI rebate on the open (vpiAccrued)      -7834.655437 USDG
    coalition share of Junior supply (bps)            1666
    passive Junior LP withdrawal, comparison          10001597.207013 USDG
    passive Junior LP withdrawal, attack              10000290.516235 USDG
    passive Junior LP loss                            1306.690778 USDG
    passive Junior LP result vs deposit, attack       290.516235 USDG
    coalition cash gain over the comparison           -1862.531327 USDG
    coalition result vs funded capital, attack        -1862.531330 USDG
[PASS] test_seniorHeavyPoolHalfJuniorShare() (gas: 17601333)
Logs:
  == Senior 7.5M, passive Junior 2.5M, entrant Junior 2.5M
    negative VPI rebate on the open (vpiAccrued)      -7834.655437 USDG
    coalition share of Junior supply (bps)            5000
    passive Junior LP withdrawal, comparison          2501322.999748 USDG
    passive Junior LP withdrawal, attack              2497402.728329 USDG
    passive Junior LP loss                            3920.271419 USDG
    passive Junior LP result vs deposit, attack       -2597.271671 USDG
    coalition cash gain over the comparison           749.005290 USDG
    coalition result vs funded capital, attack        680.617824 USDG
    Senior LP withdrawal, comparison                  7500342.470755 USDG
    Senior LP withdrawal, attack                      7500342.470755 USDG

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 49.07ms (124.39ms CPU time)
```

### Source files

#### `ReleaseFixture.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Deploys the release contracts with script/DeployPerpsArbitrumSepolia.s.sol inside Forge.
// Pyth, USDG and ArbSys are replaced by local test components; in-scope contracts are unmodified.

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Test, console2} from "forge-std/Test.sol";

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
    uint8 internal constant MODE_LIVE = 1;
    uint8 internal constant MODE_FAD = 2;
    uint8 internal constant MODE_FROZEN = 4;
    uint8 internal constant MODE_ANY = 7;

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;
    address internal owner;
    address internal keeper = makeAddr("keeper");
    uint256 internal clientNonce;

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

        // Permanent 1 USDG seeds per tranche (release values), then activation.
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

    // ---------------------------------------------------------------- prices

    function _oracle() internal view returns (PletherOracle) {
        return PletherOracle(d.pletherOracle);
    }

    /// @dev Sets all six feeds so the normalized basket equals `basketPrice`, published at the current block time.
    function _setBasketPrices(
        uint256 basketPrice
    ) internal {
        _setBasketPricesAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPricesAt(
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        for (uint256 i; i < 6; ++i) {
            _setFeed(i, basketPrice, publishTime);
        }
    }

    /// @dev Sets one component so that it alone would give normalized basket `basketPrice`, at `publishTime`.
    function _setFeed(
        uint256 i,
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
    }

    /// @dev Same as _setFeed with an explicit confidence (in feed units).
    function _setFeedWithConf(
        uint256 i,
        uint256 basketPrice,
        uint64 conf,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), conf, int32(-8), publishTime);
    }

    /// @dev A non-encoded payload makes the MockPyth double use the configured per-feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _refreshMark(
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- time

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

    // ---------------------------------------------------------------- USDG and LPs

    function _mint(
        address to,
        uint256 amount
    ) internal {
        d.settlementToken.mint(to, amount);
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
        _mint(lp, assets);
        vm.startPrank(lp);
        d.settlementToken.approve(address(v), assets);
        requestId = v.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    /// @dev Warps to the request's epoch start (if not past) and settles through the Router's atomic oracle route.
    function _settleEpoch(
        uint256 requestId,
        uint256 basketPrice
    ) internal {
        uint256 start = d.housePool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < start) {
            _warpTo(start);
        }
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.settleLpEpoch(_updateData());
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

    /// @dev Request, settle at `basketPrice`, claim. Returns shares.
    function _lpDeposit(
        bool senior,
        address lp,
        uint256 assets,
        uint256 basketPrice
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(senior, lp, assets);
        _settleEpoch(requestId, basketPrice);
        shares = _claimDeposit(senior, lp, requestId);
    }

    function _requestRedeem(
        bool senior,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
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

    // ---------------------------------------------------------------- traders

    function _fundTrader(
        address trader,
        uint256 assets
    ) internal {
        _mint(trader, assets);
        vm.startPrank(trader);
        d.settlementToken.approve(address(d.clearinghouse), assets);
        d.clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @dev Permissive bounds: no practical ceilings (zero means zero, so ceilings use max values).
    function _bounds(
        uint8 modes
    ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + 60);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = modes;
        b.expectedConfigHash = d.lifecycleBook.currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    function _nextClientId() internal returns (bytes32) {
        return bytes32(uint256(keccak256(abi.encode("poc", ++clientNonce))));
    }

    /// @param size synthetic size in 1e18 units; must be a multiple of 100e18 (positionSizeQuantum 1e20).
    function _openReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = targetPrice;
        r.isClose = false;
        r.bounds = _bounds(MODE_LIVE | MODE_FAD);
    }

    function _closeReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = 0;
        r.targetPrice = targetPrice;
        r.isClose = true;
        r.bounds = _bounds(MODE_ANY);
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory r
    ) internal returns (uint64 orderId) {
        vm.prank(trader);
        orderId = d.router.commitOrder(r);
    }

    /// @dev Advances 7 s / 2 blocks (past the commit tick), publishes `basketPrice`, executes as `keeper`.
    function _execute(
        uint64 orderId,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(orderId, _updateData());
    }

    function _openPosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        // Slippage targets: a LONG gains when the basket falls, so it opens at a price >= target and closes at <= target.
        uint256 target = side == CfdTypes.Side.LONG ? 1 : basketPrice * 2;
        uint64 id = _commit(trader, _openReq(side, size, margin, target));
        result = _execute(id, basketPrice);
    }

    function _closePosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint256 target = side == CfdTypes.Side.LONG ? basketPrice * 2 : 1;
        uint64 id = _commit(trader, _closeReq(side, size, target));
        result = _execute(id, basketPrice);
    }

    function _requireExecuted(
        OrderV3Types.ExecutionResult memory r,
        string memory label
    ) internal pure {
        if (r.status != OrderV3Types.LifecycleStatus.Executed) {
            console2.log("order not executed:", label);
            console2.log("  status", uint256(uint8(r.status)));
            console2.log("  terminalReason", uint256(uint8(r.terminalReason)));
            console2.log("  pendingReason", uint256(uint8(r.pendingReason)));
            revert(label);
        }
    }

    function _liquidate(
        address account,
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.executeLiquidation(account, _updateData());
    }

    // ---------------------------------------------------------------- state views

    function _poolAssets() internal view returns (uint256) {
        return d.housePool.totalAssets();
    }

    function _usdgOf(
        address a
    ) internal view returns (uint256) {
        return d.settlementToken.balanceOf(a);
    }

    function _logState(
        string memory label
    ) internal view {
        console2.log("==", label);
        console2.log("  pool totalAssets      ", d.housePool.totalAssets());
        console2.log("  pool raw USDG         ", _usdgOf(address(d.housePool)));
        console2.log("  seniorPrincipal       ", d.housePool.seniorPrincipal());
        console2.log("  juniorPrincipal       ", d.housePool.juniorPrincipal());
        console2.log("  clearinghouse USDG    ", _usdgOf(address(d.clearinghouse)));
        console2.log("  seniorVault USDG      ", _usdgOf(address(d.seniorVault)));
        console2.log("  juniorVault USDG      ", _usdgOf(address(d.juniorVault)));
        console2.log("  total trader claims   ", d.engine.totalTraderClaimBalanceAssets());
        console2.log("  engine mark           ", d.engine.lastMarkPrice());
        console2.log("  degradedMode          ", d.engine.degradedMode());
    }

    /// @dev Sum of USDG held by every in-scope custody contract (Clearinghouse, HousePool, both vaults, Engine).
    function _custodyUsdg() internal view returns (uint256) {
        return _usdgOf(address(d.clearinghouse)) + _usdgOf(address(d.housePool)) + _usdgOf(address(d.seniorVault))
            + _usdgOf(address(d.juniorVault)) + _usdgOf(address(d.engine));
    }

}
```

#### `NegativeVpiReserveEntryPricing.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseFixture} from "./ReleaseFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/Test.sol";

/// @notice A skew-reducing open receives a negative-VPI rebate paid from HousePool cash. The clawback the position owes
///         back is held in its MarginClearinghouse VPI reserve, which the equity used for LP entry and exit pricing
///         does not count. A Junior deposit activated while the reserve is outstanding is priced on equity lowered by
///         the full rebate; the clawback returns to HousePool cash at the close and is shared with the new shares.
///         Every test runs one history twice from the same snapshot: a comparison run without the coalition trader's
///         open and close, and the attack run with them. All other LP and trader actions, prices, timestamps and
///         blocks are identical in both runs.
contract NegativeVpiReserveEntryPricingTest is ReleaseFixture {

    address internal passiveJuniorLp = makeAddr("passiveJuniorLp");
    address internal seniorLp = makeAddr("seniorLp");
    address internal entrantJuniorLp = makeAddr("entrantJuniorLp"); // coalition Junior LP
    address internal skewTrader = makeAddr("skewTrader"); // independent SHORT trader
    address internal coalitionTrader = makeAddr("coalitionTrader");

    uint256 internal constant TRADER_FUNDS = 1_000_000e6;

    struct Config {
        uint256 passiveJunior;
        uint256 senior;
        uint256 entrant;
        uint256 skewSize; // SHORT held by the independent trader (synthetic size, 1e18 units)
        uint256 coalitionSize; // skew-reducing LONG of the coalition trader
    }

    struct Run {
        uint256 poolAssetsBeforeOpen;
        int256 coalitionVpiAccrued;
        uint256 reserveAfterOpen;
        uint256 lpEquityDropAtOpen;
        uint256 entrantShares;
        uint256 poolUsdgBeforeClose;
        uint256 poolUsdgAfterClose;
        uint256 reserveAfterClose;
        uint256 entrantPaid;
        uint256 passivePaid;
        uint256 seniorPaid;
        uint256 coalitionTraderPaid;
        uint256 treasury;
        uint256 held;
    }

    struct Result {
        Run cmp;
        Run atk;
        uint256 rebate;
        uint256 entrantShareBps;
        int256 passiveLoss;
        int256 coalitionGain;
        int256 treasuryGain;
    }

    uint256 internal passiveShares;
    uint256 internal seniorShares;
    uint256 internal depositRequest;

    // ------------------------------------------------------------------ tests

    /// Passive Junior LP 10,000,000 USDG, coalition Junior deposit 30,000,000 USDG, SHORT skew 3,960,000 USDG notional
    /// (39.6% of pool depth) held by an independent trader, coalition LONG of the same size.
    function test_entrantCapturesClawbackFromPassiveJuniorLp() public {
        Result memory r = _scenario(_base(), "passive Junior 10M, entrant Junior 30M, skew 39.6% of depth");

        // The open pays the rebate from pool cash and locks an equal clawback reserve outside LP equity.
        assertEq(r.rebate, 7_834_655_437, "negative VPI rebate on the skew-reducing open");
        assertEq(r.atk.reserveAfterOpen, r.rebate, "VPI reserve = max(-vpiAccrued, 0)");
        assertEq(r.atk.lpEquityDropAtOpen, r.rebate, "pending LP equity falls by the full rebate at the open");
        assertEq(r.cmp.lpEquityDropAtOpen, 0, "no LP equity change without the open");

        // The close returns the clawback to HousePool cash after the entrant's deposit was priced.
        assertEq(r.atk.reserveAfterClose, 0, "reserve consumed at the close");
        assertGe(r.atk.poolUsdgAfterClose - r.atk.poolUsdgBeforeClose, r.rebate, "close returns at least the rebate");
        assertEq(r.treasuryGain, 3_168e6, "coalition execution fees go to the treasury");

        // Value moved from the passive Junior LP to the coalition.
        assertGt(r.passiveLoss, 0, "passive Junior LP loses");
        assertApproxEqRel(uint256(r.passiveLoss), r.rebate * r.entrantShareBps / 10_000, 0.002e18, "loss = share * rebate");
        assertEq(r.passiveLoss, 5_880_565_384, "passive Junior LP withdrawal loss");
        assertEq(r.coalitionGain, 2_707_763_309, "coalition cash gain over the comparison run");
        assertEq(10_000_000e6 - r.atk.passivePaid, 4_286_445_819, "passive Junior LP withdraws less than its deposit");
        assertGt(r.atk.entrantPaid + r.atk.coalitionTraderPaid, 30_000_000e6 + TRADER_FUNDS, "coalition exits above funding");
    }

    /// Same history with a 2,000,000 USDG entrant: the coalition loses money and the passive Junior LP still loses.
    function test_passiveJuniorLpLosesWhenCoalitionDoesNotProfit() public {
        Config memory c = _base();
        c.entrant = 2_000_000e6;
        Result memory r = _scenario(c, "entrant Junior 2M, coalition unprofitable");
        assertGt(r.passiveLoss, 1_000e6, "passive Junior LP loses");
        assertApproxEqRel(uint256(r.passiveLoss), r.rebate * r.entrantShareBps / 10_000, 0.002e18, "loss = share * rebate");
        assertLt(r.coalitionGain, 0, "coalition pays more in fees than it captures");
    }

    /// 5,000,000 USDG entrant (one third of Junior supply) with a smaller coalition LONG.
    function test_oneThirdJuniorShareWithSmallerOpen() public {
        Config memory c = _base();
        c.entrant = 5_000_000e6;
        c.coalitionSize = 1_300_000e18;
        Result memory r = _scenario(c, "entrant Junior 5M, coalition LONG 1.56M notional");
        assertGt(r.passiveLoss, 0, "passive Junior LP loses");
        assertApproxEqRel(uint256(r.passiveLoss), r.rebate * r.entrantShareBps / 10_000, 0.002e18, "loss = share * rebate");
        assertGt(r.coalitionGain, 0, "coalition profits at one third of Junior supply");
        assertLt(r.atk.passivePaid, 10_000_000e6, "passive Junior LP withdraws less than its deposit");
    }

    /// Senior 7,500,000 USDG, passive Junior 2,500,000 USDG, entrant 2,500,000 USDG; the Senior LP exits in the same
    /// epoch as the passive Junior LP so the Junior covenant does not cap the Junior redemption.
    function test_seniorHeavyPoolHalfJuniorShare() public {
        Config memory c = _base();
        c.passiveJunior = 2_500_000e6;
        c.senior = 7_500_000e6;
        c.entrant = 2_500_000e6;
        Result memory r = _scenario(c, "Senior 7.5M, passive Junior 2.5M, entrant Junior 2.5M");
        assertGt(r.passiveLoss, 0, "passive Junior LP loses");
        assertApproxEqRel(uint256(r.passiveLoss), r.rebate * r.entrantShareBps / 10_000, 0.002e18, "loss = share * rebate");
        assertGt(r.coalitionGain, 0, "coalition profits");
        assertEq(r.atk.seniorPaid, r.cmp.seniorPaid, "Senior withdrawal unchanged");
        assertLt(r.atk.passivePaid, 2_500_000e6, "passive Junior LP withdraws less than its deposit");
    }

    // ------------------------------------------------------------------ scenario

    function _base() internal pure returns (Config memory c) {
        c.passiveJunior = 10_000_000e6;
        c.entrant = 30_000_000e6;
        c.skewSize = 3_300_000e18; // 3,960,000 USDG notional at a 1.20 basket
        c.coalitionSize = 3_300_000e18;
    }

    function _scenario(Config memory c, string memory label) internal returns (Result memory r) {
        console2.log("==", label);
        _setup(c);
        uint256 snap = vm.snapshotState();
        r.cmp = _run(c, false);
        vm.revertToState(snap);
        r.atk = _run(c, true);

        r.rebate = r.atk.coalitionVpiAccrued < 0 ? uint256(-r.atk.coalitionVpiAccrued) : 0;
        r.entrantShareBps = r.atk.entrantShares * 10_000 / (r.atk.entrantShares + passiveShares);
        r.passiveLoss = int256(r.cmp.passivePaid) - int256(r.atk.passivePaid);
        r.coalitionGain = int256(r.atk.entrantPaid + r.atk.coalitionTraderPaid)
            - int256(r.cmp.entrantPaid + r.cmp.coalitionTraderPaid);
        r.treasuryGain = int256(r.atk.treasury) - int256(r.cmp.treasury);
        assertEq(r.atk.held, r.cmp.held, "same USDG supply in both runs");

        console2.log("  passive Junior LP deposit                        ", _usd(int256(c.passiveJunior)));
        if (c.senior > 0) console2.log("  Senior LP deposit                                ", _usd(int256(c.senior)));
        console2.log("  coalition Junior LP deposit                      ", _usd(int256(c.entrant)));
        console2.log("  coalition trader funding                         ", _usd(int256(TRADER_FUNDS)));
        console2.log("  HousePool assets before the open                 ", _usd(int256(r.atk.poolAssetsBeforeOpen)));
        console2.log("  independent SHORT notional                       ", _usd(int256(_notional(c.skewSize))));
        console2.log("  coalition LONG notional                          ", _usd(int256(_notional(c.coalitionSize))));
        console2.log("  negative VPI rebate on the open (vpiAccrued)     ", _usd(r.atk.coalitionVpiAccrued));
        console2.log("  coalition VPI reserve after the open             ", _usd(int256(r.atk.reserveAfterOpen)));
        console2.log("  pending LP equity drop at the open               ", _usd(int256(r.atk.lpEquityDropAtOpen)));
        console2.log("  coalition share of Junior supply (bps)           ", r.entrantShareBps);
        console2.log("  HousePool USDG change over the coalition close   ",
            _usd(int256(r.atk.poolUsdgAfterClose) - int256(r.atk.poolUsdgBeforeClose)));
        console2.log("  coalition VPI reserve after the close            ", _usd(int256(r.atk.reserveAfterClose)));
        console2.log("  passive Junior LP withdrawal, comparison         ", _usd(int256(r.cmp.passivePaid)));
        console2.log("  passive Junior LP withdrawal, attack             ", _usd(int256(r.atk.passivePaid)));
        console2.log("  passive Junior LP loss                           ", _usd(r.passiveLoss));
        console2.log("  passive Junior LP result vs deposit, attack      ",
            _usd(int256(r.atk.passivePaid) - int256(c.passiveJunior)));
        console2.log("  coalition Junior LP withdrawal, comparison       ", _usd(int256(r.cmp.entrantPaid)));
        console2.log("  coalition Junior LP withdrawal, attack           ", _usd(int256(r.atk.entrantPaid)));
        console2.log("  coalition trader withdrawal, comparison          ", _usd(int256(r.cmp.coalitionTraderPaid)));
        console2.log("  coalition trader withdrawal, attack              ", _usd(int256(r.atk.coalitionTraderPaid)));
        console2.log("  coalition cash gain over the comparison          ", _usd(r.coalitionGain));
        console2.log("  coalition result vs funded capital, attack       ",
            _usd(int256(r.atk.entrantPaid + r.atk.coalitionTraderPaid) - int256(c.entrant + TRADER_FUNDS)));
        console2.log("  treasury gain (coalition execution fees)         ", _usd(r.treasuryGain));
        if (c.senior > 0) {
            console2.log("  Senior LP withdrawal, comparison                 ", _usd(int256(r.cmp.seniorPaid)));
            console2.log("  Senior LP withdrawal, attack                     ", _usd(int256(r.atk.seniorPaid)));
        }
        console2.log("  USDG in custody contracts + wallets = supply     ", _usd(int256(r.atk.held)));
    }

    /// @dev USDG notional of a synthetic size at the 1.20 basket.
    function _notional(uint256 size) internal pure returns (uint256) {
        return size * BASKET_1_20 / 1e20;
    }

    /// Shared state before the snapshot: LP positions, trader funding, the independent SHORT and the pending Junior
    /// deposit of the coalition's LP (cancellable until its epoch is active).
    function _setup(Config memory c) internal {
        passiveShares = _lpDeposit(false, passiveJuniorLp, c.passiveJunior, BASKET_1_20);
        if (c.senior > 0) seniorShares = _lpDeposit(true, seniorLp, c.senior, BASKET_1_20);
        _fundTrader(skewTrader, TRADER_FUNDS);
        _fundTrader(coalitionTrader, TRADER_FUNDS);
        _requireExecuted(_openPosition(skewTrader, CfdTypes.Side.SHORT, c.skewSize, 500_000e6, BASKET_1_20), "skew open");
        depositRequest = _requestDeposit(false, entrantJuniorLp, c.entrant);
    }

    function _run(Config memory c, bool attack) internal returns (Run memory o) {
        uint256 boundary = d.housePool.lpEpochStart(depositRequest);

        // 1. 60 s before the LP epoch boundary the coalition trader opens a skew-reducing LONG (attack run only).
        _warpTo(boundary - 60);
        uint256 equityBefore = _pendingLpEquity();
        o.poolAssetsBeforeOpen = d.housePool.totalAssets();
        if (attack) {
            _requireExecuted(
                _openPosition(coalitionTrader, CfdTypes.Side.LONG, c.coalitionSize, 500_000e6, BASKET_1_20), "coalition open"
            );
            (,,,,,, o.coalitionVpiAccrued) = d.engine.positions(coalitionTrader);
            o.reserveAfterOpen = d.clearinghouse.vpiRebateReserveAssets(coalitionTrader);
        } else {
            _sameTimeAsOrder();
        }
        uint256 equityAfter = _pendingLpEquity();
        o.lpEquityDropAtOpen = equityBefore > equityAfter ? equityBefore - equityAfter : 0;

        // 2. At the boundary any address settles the LP epoch; the coalition's Junior deposit is activated.
        _settleEpoch(depositRequest, BASKET_1_20);
        o.entrantShares = _claimDeposit(false, entrantJuniorLp, depositRequest);

        // 3. 10 s later the coalition trader closes the LONG; the clawback returns to HousePool cash.
        _warp(10);
        if (attack) {
            o.poolUsdgBeforeClose = _usdgOf(address(d.housePool));
            _requireExecuted(_closePosition(coalitionTrader, CfdTypes.Side.LONG, c.coalitionSize, BASKET_1_20), "close");
            o.poolUsdgAfterClose = _usdgOf(address(d.housePool));
            o.reserveAfterClose = d.clearinghouse.vpiRebateReserveAssets(coalitionTrader);
        } else {
            _sameTimeAsOrder();
        }

        // 4. After the 1 h deposit cooldown the coalition's Junior LP redeems in full.
        _warp(3600);
        uint256 exitRequest = _requestRedeem(false, entrantJuniorLp, o.entrantShares);
        _settleEpoch(exitRequest, BASKET_1_20);
        assertEq(d.juniorVault.claimableRedeemRequest(exitRequest, entrantJuniorLp), o.entrantShares, "entrant funded");
        o.entrantPaid = _claimRedeem(false, entrantJuniorLp, exitRequest);

        // 5. The independent trader closes its SHORT; one epoch later the passive Junior LP (and Senior LP) redeem.
        _warp(10);
        _requireExecuted(_closePosition(skewTrader, CfdTypes.Side.SHORT, c.skewSize, BASKET_1_20), "skew close");
        _warp(3600);
        uint256 seniorRequest;
        if (c.senior > 0) seniorRequest = _requestRedeem(true, seniorLp, seniorShares);
        uint256 passiveRequest = _requestRedeem(false, passiveJuniorLp, passiveShares);
        _settleEpoch(passiveRequest, BASKET_1_20);
        if (c.senior > 0) {
            assertEq(d.seniorVault.claimableRedeemRequest(seniorRequest, seniorLp), seniorShares, "Senior funded");
            o.seniorPaid = _claimRedeem(true, seniorLp, seniorRequest);
        }
        assertEq(
            d.juniorVault.claimableRedeemRequest(passiveRequest, passiveJuniorLp), passiveShares, "passive LP fully funded"
        );
        assertEq(d.juniorVault.pendingRedeemRequest(passiveRequest, passiveJuniorLp), 0, "no pending passive LP shares");
        o.passivePaid = _claimRedeem(false, passiveJuniorLp, passiveRequest);

        // 6. Both traders withdraw their whole clearinghouse balance.
        o.coalitionTraderPaid = _cashOut(coalitionTrader);
        _cashOut(skewTrader);
        o.treasury = d.clearinghouse.balanceAssets(d.engine.protocolTreasury());
        o.held = _held();
        assertEq(o.held, d.settlementToken.totalSupply(), "all USDG held by custody contracts and actor wallets");
    }

    // ------------------------------------------------------------------ helpers

    /// @dev Senior plus Junior pending LP equity after a permissionless mark refresh at the current price.
    function _pendingLpEquity() internal returns (uint256) {
        _refreshMark(BASKET_1_20);
        (uint256 seniorEquity, uint256 juniorEquity,,) = d.housePool.getPendingTrancheState();
        return seniorEquity + juniorEquity;
    }

    /// @dev Advances time and blocks exactly as an order execution does, so both runs stay aligned.
    function _sameTimeAsOrder() internal {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
    }

    function _cashOut(address t) internal returns (uint256 bal) {
        bal = d.clearinghouse.balanceAssets(t);
        if (bal > 0) {
            vm.prank(t);
            d.clearinghouse.withdrawMargin(bal);
        }
        assertEq(d.clearinghouse.balanceAssets(t), 0, "trader balance withdrawn");
    }

    /// @dev USDG in the five custody contracts plus every actor wallet; no USDG is minted inside a run.
    function _held() internal view returns (uint256 sum) {
        sum = _custodyUsdg();
        address[7] memory w = [passiveJuniorLp, seniorLp, entrantJuniorLp, skewTrader, coalitionTrader, keeper, owner];
        for (uint256 i; i < 7; ++i) sum += _usdgOf(w[i]);
    }

    function _usd(int256 a) internal pure returns (string memory) {
        uint256 u = a < 0 ? uint256(-a) : uint256(a);
        bytes memory frac = bytes(vm.toString(1e6 + u % 1e6));
        frac[0] = ".";
        return string.concat(a < 0 ? "-" : "", vm.toString(u / 1e6), string(frac), " USDG");
    }

}
```

