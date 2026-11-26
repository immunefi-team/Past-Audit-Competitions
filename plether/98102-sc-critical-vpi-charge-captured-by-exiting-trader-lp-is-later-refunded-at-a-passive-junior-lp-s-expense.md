# \#98102 [SC-Critical] VPI charge captured by exiting Trader/LP is later refunded at a passive Junior LP's expense

**Submitted on Nov 5th 2026 at 04:45:44 UTC by @Rrudy for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98102
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Summary

At the in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, one actor can control both a Trader and a Junior LP. The Trader opens a position and pays positive VPI into `HousePool` revenue. The actor then has its matured Junior redemption **priced and funded while that position is still open**, locking in its share of that revenue. The Trader subsequently closes at the same market price and receives a VPI rebate from Pool cash. The departing LP no longer bears that cash debit; an unrelated Junior LP who stays invested receives less USDG when it later redeems.

The issue is **which LP receives the open charge and which LP pays for its later refund**. It does not require an oversized VPI refund, a price-PnL gain, a privileged account, or a race against another user's committed close. The Trader controls when it first commits its own close. `claimRedeem()` demonstrates actual delivery of the already funded amount; the relevant pricing boundary is **LP epoch funding**, when shares are burned and the cash claim is fixed.

### Root cause and attack path

1. On open, [`CfdEngineSettlementSidecar.executeOpen`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L445-L464) routes positive action cash to [`HousePool.recordClaimantInflow(..., Revenue, CashArrived)`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L647). It becomes current Pool revenue and increases `accountedAssets`.
2. The attacker Junior LP's existing redemption request matures. [`HousePool._settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L891) prices and funds it while the Trader position remains open. The funded shares are burned; the resulting USDG claim is fixed. The attacker can claim that cash before making its first close commitment.
3. The Trader closes at the unchanged mark. [`CloseAccountingLib`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L102-L125) applies the lifetime clamp; the gross negative VPI never exceeds the position's positive accrued VPI. Nevertheless, the [settlement sidecar pays the net action rebate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624) from Pool cash after the attacker LP has left. Remaining LP capital bears that payment.

This report does not assert a miscalculated trader-side `vpiRebateReserveAssets`. That reserve backs negative accrued VPI; the tested positive accrued VPI keeps its target at zero. The asserted economic gap is that a charge which may later be returned to the same Trader can be captured in that Trader's LP exit before the corresponding Pool outflow is imposed on a different LP cohort.

### Measured third-party harm

The runnable PoC below deploys the unmodified production `OrderRouter` and the rest of the pinned in-scope contracts using the repository's test setup and permitted Pyth/USDG mocks. It uses the release six-feed basket configuration and `vpiFactor = 0.01e18`. The Trader's open executes **five seconds before** the scheduled epoch boundary. Both branches start from the same saved state, use the same market mark and final observation time, and settle the same matured LP requests. Realized price PnL is **zero**.

| One cycle, final actual USDG wallets | Close before LP funding | LP funding before close | Attack minus control |
| --- | ---: | ---: | ---: |
| Attacker Trader + Junior LP | 69,988.622669 | 70,006.681289 | **+18.058620** |
| Unrelated Junior LP | 15,001.816420 | 14,983.759001 | **−18.057419** |

The attacker starts with 70,000 USDG across the two roles and finishes **6.681289 USDG above that principal before external costs** in the attack branch. The unrelated LP starts with 15,000 USDG and finishes **16.240999 USDG below it**. Its redemption is fully funded and claimed in actual USDG. The branches' Pool-paid rebates differ by only **0.013038 USDG**, far smaller than the victim's 18.057419 USDG relative shortfall. The attacker/victim/Pool difference reconciles to the USDG atom: `+18,058,620 − 18,057,419 − 1,201 = 0`.

The same original-code test also executes **1, 2, 3 and 5 continuous cycles**, with attacker Junior ownership of 25%, 50%, or 75%. It reuses actual claimed wallet USDG for the next LP deposit and Trader funding, respects cooldowns and epoch timing, and makes **no token mint after the initial setup**. The passive victim keeps its shares through every cycle and claims actual USDG only at the end. Each case compares close-before-funding and funding-before-close from an identical initial state; there is no snapshot reset between cycles inside a branch. In **all 12 supported cases**, the test asserts that the combined Trader/LP receives more, the passive LP receives less, and the Pool residual decreases by the atom-exact balancing amount. These are **relative branch differences**; not every matrix case yields an absolute attacker profit or leaves the passive LP below its starting principal. A 90% full-exit variation was tested separately but is not claimed: its first attacker exit is only partially fundable because Pool free cash reaches zero.

| 75% attacker Junior ownership, five cycles | Close before LP funding | LP funding before close | Attack minus control |
| --- | ---: | ---: | ---: |
| Attacker Trader + Junior LP actual USDG | 67,441.632655 | 67,529.208666 | **+87.576011** |
| Passive Junior LP actual redemption | 17,510.734313 | 17,423.163296 | **−87.571017** |
| Residual Pool cash | 2.000815 | 1.995821 | **−0.004994** |

The five-cycle attack branch leaves the attacker **29.208666 USDG above its initial 67,500 USDG** before external costs; the victim is **76.836704 USDG below its initial 17,500 USDG**. The three branch deltas sum to zero at atom precision. Senior final USDG, token supply and final timestamp match between branches. Open and close execution fees are already reflected in the actual wallet balances. These are local simulated contract outcomes, not observed mainnet transactions.

A separate matched six-feed negative control sets only the **test** VPI factor to zero. The victim's attack-minus-control difference then becomes `+0.000015 USDG`, instead of `−18.057419 USDG` with release VPI. This is a causal control, not a proposed live configuration. Supplementary original-code Trader-only and No-Trader controls also distinguish the combined-role extraction from ordinary LP settlement. The self-contained PoC field below reproduces the main and repeated original-contract tests within the form's character limit; the additional control sources are available separately if triage requests them.

### Actual v2.0.0 deployed-address fork corroboration

I also ran a **local Arbitrum Sepolia fork of the published v2.0.0 test deployment**, at block `325603167` (block hash `0x238620030acfc242e5dd73676fbad87e352dfeebd8848604a13af7ce0f1a986c`). This test called the *deployed* [`HousePool` at `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json), Junior Vault `0x4a8b67184a1e3224c11a191cc69f22c50eef734e`, `OrderRouter` `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444`, and `PletherOracle` `0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2`. The harness asserted their deployed code hashes. Both orders (`32` and `33`) executed using previously captured **signed six-feed Pyth update bytes**, verified by the deployed Oracle; no local Pyth mock was used in this supplemental fork. No transaction was sent to the public deployment.

The fork prepared distinct LP actors through a **local-fork-only impersonation of the release owner** to transfer existing testnet Junior shares, and funded the Trader with the release's freely mintable **test** USDG. That actor setup is not an attack step or a claim that the attacker can take the owner's shares. The fixture declares signed-update execution times `1793702540` and `1793702598` (58 seconds apart). The harness commits each order one second before its respective execution time, advances the local fork clock with `vm.warp`, and asserts that the deployed Oracle accepts both updates and that orders `32` and `33` execute. This is a **local replay**, not proof that those transactions occurred on the public testnet. It then compares `close execution → LP epoch funding` against `LP epoch funding → close execution` from the **same fork snapshot**, using the same signed update for both final branches. The attacker's Junior exit was fully funded and claimed in actual test USDG. Relative to the close-first branch, the funding-first branch paid the attacker LP **28.300567 more USDG**, left the Trader's cash **unchanged**, and lowered Pool cash by **28.300567 USDG**. The unrelated LP's **retained Junior share claim value** was **9.433523 USDG lower** and the remaining owner's Junior claim value was **18.867045 USDG lower** (their combined difference is one atom larger because of share conversion rounding). These latter two figures are `convertToAssets` rights, **not redeemed wallet cash in this fork**; the self-contained PoC below separately measures an unrelated LP's actual redeemed wallet cash. The fork test passed `1/1`; its harness SHA-256 is `d8dad448443d6263af32d1da57d790cd204f241926149aa90fc87de42e4f0118`, saved signed-update fixture SHA-256 is `690f95a682abfe821d8fb70c106c6e8a49eff50e1c046e635eff066f821e5c2a` (the copyable LF version in the supplemental fork packet hashes to `67f8d93e121ec7a9d88dac0419bac2b31d2cee73ff313e0b16cd980455339175`), and saved stdout SHA-256 is `6f4b8b35bb24ec95fe60915d41e8c5cf6a819ecf744e82ea66d1af58fc080d7c`. The fork uses the published testing deployment's liquidity and MockUSDG setup, **not the prospective production configuration**. It corroborates reachability with deployed addresses and signed prices; the separate VPI-zero control above is the causal test of VPI itself. The optional secret Gist contains the complete fork source, signed bytes, exact replay command and result excerpt so triage can reproduce this supplemental claim. The main original-code test is fully included in the Proof of Concept field.

### Public issue comparison and eligibility question

[PR #62](https://github.com/Plether-Fi/plether-core/pull/62) addresses LP entry/exit pricing of **marked price PnL**. It considered and rejected cohort accounting as a remedy for inherited marked liabilities; it did not describe the same Trader's realized VPI charge being captured by an LP exit before a later close rebate. Here price PnL is zero and the transfer concerns that charge and refund. The published [AI scan VPI item](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1808) uses a temporary reduction in Pool depth to **increase the close rebate itself**. The measured single-cycle rebate difference here is 0.013038 USDG; the five-cycle aggregate Pool-paid rebate difference is 0.068976 USDG, versus 87.571017 USDG less received by the passive LP. The first disputed boundary here is **LP epoch funding fixing the departing LP's cash entitlement while the same position's later VPI rebate remains conditional**, not an oversized rebate calculation. This report concerns **cohort allocation** even where rebate-size manipulation is immaterial.

The pinned [SECURITY.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L862-L872) states that anticipated VPI is excluded from terminal NAV and new close rebates use available Pool cash. Those published component rules are the strongest objection: the project may regard the observed cohort transfer as intended time-based LP economics. The report asks triage to distinguish that interpretation from the **combined Trader/LP extraction and passive-LP cash loss demonstrated here**. The public documents reviewed do not expressly describe that sequence or its measured consequence. I do not claim that merely showing a different wallet amount proves an accounting conservation violation, or that this question has already been resolved in my favor.

### Preconditions, impact and suggested remediation

The attacker needs a Trader position, Junior LP shares with a matured redemption request, and valid executions before and after funding. The self-contained PoC uses separate Trader and LP addresses to show each cash flow, then counts both wallets as **one economic actor**. It uses no per-address reward, eligibility rule, account cap or identity limit; the alleged impact comes from the order of LP funding and the same actor's later VPI rebate, not from multiplying identities. A single-address version was not tested. The self-contained PoC uses mock updates for six configured Pyth feeds; the supplemental deployed-address fork uses signed six-feed updates. Neither test measures the frequency of the exact favorable market sequence on Arbitrum One, transaction inclusion, Arbitrum gas, Hermes/Pyth fees, or capital opportunity cost. Those costs are **not included** in the 29.208666 USDG five-cycle absolute surplus. The claimed in-scope impact is **Medium — Griefing: reduced actual cash received by an unrelated Junior LP in the self-contained PoC**. It is not a claim of guaranteed net arbitrage, direct theft, insolvency or freezing.

One possible mitigation is to avoid fixing an LP exit amount that includes a positive VPI charge while its originating live position may later receive a conditional rebate, or to account for that exposure at the funding boundary. Any change should preserve the existing lifetime clamp, trader-claim priority and Senior/Junior waterfall.

## Link to Proof of Concept

https://gist.github.com/0xRrudy/fed9c9cd31a609fb70c0e64a000e3128

## Proof of Concept

### Reproduction steps

1. In a disposable local test environment, obtain the competition repository and pin the exact scope commit. Use Foundry 1.5.1 and Solidity 0.8.35 with the repository's submodules. Do not send transactions to the public deployment.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

2. Save the complete Solidity block below as `packages/perps/test/perps/spec/trader/VpiEpochCohortSubmissionFinal.t.sol`. It inherits the repository's `BasePerpTest` for permitted Pyth/USDG mocks, deploys the original production `OrderRouter`, and changes no file under `packages/perps/src`. The six configured Pyth feed IDs, weights, base prices and inversion flags are set in the fixture. Source SHA-256 (LF bytes): `f7488342ca4f49d8569922992ed38fe3db569a90e31541987bfff41a61f11caa`.

3. From the repository root, run the whole test contract. `--offline` works after the pinned toolchain and submodules are installed in the disposable environment.

```bash
FOUNDRY_PROFILE=default forge test --root packages/perps \
  --match-path test/perps/spec/trader/VpiEpochCohortSubmissionFinal.t.sol \
  --match-contract VpiPassiveCycleMatrixExperimentTest \
  --match-test test_Q261_ --offline -vv
```

The contract contains the single-cycle proof plus 1/2/3/5 continuous-cycle tests for 25%, 50% and 75% attacker Junior ownership. The 90% full-exit case is excluded because it is only partially fundable and is not claimed. The final exact file returned **13 passed, 0 failed**, exit code 0, in a no-network isolated Foundry run. Its saved stdout SHA-256 is `cb7be192d7eeb27466256cbbb2b15885f716cc4089ae692fbbf924970a6ffd62`.

### Complete test source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Q261 claim-first fixture: copy only into pinned offline scratch.
// Only the repository's BasePerpTest is inherited; in-scope contracts remain untouched.
import {BasePerpTest} from "../../BasePerpTest.sol";
import {Vm} from "forge-std/Vm.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {LegacyOrderRouterHarness} from "../../../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract VpiPassiveCycleMatrixExperimentTest is BasePerpTest {
    address internal constant JUNIOR_LP = address(0xA11CE);
    address internal constant SENIOR_LP = address(0xB0B);
    address internal constant KEEPER = address(0xB0A7D);
    uint256 internal constant ENTRY_PRICE = 1e8;
    address private constant LOSS_TRADER232 = address(0xC232);
    address private constant JUNIOR_VICTIM232 = address(0xC233);
    uint256 private constant SENIOR_ACTIVE232 = 10_000e6;
    uint256 private constant JUNIOR_ACTIVE232 = 70_000e6;
    uint256 private constant LOSS_SIZE232 = 22_500e18;
    uint256 private constant LOSS_MARGIN232 = 13_000e6;
    uint256 private constant LOSS_MARK232 = 100_000_000;

    function _deployLegacyOrderRouter(
        address engine_, address engineLens_, address pool_, address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        bytes32[] memory ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        uint256[] memory weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
        uint256[] memory bases = new uint256[](6);
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
        bool[] memory inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
        pletherOracle = new PletherOracle(
            engine_, pool_, address(baseMockPyth), ids, weights, bases, inversions
        );
        oracle_ = address(pletherOracle);
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar =
            new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter exactRouter = new OrderRouter(
            engine_, engineLens_, pool_, oracle_, address(keeperSidecar),
            address(policyEvaluator), address(orderExecutionSidecar), address(lifecycleBook)
        );
        deployedRouter = LegacyOrderRouterHarness(address(exactRouter));
        assertEq(address(deployedRouter), predictedRouter);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        BasePerpTest.setUp();
        CfdTypes.RiskParams memory risk = _riskParams();
        risk.vpiFactor = 0.01e18;
        risk.maintMarginBps = 10;
        risk.initMarginBps = 20;
        _setRiskParams(risk);

        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1_000e6;
        routerConfig.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior ratio");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minimum open");
    }

    function _sixFeedUpdateData(uint256 mark) internal returns (bytes[] memory updateData) {
        assertEq(mark, ENTRY_PRICE, "same-price six-feed fixture");
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        bytes32[6] memory ids = [
            bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
            bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
            bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
            bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
            bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
            bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
        ];
        int64[6] memory prices = [
            int64(117_500_000), int64(15_673_981_191), int64(134_480_000),
            int64(137_211_855), int64(920_810_313), int64(79_302_141)
        ];
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: prices[i], conf: 0, expo: -8,
                    publishTime: publishTime, prevPublishTime: 0
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _publicDeposit(TrancheVault vault, address lp, uint256 assets)
        internal returns (uint256 shares)
    {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp);
        vm.stopPrank();
        uint256 epochStart = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < epochStart) vm.warp(epochStart);
        bytes[] memory tick = _sixFeedUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
        shares = vault.claimableDepositShares(id, lp);
        assertGt(shares, 0, "LP shares activated");
        vm.prank(lp);
        uint256 delivered = vault.claimDeposit(id, assets, lp, lp);
        assertEq(delivered, shares, "LP shares delivered");
        assertEq(vault.balanceOf(lp), shares, "LP share balance");
    }

    struct Outcome232 {
        uint256 bPaid;
        uint256 cPaid;
        uint256 bFunded;
        uint256 cFunded;
        uint256 bPending;
        uint256 cPending;
        uint256 bQuote;
        uint256 cQuote;
        uint256 bWallet;
        uint256 cWallet;
        uint256 bNewDepositClaimable;
        uint256 bNewSharesClaimable;
        uint256 reservation;
        uint256 seniorPrincipal;
        uint256 seniorHwm;
        uint256 juniorPrincipal;
        uint256 poolCash;
        uint256 poolAccounted;
        uint256 seniorVaultCash;
        uint256 juniorVaultCash;
        uint256 seniorDepositEscrow;
        uint256 seniorWithdrawalEscrow;
        uint256 juniorWithdrawalEscrow;
        uint256 traderClaim;
        uint256 pendingRevenue;
        uint256 unassigned;
        uint256 freeAssets;
        uint256 aWallet;
        uint256 aClearing;
        uint256 closeRebatePaid;
        uint256 settleTime;
        uint256 observedTime;
        uint256 secondTime;
        uint256 bSecondPaid;
        uint256 cSecondPaid;
        uint256 dSecondPaid;
        uint256 dSecondFunded;
        uint256 dSecondPending;
        uint256 dResidualQuote;
        uint256 secondSeniorPrincipal;
        uint256 secondJuniorPrincipal;
        uint256 secondPoolCash;
        uint256 cFinalPending;
        uint256 cFinalQuote;
        uint256 cClaimBeforeClose;
        uint256 cWalletBeforeEarlyClaim;
        uint256 cWalletAtEarlyClaim;
        uint256 poolCashBeforeEarlyClaim;
        uint256 poolCashAfterEarlyClaim;
        uint256 nextCommitBeforeEarlyClaim;
        uint256 nextCommitAfterEarlyClaim;
        uint256 aPositionBeforeEarlyClaim;
        uint256 aPositionAfterEarlyClaim;
        uint256 bWalletFinal;
        uint256 cWalletFinal;
        uint256 dWalletFinal;
        uint256 finalPoolAccounted;
        uint256 finalSeniorVaultCash;
        uint256 finalJuniorVaultCash;
        uint256 finalSeniorWithdrawalEscrow;
        uint256 finalJuniorWithdrawalEscrow;
    }

    function _trade232(bool close, uint256 margin, uint256 mark, bytes32 tag) private returns (uint64 id) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(tag, router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.isClose = close;
        request.sizeDelta = LOSS_SIZE232;
        request.marginDelta = margin;
        request.targetPrice = close ? engine.CAP_PRICE() : 1;
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
        vm.prank(LOSS_TRADER232);
        id = router.commitOrder(request);
        bytes[] memory tick = _sixFeedUpdateData(mark);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, tick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public Router order executed");
    }

    function _open232() private {
        vm.recordLogs();
        uint64 openId = _trade232(false, LOSS_MARGIN232, ENTRY_PRICE, keccak256("q232-loss-open"));
        Vm.Log[] memory recorded = vm.getRecordedLogs();
        bool receiptFound;
        for (uint256 i; i < recorded.length; ++i) {
            if (recorded[i].emitter != address(router.lifecycleBook()) || recorded[i].topics.length != 4
                || recorded[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector
                || recorded[i].topics[1] != bytes32(uint256(openId))) continue;
            (bytes32 receiptHash, uint64 terminalBlock, uint64 terminalTime,
                OrderV3Types.OrderReceipt memory receipt) = abi.decode(recorded[i].data,
                    (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            receiptHash; terminalBlock; terminalTime;
            assertEq(receipt.orderId, openId, "exact A open receipt");
            assertEq(receipt.executionPrice, ENTRY_PRICE, "constant price at open");
            assertEq(receipt.economics.realizedPnlAssets, int256(0), "open price PnL zero");
            assertGt(receipt.economics.vpiAssets, int256(0), "open VPI prepaid by A");
            emit log_named_uint("OWN_OPEN_VPI_CHARGE", uint256(receipt.economics.vpiAssets));
            emit log_named_uint("OWN_OPEN_EXECUTION_FEE", receipt.economics.executionFeeAssets);
            (, , , , , , int256 accrued) = engine.positions(LOSS_TRADER232);
            assertEq(accrued, receipt.economics.vpiAssets, "open receipt equals live lifetime VPI");
            emit log_named_uint("OWN_OPEN_LIVE_VPI_ACCRUED", uint256(accrued));
            assertEq(clearinghouse.vpiRebateReserveAssets(LOSS_TRADER232), 0,
                "positive accrued VPI has no trader-side negative reserve");
            receiptFound = true;
        }
        assertTrue(receiptFound, "A open terminal receipt found");
    }

    function _close232() private returns (uint256 paidRebate) {
        uint256 beforeCash = settlementToken.balanceOf(address(pool));
        (, , , , , , int256 accruedBeforeClose) = engine.positions(LOSS_TRADER232);
        assertGt(accruedBeforeClose, int256(0), "positive VPI remains live until A closes");
        assertEq(clearinghouse.vpiRebateReserveAssets(LOSS_TRADER232), 0,
            "positive accrued VPI still has zero trader-side reserve");
        emit log_named_uint("OWN_PRE_CLOSE_LIVE_VPI_ACCRUED", uint256(accruedBeforeClose));
        assertEq(engine.positionEntryCostAssetAtoms(LOSS_TRADER232),
            (LOSS_SIZE232 / CfdTypes.SIZE_QUANTUM) * LOSS_MARK232, "A same-price entry cost, zero price PnL basis");
        vm.recordLogs();
        uint64 closeId = _trade232(true, 0, LOSS_MARK232, keccak256("q261-loss-close"));
        Vm.Log[] memory recorded = vm.getRecordedLogs();
        bytes32 eventTopic = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
        bytes32 accountTopic = bytes32(uint256(uint160(LOSS_TRADER232)));
        bool found;
        bool receiptFound;
        for (uint256 i; i < recorded.length; ++i) {
            if (recorded[i].emitter == address(router.lifecycleBook()) && recorded[i].topics.length == 4
                && recorded[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                && recorded[i].topics[1] == bytes32(uint256(closeId))) {
                (bytes32 receiptHash, uint64 terminalBlock, uint64 terminalTime,
                    OrderV3Types.OrderReceipt memory receipt) = abi.decode(recorded[i].data,
                        (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                receiptHash; terminalBlock; terminalTime;
                assertEq(receipt.orderId, closeId, "exact A close receipt");
                assertEq(receipt.executionPrice, LOSS_MARK232, "A exact same-price execution");
                assertEq(receipt.economics.realizedPnlAssets, int256(0), "A actual realized price PnL zero");
                assertLt(receipt.economics.vpiAssets, int256(0), "close reverses prepaid VPI");
                emit log_named_uint("OWN_CLOSE_GROSS_VPI_REBATE", uint256(-receipt.economics.vpiAssets));
                emit log_named_uint("OWN_CLOSE_EXECUTION_FEE", receipt.economics.executionFeeAssets);
                emit log_named_int("OWN_CLOSE_CARRY_ASSESSED", receipt.economics.carryAssets);
                receiptFound = true;
            }
            if (recorded[i].topics.length != 2 || recorded[i].topics[0] != eventTopic
                || recorded[i].topics[1] != accountTopic) continue;
            (uint256 assessed, uint256 paid, uint256 waived) = abi.decode(recorded[i].data,
                (uint256, uint256, uint256));
            assertEq(assessed, paid + waived, "close rebate event balances");
            assertEq(waived, 0, "same-price rebate fully paid");
            paidRebate = paid;
            found = true;
        }
        assertTrue(found, "close rebate event from official sidecar");
        assertTrue(receiptFound, "A V3 terminal receipt with zero price PnL");
        emit log_named_uint("Q261_CLOSE_PRICE_PNL_ZERO", 1);
        assertEq(engine.traderClaimBalanceAssets(LOSS_TRADER232), 0, "A loss has no trader claim");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no PR74 shortage in Q232");
        (uint256 remaining,,,,,,) = engine.positions(LOSS_TRADER232);
        assertEq(remaining, 0, "A public close completed");
        uint256 afterCash = settlementToken.balanceOf(address(pool));
        assertLt(afterCash, beforeCash, "same-price close pays action rebate");
    }

    function _claimCBeforeClose261(Outcome232 memory o, uint256 cId) private {
        o.cWalletBeforeEarlyClaim = settlementToken.balanceOf(JUNIOR_LP);
        o.poolCashBeforeEarlyClaim = settlementToken.balanceOf(address(pool));
        uint256 accountedBefore = pool.accountedAssets();
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 hwmBefore = pool.seniorHighWaterMark();
        o.nextCommitBeforeEarlyClaim = router.nextCommitId();
        (o.aPositionBeforeEarlyClaim,,,,,,) = engine.positions(LOSS_TRADER232);
        assertEq(o.aPositionBeforeEarlyClaim, LOSS_SIZE232, "A live full position before C claim");
        vm.prank(JUNIOR_LP);
        o.cPaid = juniorVault.claimRedeem(cId, o.cFunded, JUNIOR_LP, JUNIOR_LP);
        o.cWalletAtEarlyClaim = settlementToken.balanceOf(JUNIOR_LP);
        o.cWallet = o.cWalletAtEarlyClaim;
        o.cClaimBeforeClose = 1;
        o.poolCashAfterEarlyClaim = settlementToken.balanceOf(address(pool));
        o.nextCommitAfterEarlyClaim = router.nextCommitId();
        (o.aPositionAfterEarlyClaim,,,,,,) = engine.positions(LOSS_TRADER232);
        assertEq(o.cWalletAtEarlyClaim - o.cWalletBeforeEarlyClaim, o.cPaid,
            "C actual wallet funded before first A close commit");
        assertEq(o.cPaid, o.cQuote, "C early claim exact independent share quote");
        assertEq(o.poolCashAfterEarlyClaim, o.poolCashBeforeEarlyClaim, "C claim does not move Pool cash");
        assertEq(pool.accountedAssets(), accountedBefore, "C claim does not move accounted cash");
        assertEq(pool.seniorPrincipal(), seniorBefore, "C claim does not move Senior principal");
        assertEq(pool.juniorPrincipal(), juniorBefore, "C claim does not move Junior principal");
        assertEq(pool.seniorHighWaterMark(), hwmBefore, "C claim does not move Senior HWM");
        assertEq(o.aPositionAfterEarlyClaim, o.aPositionBeforeEarlyClaim, "C claim does not close A");
        assertEq(o.nextCommitAfterEarlyClaim, o.nextCommitBeforeEarlyClaim,
            "no A close committed before C wallet received");
        assertEq(juniorVault.claimableRedeemRequest(cId, JUNIOR_LP), 0, "C claim consumed before A commit");
    }

    function _branch232(bool lossBeforeEpoch, uint256 bId, uint256 cId,
        uint256 bShares, uint256 cShares) private returns (Outcome232 memory o) {
        // Both branches use the same mark and timestamp. Only close/epoch order changes.
        bytes[] memory tick = _sixFeedUpdateData(LOSS_MARK232);
        vm.prank(KEEPER);
        router.updateMarkPrice(tick);
        if (lossBeforeEpoch) {
            o.closeRebatePaid = _close232();
        } else {
            // The baseline close consumes one MockPyth second before the epoch.
            // Match the attack epoch timestamp without changing any contract state.
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 1);
        }

        // Generate the settlement tick before quoting. BasePerpTest advances
        // timestamp by one second here, which accrues Senior coupon.
        bytes[] memory settleTick = _sixFeedUpdateData(LOSS_MARK232);
        (uint256 pricingSenior, uint256 pricingJunior) = pool.getPendingDepositTrancheState();
        uint256 supplySenior = seniorVault.accruedTotalSupply();
        uint256 supplyJunior = juniorVault.accruedTotalSupply();
        o.bQuote = bShares * (pricingSenior + 1) / (supplySenior + 1000);
        o.settleTime = vm.getBlockTimestamp();

        vm.prank(KEEPER);
        router.settleLpEpoch(settleTick);
        o.bFunded = seniorVault.claimableRedeemRequest(bId, SENIOR_LP);
        o.cFunded = juniorVault.claimableRedeemRequest(cId, JUNIOR_LP);
        o.bPending = seniorVault.pendingRedeemRequest(bId, SENIOR_LP);
        o.cPending = juniorVault.pendingRedeemRequest(cId, JUNIOR_LP);
        assertEq(o.bFunded + o.bPending, bShares, "B shares conserved");
        assertEq(o.cFunded + o.cPending, cShares, "C shares conserved");
        assertEq(o.bFunded, bShares, "B accepted Senior redeem funded");
        assertGt(o.cFunded, 0, "Q232 partial-funding gate: C nonzero");
        assertEq(o.cFunded, cShares, "C full exit before or after close");

        o.cQuote = o.cFunded * (pricingJunior + 1) / (supplyJunior + 1000);
        assertEq(seniorVault.claimableRedeemAssets(bId, SENIOR_LP), o.bQuote,
            "B independent pre-burn share quote");
        assertEq(juniorVault.claimableRedeemAssets(cId, JUNIOR_LP), o.cQuote,
            "C independent partial-share quote");

        if (!lossBeforeEpoch) {
            _claimCBeforeClose261(o, cId);
            o.closeRebatePaid = _close232();
            assertEq(router.nextCommitId(), o.nextCommitAfterEarlyClaim + 1,
                "exactly one fresh A V3 close committed after C received");
        } else {
            // Baseline close occurred before epoch; align its observation clock to attack close.
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 1);
        }
        o.observedTime = vm.getBlockTimestamp();

        uint256 bBefore = settlementToken.balanceOf(SENIOR_LP);
        uint256 cBefore = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(SENIOR_LP);
        o.bPaid = seniorVault.claimRedeem(bId, o.bFunded, SENIOR_LP, SENIOR_LP);
        if (lossBeforeEpoch) {
            vm.prank(JUNIOR_LP);
            o.cPaid = juniorVault.claimRedeem(cId, o.cFunded, JUNIOR_LP, JUNIOR_LP);
        } else {
            assertEq(cBefore, o.cWalletAtEarlyClaim, "C already paid once before close");
        }
        o.bWallet = settlementToken.balanceOf(SENIOR_LP);
        o.cWallet = settlementToken.balanceOf(JUNIOR_LP);
        assertEq(o.bWallet - bBefore, o.bPaid, "B actual USDG wallet");
        if (lossBeforeEpoch) assertEq(o.cWallet - cBefore, o.cPaid, "C baseline actual USDG wallet");
        else assertEq(o.cWallet, o.cWalletAtEarlyClaim, "C first wallet payment preserved after close");
        assertEq(o.bPaid, o.bQuote, "B exact funded entitlement");
        assertEq(o.cPaid, o.cQuote, "C exact funded entitlement");
        assertEq(seniorVault.claimableRedeemRequest(bId, SENIOR_LP), 0, "B claim consumed");
        assertEq(juniorVault.claimableRedeemRequest(cId, JUNIOR_LP), 0, "C claim consumed");
        assertEq(juniorVault.pendingRedeemRequest(cId, JUNIOR_LP), o.cPending, "C residual request retained");

        o.reservation = pool.reservedSeniorDepositAssets();
        assertEq(o.reservation, 0, "no pending Senior reservation");
        assertEq(seniorVault.pendingDepositEscrowAssets(), 0, "no pending Senior deposit escrow");
        o.seniorPrincipal = pool.seniorPrincipal();
        o.seniorHwm = pool.seniorHighWaterMark();
        o.juniorPrincipal = pool.juniorPrincipal();
        o.poolCash = settlementToken.balanceOf(address(pool));
        o.poolAccounted = pool.accountedAssets();
        o.seniorVaultCash = settlementToken.balanceOf(address(seniorVault));
        o.juniorVaultCash = settlementToken.balanceOf(address(juniorVault));
        o.seniorDepositEscrow = seniorVault.pendingDepositEscrowAssets();
        o.seniorWithdrawalEscrow = seniorVault.withdrawalEscrowAssets();
        o.juniorWithdrawalEscrow = juniorVault.withdrawalEscrowAssets();
        o.traderClaim = engine.totalTraderClaimBalanceAssets();
        o.pendingRevenue = pool.pendingTradingRevenueAssets();
        o.unassigned = pool.unassignedAssets();
        o.freeAssets = pool.getFreeAssets();
        o.aWallet = settlementToken.balanceOf(LOSS_TRADER232);
        o.aClearing = clearinghouse.balanceAssets(LOSS_TRADER232);
        assertEq(o.poolCash, pool.rawAssets(), "Pool physical raw cash");
        assertGe(o.seniorVaultCash, o.seniorDepositEscrow + o.seniorWithdrawalEscrow,
            "Senior Vault escrow physical backing");
        assertGe(o.juniorVaultCash, o.juniorWithdrawalEscrow, "Junior Vault escrow physical backing");
        assertEq(o.traderClaim, 0, "no PR74 deferred claim");

        // B exits the rest of the old Senior position. This releases covenant
        // headroom for C's old Junior request and an unrelated Junior LP D.
        uint256 bRemainingShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 dShares = juniorVault.balanceOf(JUNIOR_VICTIM232);
        assertGt(bRemainingShares, 0, "old Senior shares remain");
        assertGt(dShares, 0, "independent Junior D owns shares");
        vm.prank(SENIOR_LP);
        uint256 bSecondId = seniorVault.requestRedeem(bRemainingShares, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_VICTIM232);
        uint256 dSecondId = juniorVault.requestRedeem(dShares, JUNIOR_VICTIM232, JUNIOR_VICTIM232);
        assertEq(bSecondId, dSecondId, "B and D second epoch");
        uint256 secondStart = pool.lpEpochStart(dSecondId);
        if (vm.getBlockTimestamp() < secondStart) vm.warp(secondStart);
        bytes[] memory secondTick = _sixFeedUpdateData(LOSS_MARK232);
        o.secondTime = vm.getBlockTimestamp();
        vm.prank(KEEPER);
        router.settleLpEpoch(secondTick);

        uint256 bSecondFunded = seniorVault.claimableRedeemRequest(bSecondId, SENIOR_LP);
        uint256 cSecondFunded = juniorVault.claimableRedeemRequest(cId, JUNIOR_LP);
        o.dSecondFunded = juniorVault.claimableRedeemRequest(dSecondId, JUNIOR_VICTIM232);
        o.dSecondPending = juniorVault.pendingRedeemRequest(dSecondId, JUNIOR_VICTIM232);
        assertEq(o.dSecondFunded + o.dSecondPending, dShares, "D shares conserved");
        assertGt(bSecondFunded, 0, "Senior exit releases headroom");
        uint256 bBeforeSecond = settlementToken.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(bSecondId, bSecondFunded, SENIOR_LP, SENIOR_LP);
        o.bSecondPaid = settlementToken.balanceOf(SENIOR_LP) - bBeforeSecond;
        if (cSecondFunded != 0) {
            uint256 cBeforeSecond = settlementToken.balanceOf(JUNIOR_LP);
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeem(cId, cSecondFunded, JUNIOR_LP, JUNIOR_LP);
            o.cSecondPaid = settlementToken.balanceOf(JUNIOR_LP) - cBeforeSecond;
        }
        if (o.dSecondFunded != 0) {
            uint256 dBeforeSecond = settlementToken.balanceOf(JUNIOR_VICTIM232);
            vm.prank(JUNIOR_VICTIM232);
            juniorVault.claimRedeem(dSecondId, o.dSecondFunded, JUNIOR_VICTIM232, JUNIOR_VICTIM232);
            o.dSecondPaid = settlementToken.balanceOf(JUNIOR_VICTIM232) - dBeforeSecond;
        }
        o.secondSeniorPrincipal = pool.seniorPrincipal();
        o.secondJuniorPrincipal = pool.juniorPrincipal();
        o.secondPoolCash = settlementToken.balanceOf(address(pool));
        o.cFinalPending = juniorVault.pendingRedeemRequest(cId, JUNIOR_LP);
        o.cFinalQuote = o.cFinalPending * (o.secondJuniorPrincipal + 1)
            / (juniorVault.accruedTotalSupply() + 1000);
        o.dResidualQuote = o.dSecondPending * (o.secondJuniorPrincipal + 1)
            / (juniorVault.accruedTotalSupply() + 1000);
        assertEq(o.secondPoolCash, pool.rawAssets(), "second Pool physical cash");
        uint256 withdrawableA = clearinghouse.balanceAssets(LOSS_TRADER232);
        vm.prank(LOSS_TRADER232);
        clearinghouse.withdrawMargin(withdrawableA);
        o.aWallet = settlementToken.balanceOf(LOSS_TRADER232);
        assertEq(o.aWallet, withdrawableA, "A actual USDG wallet withdrawal");
        assertEq(clearinghouse.balanceAssets(LOSS_TRADER232), 0, "A clearing emptied");
        o.bWalletFinal = settlementToken.balanceOf(SENIOR_LP);
        o.cWalletFinal = settlementToken.balanceOf(JUNIOR_LP);
        o.dWalletFinal = settlementToken.balanceOf(JUNIOR_VICTIM232);
        o.finalPoolAccounted = pool.accountedAssets();
        o.finalSeniorVaultCash = settlementToken.balanceOf(address(seniorVault));
        o.finalJuniorVaultCash = settlementToken.balanceOf(address(juniorVault));
        o.finalSeniorWithdrawalEscrow = seniorVault.withdrawalEscrowAssets();
        o.finalJuniorWithdrawalEscrow = juniorVault.withdrawalEscrowAssets();
        assertEq(o.bWalletFinal, o.bWallet + o.bSecondPaid, "B final actual wallet ledger");
        assertEq(o.cWalletFinal, o.cWallet + o.cSecondPaid, "C final actual wallet ledger");
        assertEq(o.finalPoolAccounted, o.secondPoolCash, "final Pool actual/accounted cash");
        assertGe(o.finalSeniorVaultCash, o.finalSeniorWithdrawalEscrow, "final Senior escrow backed");
        assertGe(o.finalJuniorVaultCash, o.finalJuniorWithdrawalEscrow, "final Junior escrow backed");
    }

    function test_Q261_PreBoundaryOpenClaimBeforeClose() public {
        _publicDeposit(juniorVault, JUNIOR_LP, 55_000e6);
        _publicDeposit(juniorVault, JUNIOR_VICTIM232, 15_000e6);
        _publicDeposit(seniorVault, SENIOR_LP, SENIOR_ACTIVE232);
        assertEq(pool.maxSeniorShareBps(), 8000, "manifest Senior ratio");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "manifest Senior absolute cap");
        _fundTrader(LOSS_TRADER232, 15_000e6);
        vm.warp(block.timestamp + 3601); // B/C active share cooldown before redeem requests.
        // Attacker C makes a small public Junior exit before its main exit.
        // This is completed before both branches' shared snapshot.
        uint256 earlyShares = juniorVault.balanceOf(JUNIOR_LP) / 50;
        vm.prank(JUNIOR_LP);
        uint256 earlyId = juniorVault.requestRedeem(earlyShares, JUNIOR_LP, JUNIOR_LP);
        uint256 earlyStart = pool.lpEpochStart(earlyId);
        if (block.timestamp < earlyStart) vm.warp(earlyStart);
        vm.prank(KEEPER);
        router.settleLpEpoch(_sixFeedUpdateData(ENTRY_PRICE));
        uint256 earlyFunded = juniorVault.claimableRedeemRequest(earlyId, JUNIOR_LP);
        assertEq(earlyFunded, earlyShares, "attacker prior Junior funding");
        vm.prank(JUNIOR_LP);
        uint256 earlyCpaid = juniorVault.claimRedeem(earlyId, earlyFunded, JUNIOR_LP, JUNIOR_LP);
        // Keep a fresh public mark before the next LP requests.
        vm.prank(KEEPER);
        router.updateMarkPrice(_sixFeedUpdateData(ENTRY_PRICE));
        assertEq(pool.reservedSeniorDepositAssets(), 0, "no Senior R at shared prestate");

        uint256 bShares = seniorVault.balanceOf(SENIOR_LP) / 20;
        uint256 cShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 bId = seniorVault.requestRedeem(bShares, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 cId = juniorVault.requestRedeem(cShares, JUNIOR_LP, JUNIOR_LP);
        assertEq(bId, cId, "B/C redemptions same epoch");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no old claim");
        uint256 start = pool.lpEpochStart(bId);
        assertGt(start, block.timestamp + 5, "pre-boundary open window exists");
        vm.warp(start - 5);
        emit log_named_uint("VPI_JIT_OPEN_TIME", block.timestamp);
        _open232();
        assertLt(block.timestamp, start, "trader open executes before LP epoch");
        vm.warp(start);
        uint256 common = vm.snapshotState();

        Outcome232 memory baseline = _branch232(true, bId, cId, bShares, cShares);
        assertTrue(vm.revertToState(common), "restore identical accepted prestate");
        Outcome232 memory attack = _branch232(false, bId, cId, bShares, cShares);
        assertEq(baseline.settleTime, attack.settleTime, "same LP epoch timestamp");
        assertEq(baseline.observedTime, attack.observedTime, "same final observation timestamp");
        assertEq(baseline.secondTime, attack.secondTime, "same second epoch timestamp");
        assertGt(baseline.dSecondFunded, 0, "D actually receives in baseline");
        assertGt(attack.dSecondFunded, 0, "D actually receives in attack");
        emit log_named_uint("VPI_JIT_BASELINE_REBATE", baseline.closeRebatePaid);
        emit log_named_uint("VPI_JIT_ATTACK_REBATE", attack.closeRebatePaid);
        assertGt(attack.closeRebatePaid, 0, "actual same-price rebate paid in both branches");
        assertEq(baseline.cClaimBeforeClose, 0, "baseline C claims after A close");
        assertEq(attack.cClaimBeforeClose, 1, "attack C claims before first A close commit");
        assertEq(baseline.dSecondPending, 0, "baseline D exits in cash");
        assertEq(attack.dSecondPending, 0, "attack D exits in cash");
        assertEq(baseline.cFinalPending, 0, "baseline C exits in cash");
        assertEq(attack.cFinalPending, 0, "attack C exits in cash");
        assertEq(baseline.bPaid + baseline.bSecondPaid, attack.bPaid + attack.bSecondPaid,
            "Senior B payout is unchanged");
        emit log_named_uint("Q232_BASELINE_B_PAID", baseline.bPaid);
        emit log_named_uint("Q232_ATTACK_B_PAID", attack.bPaid);
        emit log_named_uint("Q232_BASELINE_C_PAID", baseline.cPaid);
        emit log_named_uint("Q232_ATTACK_C_PAID", attack.cPaid);
        emit log_named_uint("Q232_BASELINE_C_PENDING", baseline.cPending);
        emit log_named_uint("Q232_ATTACK_C_PENDING", attack.cPending);
        emit log_named_uint("Q232_BASELINE_SENIOR", baseline.seniorPrincipal);
        emit log_named_uint("Q232_ATTACK_SENIOR", attack.seniorPrincipal);
        emit log_named_uint("Q232_BASELINE_HWM", baseline.seniorHwm);
        emit log_named_uint("Q232_ATTACK_HWM", attack.seniorHwm);
        emit log_named_uint("Q232_BASELINE_JUNIOR", baseline.juniorPrincipal);
        emit log_named_uint("Q232_ATTACK_JUNIOR", attack.juniorPrincipal);
        emit log_named_uint("Q232_BASELINE_R", baseline.reservation);
        emit log_named_uint("Q232_ATTACK_R", attack.reservation);
        emit log_named_uint("Q232_TARGET_REACHED", 1);
        emit log_named_uint("IND_BASELINE_SETTLE_TIME", baseline.settleTime);
        emit log_named_uint("IND_ATTACK_SETTLE_TIME", attack.settleTime);
        emit log_named_uint("IND_BASELINE_OBSERVE_TIME", baseline.observedTime);
        emit log_named_uint("IND_ATTACK_OBSERVE_TIME", attack.observedTime);
        emit log_named_uint("IND_BASELINE_LP_RIGHTS", baseline.seniorPrincipal + baseline.juniorPrincipal + baseline.bPaid + baseline.cPaid);
        emit log_named_uint("IND_ATTACK_LP_RIGHTS", attack.seniorPrincipal + attack.juniorPrincipal + attack.bPaid + attack.cPaid);
        emit log_named_uint("IND_BASELINE_PENDING_REVENUE", baseline.pendingRevenue);
        emit log_named_uint("IND_ATTACK_PENDING_REVENUE", attack.pendingRevenue);
        emit log_named_uint("IND_BASELINE_UNASSIGNED", baseline.unassigned);
        emit log_named_uint("IND_ATTACK_UNASSIGNED", attack.unassigned);
        emit log_named_uint("IND_BASELINE_POOL_CASH", baseline.poolCash);
        emit log_named_uint("IND_ATTACK_POOL_CASH", attack.poolCash);
        emit log_named_uint("IND_BASELINE_POOL_ACCOUNTED", baseline.poolAccounted);
        emit log_named_uint("IND_ATTACK_POOL_ACCOUNTED", attack.poolAccounted);
        emit log_named_uint("IND_BASELINE_FREE", baseline.freeAssets);
        emit log_named_uint("IND_ATTACK_FREE", attack.freeAssets);
        emit log_named_uint("IND_BASELINE_A_WALLET", baseline.aWallet);
        emit log_named_uint("IND_ATTACK_A_WALLET", attack.aWallet);
        emit log_named_uint("IND_BASELINE_A_CLEARING", baseline.aClearing);
        emit log_named_uint("IND_ATTACK_A_CLEARING", attack.aClearing);
        emit log_named_uint("IND_BASELINE_B_SECOND_PAID", baseline.bSecondPaid);
        emit log_named_uint("IND_ATTACK_B_SECOND_PAID", attack.bSecondPaid);
        emit log_named_uint("IND_BASELINE_C_SECOND_PAID", baseline.cSecondPaid);
        emit log_named_uint("IND_ATTACK_C_SECOND_PAID", attack.cSecondPaid);
        emit log_named_uint("IND_BASELINE_D_PAID", baseline.dSecondPaid);
        emit log_named_uint("IND_ATTACK_D_PAID", attack.dSecondPaid);
        emit log_named_uint("IND_BASELINE_D_PENDING", baseline.dSecondPending);
        emit log_named_uint("IND_ATTACK_D_PENDING", attack.dSecondPending);
        emit log_named_uint("IND_BASELINE_D_RESIDUAL_QUOTE", baseline.dResidualQuote);
        emit log_named_uint("IND_ATTACK_D_RESIDUAL_QUOTE", attack.dResidualQuote);
        emit log_named_uint("IND_BASELINE_C_FINAL_PENDING", baseline.cFinalPending);
        emit log_named_uint("IND_ATTACK_C_FINAL_PENDING", attack.cFinalPending);
        emit log_named_uint("IND_BASELINE_C_FINAL_QUOTE", baseline.cFinalQuote);
        emit log_named_uint("IND_ATTACK_C_FINAL_QUOTE", attack.cFinalQuote);
        emit log_named_uint("IND_BASELINE_SECOND_JUNIOR", baseline.secondJuniorPrincipal);
        emit log_named_uint("IND_ATTACK_SECOND_JUNIOR", attack.secondJuniorPrincipal);
        emit log_named_uint("IND_BASELINE_SECOND_POOL_CASH", baseline.secondPoolCash);
        emit log_named_uint("IND_ATTACK_SECOND_POOL_CASH", attack.secondPoolCash);
        emit log_named_uint("IND_BASELINE_A_WALLET_FINAL", baseline.aWallet);
        emit log_named_uint("IND_ATTACK_A_WALLET_FINAL", attack.aWallet);
        emit log_named_uint("IND_EARLY_C_PAID", earlyCpaid);
        emit log_named_uint("IND_BASELINE_POOL_REBATE_PAID", baseline.closeRebatePaid);
        emit log_named_uint("IND_ATTACK_POOL_REBATE_PAID", attack.closeRebatePaid);
        emit log_named_uint("IND_BASELINE_ATTACKER_FINAL", earlyCpaid + baseline.aWallet + baseline.cPaid + baseline.cSecondPaid + baseline.cFinalQuote);
        emit log_named_uint("IND_ATTACK_ATTACKER_FINAL", earlyCpaid + attack.aWallet + attack.cPaid + attack.cSecondPaid + attack.cFinalQuote);
        emit log_named_uint("IND_BASELINE_VICTIM_FINAL", baseline.dSecondPaid + baseline.dResidualQuote);
        emit log_named_uint("IND_ATTACK_VICTIM_FINAL", attack.dSecondPaid + attack.dResidualQuote);
        emit log_named_uint("Q261_BASELINE_C_CLAIM_BEFORE_CLOSE", baseline.cClaimBeforeClose);
        emit log_named_uint("Q261_ATTACK_C_CLAIM_BEFORE_CLOSE", attack.cClaimBeforeClose);
        emit log_named_uint("Q261_ATTACK_C_WALLET_BEFORE_CLAIM", attack.cWalletBeforeEarlyClaim);
        emit log_named_uint("Q261_ATTACK_C_WALLET_AT_CLAIM", attack.cWalletAtEarlyClaim);
        emit log_named_uint("Q261_ATTACK_POOL_BEFORE_CLAIM", attack.poolCashBeforeEarlyClaim);
        emit log_named_uint("Q261_ATTACK_POOL_AFTER_CLAIM", attack.poolCashAfterEarlyClaim);
        emit log_named_uint("Q261_ATTACK_NEXT_COMMIT_BEFORE_CLAIM", attack.nextCommitBeforeEarlyClaim);
        emit log_named_uint("Q261_ATTACK_NEXT_COMMIT_AFTER_CLAIM", attack.nextCommitAfterEarlyClaim);
        emit log_named_uint("Q261_ATTACK_A_POSITION_BEFORE_CLAIM", attack.aPositionBeforeEarlyClaim);
        emit log_named_uint("Q261_ATTACK_A_POSITION_AFTER_CLAIM", attack.aPositionAfterEarlyClaim);
        emit log_named_uint("Q261_BASELINE_B_WALLET_FINAL", baseline.bWalletFinal);
        emit log_named_uint("Q261_ATTACK_B_WALLET_FINAL", attack.bWalletFinal);
        emit log_named_uint("Q261_BASELINE_C_WALLET_FINAL", baseline.cWalletFinal);
        emit log_named_uint("Q261_ATTACK_C_WALLET_FINAL", attack.cWalletFinal);
        emit log_named_uint("Q261_BASELINE_D_WALLET_FINAL", baseline.dWalletFinal);
        emit log_named_uint("Q261_ATTACK_D_WALLET_FINAL", attack.dWalletFinal);
        emit log_named_uint("Q261_BASELINE_POOL_ACCOUNTED_FINAL", baseline.finalPoolAccounted);
        emit log_named_uint("Q261_ATTACK_POOL_ACCOUNTED_FINAL", attack.finalPoolAccounted);
        emit log_named_uint("Q261_BASELINE_SENIOR_VAULT_CASH_FINAL", baseline.finalSeniorVaultCash);
        emit log_named_uint("Q261_ATTACK_SENIOR_VAULT_CASH_FINAL", attack.finalSeniorVaultCash);
        emit log_named_uint("Q261_BASELINE_JUNIOR_VAULT_CASH_FINAL", baseline.finalJuniorVaultCash);
        emit log_named_uint("Q261_ATTACK_JUNIOR_VAULT_CASH_FINAL", attack.finalJuniorVaultCash);
        emit log_named_uint("Q261_BASELINE_SENIOR_ESCROW_FINAL", baseline.finalSeniorWithdrawalEscrow);
        emit log_named_uint("Q261_ATTACK_SENIOR_ESCROW_FINAL", attack.finalSeniorWithdrawalEscrow);
        emit log_named_uint("Q261_BASELINE_JUNIOR_ESCROW_FINAL", baseline.finalJuniorWithdrawalEscrow);
        emit log_named_uint("Q261_ATTACK_JUNIOR_ESCROW_FINAL", attack.finalJuniorWithdrawalEscrow);
        emit log_named_uint("Q261_ATTACK_C_PAID_AT_CLAIM", attack.cPaid);
        assertGt(earlyCpaid + attack.aWallet + attack.cPaid + attack.cSecondPaid + attack.cFinalQuote,
            earlyCpaid + baseline.aWallet + baseline.cPaid + baseline.cSecondPaid + baseline.cFinalQuote,
            "attacker gains from moving close after LP exit");
        assertLt(attack.dSecondPaid + attack.dResidualQuote,
            baseline.dSecondPaid + baseline.dResidualQuote,
            "independent D receives less in attack than baseline");
        assertGt(earlyCpaid + attack.aWallet + attack.cPaid + attack.cSecondPaid + attack.cFinalQuote,
            70_000e6, "A plus C end above their initial USDG principal before execution costs");
        assertLt(attack.dSecondPaid + attack.dResidualQuote,
            15_000e6, "unrelated Junior D ends below its initial USDG principal at unchanged price");
    }

    struct PassiveResult {
        uint256 attacker;
        uint256 victim;
        uint256 senior;
        uint256 poolCash;
        uint256 supply;
        uint256 finalTime;
    }

    function _redepositCWallet() private {
        uint256 assets = settlementToken.balanceOf(JUNIOR_LP);
        assertGt(assets, 0, "recycle C's actual wallet USDG");
        vm.startPrank(JUNIOR_LP);
        settlementToken.approve(address(juniorVault), assets);
        uint256 id = juniorVault.requestDeposit(assets, JUNIOR_LP);
        vm.stopPrank();
        uint256 start = pool.lpEpochStart(id);
        if (block.timestamp < start) vm.warp(start);
        vm.prank(KEEPER);
        router.settleLpEpoch(_sixFeedUpdateData(ENTRY_PRICE));
        uint256 shares = juniorVault.claimableDepositShares(id, JUNIOR_LP);
        assertGt(shares, 0, "C recycled shares activate");
        vm.prank(JUNIOR_LP);
        assertEq(juniorVault.claimDeposit(id, assets, JUNIOR_LP, JUNIOR_LP),
            shares, "C recycled shares delivered");
    }

    function _redepositAWallet() private {
        uint256 assets = settlementToken.balanceOf(LOSS_TRADER232);
        assertGt(assets, LOSS_MARGIN232, "A has next-cycle margin");
        vm.startPrank(LOSS_TRADER232);
        settlementToken.approve(address(clearinghouse), assets);
        clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    function _passiveCycle(uint256 cycle, bool closeBeforeEpoch) private {
        uint256 ready = block.timestamp + 3601;
        uint256 cReady = juniorVault.lastDepositTime(JUNIOR_LP) + 3601;
        uint256 bReady = seniorVault.lastDepositTime(SENIOR_LP) + 3601;
        if (ready < cReady) ready = cReady;
        if (ready < bReady) ready = bReady;
        vm.warp(ready);
        uint256 earlyShares = juniorVault.balanceOf(JUNIOR_LP) / 50;
        assertGt(earlyShares, 0, "C early shares");
        vm.prank(JUNIOR_LP);
        uint256 earlyId = juniorVault.requestRedeem(earlyShares, JUNIOR_LP, JUNIOR_LP);
        uint256 earlyStart = pool.lpEpochStart(earlyId);
        if (block.timestamp < earlyStart) vm.warp(earlyStart);
        vm.prank(KEEPER);
        router.settleLpEpoch(_sixFeedUpdateData(ENTRY_PRICE));
        uint256 earlyFunded = juniorVault.claimableRedeemRequest(earlyId, JUNIOR_LP);
        assertEq(earlyFunded, earlyShares, "C early exit funded");
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(earlyId, earlyFunded, JUNIOR_LP, JUNIOR_LP);
        vm.prank(KEEPER);
        router.updateMarkPrice(_sixFeedUpdateData(ENTRY_PRICE));

        uint256 bShares = seniorVault.balanceOf(SENIOR_LP) / 20;
        uint256 cShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 bId = seniorVault.requestRedeem(bShares, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 cId = juniorVault.requestRedeem(cShares, JUNIOR_LP, JUNIOR_LP);
        assertEq(bId, cId, "B/C same funding epoch");
        uint256 start = pool.lpEpochStart(bId);
        assertGt(start, block.timestamp + 5, "pre-boundary open window");
        vm.warp(start - 5);
        _open232();
        vm.warp(start);

        vm.prank(KEEPER);
        router.updateMarkPrice(_sixFeedUpdateData(ENTRY_PRICE));
        if (closeBeforeEpoch) {
            _close232();
        } else {
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 1);
        }
        vm.prank(KEEPER);
        router.settleLpEpoch(_sixFeedUpdateData(ENTRY_PRICE));
        uint256 bFunded = seniorVault.claimableRedeemRequest(bId, SENIOR_LP);
        uint256 cFunded = juniorVault.claimableRedeemRequest(cId, JUNIOR_LP);
        assertEq(bFunded, bShares, "B full first-epoch exit");
        assertEq(cFunded, cShares, "C full first-epoch exit");
        if (!closeBeforeEpoch) {
            _close232();
        } else {
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 1);
        }
        // Both branches claim after the close: only the funding price differs.
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(bId, bFunded, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(cId, cFunded, JUNIOR_LP, JUNIOR_LP);
        uint256 freeA = clearinghouse.balanceAssets(LOSS_TRADER232);
        vm.prank(LOSS_TRADER232);
        clearinghouse.withdrawMargin(freeA);
        assertEq(clearinghouse.balanceAssets(LOSS_TRADER232), 0, "A wallet exit");
        assertEq(juniorVault.pendingRedeemRequest(cId, JUNIOR_LP), 0, "C exit complete");
        assertGt(juniorVault.balanceOf(JUNIOR_VICTIM232), 0,
            "D stays invested throughout the cycle");
        emit log_named_uint("PASSIVE_BRANCH_CLOSE_BEFORE", closeBeforeEpoch ? 1 : 0);
        emit log_named_uint("PASSIVE_CYCLE", cycle);
        emit log_named_uint("PASSIVE_ATTACKER_WALLET", settlementToken.balanceOf(LOSS_TRADER232)
            + settlementToken.balanceOf(JUNIOR_LP));
        emit log_named_uint("PASSIVE_VICTIM_SHARES", juniorVault.balanceOf(JUNIOR_VICTIM232));
        emit log_named_uint("PASSIVE_VICTIM_NAV", juniorVault.convertToAssets(
            juniorVault.balanceOf(JUNIOR_VICTIM232)));
        emit log_named_uint("PASSIVE_VICTIM_WALLET", settlementToken.balanceOf(JUNIOR_VICTIM232));
        emit log_named_uint("PASSIVE_TOTAL_SUPPLY", settlementToken.totalSupply());
    }

    function _runPassiveMany(bool closeBeforeEpoch, uint256 count, uint256 cInitial) private returns (PassiveResult memory result) {
        _publicDeposit(juniorVault, JUNIOR_LP, cInitial);
        _publicDeposit(juniorVault, JUNIOR_VICTIM232, 70_000e6 - cInitial);
        _publicDeposit(seniorVault, SENIOR_LP, SENIOR_ACTIVE232);
        _fundTrader(LOSS_TRADER232, 15_000e6);
        uint256 initialSupply = settlementToken.totalSupply();
        uint256 victimShares = juniorVault.balanceOf(JUNIOR_VICTIM232);
        for (uint256 cycle = 1; cycle <= count; ++cycle) {
            if (cycle > 1) {
                _redepositCWallet();
                _redepositAWallet();
            }
            _passiveCycle(cycle, closeBeforeEpoch);
            assertEq(juniorVault.balanceOf(JUNIOR_VICTIM232), victimShares,
                "D made no request or transfer before final redemption");
            assertEq(settlementToken.totalSupply(), initialSupply,
                "no additional tokens minted after initial setup");
        }

        uint256 bShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 bId = seniorVault.requestRedeem(bShares, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_VICTIM232);
        uint256 dId = juniorVault.requestRedeem(victimShares,
            JUNIOR_VICTIM232, JUNIOR_VICTIM232);
        assertEq(bId, dId, "B and passive D final exit same epoch");
        uint256 start = pool.lpEpochStart(dId);
        if (block.timestamp < start) vm.warp(start);
        vm.prank(KEEPER);
        router.settleLpEpoch(_sixFeedUpdateData(ENTRY_PRICE));
        uint256 bFunded = seniorVault.claimableRedeemRequest(bId, SENIOR_LP);
        uint256 dFunded = juniorVault.claimableRedeemRequest(dId, JUNIOR_VICTIM232);
        assertEq(bFunded, bShares, "B final full exit");
        assertEq(dFunded, victimShares, "passive D final full exit");
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(bId, bFunded, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_VICTIM232);
        juniorVault.claimRedeem(dId, dFunded, JUNIOR_VICTIM232, JUNIOR_VICTIM232);
        result.attacker = settlementToken.balanceOf(LOSS_TRADER232)
            + settlementToken.balanceOf(JUNIOR_LP);
        result.victim = settlementToken.balanceOf(JUNIOR_VICTIM232);
        result.senior = settlementToken.balanceOf(SENIOR_LP);
        result.poolCash = settlementToken.balanceOf(address(pool));
        result.supply = settlementToken.totalSupply();
        result.finalTime = block.timestamp;
        assertEq(result.supply, initialSupply, "final supply unchanged");
    }


    function _assertCase(uint256 count, uint256 cInitial) private {
        uint256 initial = vm.snapshotState();
        PassiveResult memory baseline = _runPassiveMany(true, count, cInitial);
        assertTrue(vm.revertToState(initial), "same pinned original prestate");
        PassiveResult memory attack = _runPassiveMany(false, count, cInitial);
        assertEq(baseline.supply, attack.supply, "equal token supply");
        assertEq(baseline.finalTime, attack.finalTime, "equal final timestamp");
        assertEq(baseline.senior, attack.senior, "unchanged Senior final cash");
        assertGt(attack.attacker, baseline.attacker, "combined Trader/LP receives more than control");
        assertLt(attack.victim, baseline.victim, "passive LP receives less than control");
        assertGt(baseline.poolCash, attack.poolCash, "Pool residual decreases by the balancing amount");
        assertEq(
            attack.attacker - baseline.attacker,
            baseline.victim - attack.victim + baseline.poolCash - attack.poolCash,
            "relative USDG transfer reconciles atom-for-atom"
        );
        emit log_named_uint("MATRIX_CYCLES", count);
        emit log_named_uint("MATRIX_ATTACKER_JUNIOR_INITIAL", cInitial);
        emit log_named_uint("MATRIX_BASELINE_ATTACKER", baseline.attacker);
        emit log_named_uint("MATRIX_ATTACK_ATTACKER", attack.attacker);
        emit log_named_uint("MATRIX_BASELINE_VICTIM", baseline.victim);
        emit log_named_uint("MATRIX_ATTACK_VICTIM", attack.victim);
        emit log_named_uint("MATRIX_BASELINE_POOL", baseline.poolCash);
        emit log_named_uint("MATRIX_ATTACK_POOL", attack.poolCash);
    }
    function test_Q261_Matrix_C1_S25() public { _assertCase(1, 17500e6); }
    function test_Q261_Matrix_C1_S50() public { _assertCase(1, 35000e6); }
    function test_Q261_Matrix_C1_S75() public { _assertCase(1, 52500e6); }
    function test_Q261_Matrix_C2_S25() public { _assertCase(2, 17500e6); }
    function test_Q261_Matrix_C2_S50() public { _assertCase(2, 35000e6); }
    function test_Q261_Matrix_C2_S75() public { _assertCase(2, 52500e6); }
    function test_Q261_Matrix_C3_S25() public { _assertCase(3, 17500e6); }
    function test_Q261_Matrix_C3_S50() public { _assertCase(3, 35000e6); }
    function test_Q261_Matrix_C3_S75() public { _assertCase(3, 52500e6); }
    function test_Q261_Matrix_C5_S25() public { _assertCase(5, 17500e6); }
    function test_Q261_Matrix_C5_S50() public { _assertCase(5, 35000e6); }
    function test_Q261_Matrix_C5_S75() public { _assertCase(5, 52500e6); }
}
```

### Expected observations

- `test_Q261_PreBoundaryOpenClaimBeforeClose`: `IND_BASELINE_ATTACKER_FINAL = 69988622669`, `IND_ATTACK_ATTACKER_FINAL = 70006681289`, `IND_BASELINE_VICTIM_FINAL = 15001816420`, `IND_ATTACK_VICTIM_FINAL = 14983759001` (6-decimal USDG atoms). The test asserts the open occurs before the epoch, the victim's redemption is fully funded, the attacker receives its funded LP cash before its first close commitment, Senior payout is unchanged, and the final wallets have the asserted relationship.
- `test_Q261_Matrix_C5_S75`: `MATRIX_BASELINE_ATTACKER = 67441632655`, `MATRIX_ATTACK_ATTACKER = 67529208666`, `MATRIX_BASELINE_VICTIM = 17510734313`, `MATRIX_ATTACK_VICTIM = 17423163296`, `MATRIX_BASELINE_POOL = 2000815`, `MATRIX_ATTACK_POOL = 1995821`. The final victim loss relative to control is `87.571017 USDG`, the attacker branch gain is `87.576011 USDG`, and the Pool residual change is `−0.004994 USDG`; the three deltas sum to zero atom-for-atom.
- Within each repeated branch, the test redeposits only USDG actually claimed to the attacker wallet, crosses the normal cooldown and epoch boundaries, does not reset to a snapshot between cycles, holds the passive victim's Junior shares throughout, and confirms token supply is unchanged after initial setup. The baseline and attack branches each start from the same snapshot and end at the same timestamp. Each of the 12 matrix tests asserts the direction of the attacker, passive LP and Pool deltas and their exact USDG-atom reconciliation.

The 6-feed data are local Pyth mocks with zero confidence. These tests demonstrate original-contract reachability and USDG cash effects; they do not establish the frequency of identical valid signed market prices or net profitability after external gas, oracle and capital costs.
