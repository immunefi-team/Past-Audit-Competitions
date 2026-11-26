# \#98371 [SC-Critical] Early Junior LP redemption captures refundable VPI charges and shifts rebate losses to incoming LPs

**Submitted on Nov 7th 2026 at 12:46:44 UTC by @Jornason for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98371
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, a positive virtual price impact (VPI) charge is credited immediately as LP revenue, while the originating position can later receive a bounded negative VPI rebate funded by the pool. Junior LP entry and exit pricing does not reserve that refundable charge for the LP cohort that received it. An ordinary trader who also owns Junior LP shares can open, redeem those shares before closing, and then receive the close rebate while later LPs bear the corresponding loss. In a same-price, equal-authenticated-depth local control, the early-exit actor finishes with 100.007091 test USDG more than its own 1,300,000 test-USDG funding, and an independently funded replacement LP suffers an actual 551.251249 test-USDG principal shortfall. This is a conditional direct-theft-of-user-funds candidate, not a claim of production losses, protocol insolvency, or guaranteed public-network profit.

## Vulnerability Details

### Affected code and security property

The affected implementation is in the non-view portion of `packages/perps/src`, principally:

- `CfdEngineSettlementSidecar.sol`: positive opening trade cost is recorded as LP revenue at lines 454-463, and a close action rebate is paid from the pool at lines 621-623.
- `HousePool.sol`: `recordClaimantInflow` adds received revenue to accounted assets at lines 624-642; `_settleLpEpoch` prices and funds Junior redemptions at lines 835-843 and 879-887.
- `TrancheVault.sol`: public `requestRedeem` and `claimRedeem` realize LP value at lines 606-627 and 1014-1020.
- `libraries/CloseAccountingLib.sol`: `buildCloseState` limits a negative close VPI adjustment by the position's positive lifetime accrual at lines 103-117.
- `libraries/CfdEnginePlanLib.sol`: close rebates are funded from free pool cash at lines 973-977, and close VPI uses pool assets net of trader claims at lines 1339-1341.

The security property at issue is ownership of a contingent VPI refund across LP cohorts. The lifetime clamp prevents a trader's lifetime VPI from becoming net-negative, but it does not assign the already collected positive charge to the LP cohort that will fund a later rebate. Once opening revenue is redeemable, an exiting LP can keep its share of that revenue while the remaining or incoming LPs pay the subsequent rebate.

### State transition and root cause

The opening settlement records positive trade cost as pool revenue:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

The close can then pay a negative VPI adjustment from pool cash. The lifetime clamp remains active:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A public LP redemption can occur between those two events. The redemption includes the opening revenue in the exiting LP's share value, while the close rebate reduces the pool value left to other LPs. No position-specific contingent reserve links the refundable charge to its originating position.

The resulting sequence is:

```text
ordinary trader also owns Junior LP shares
  -> public skew-increasing open pays positive VPI to the pool
  -> LP redemption prices and pays the actor's share of that revenue
  -> the same position receives a later pool-funded close rebate
  -> remaining or incoming LPs bear the rebate without having received the exited revenue
```

### Attacker, victim, and necessary conditions

The actor is an ordinary user with its own eligible Junior shares and separately funded trader margin. It needs no administrator, keeper, oracle-control, storage-write, or production token-minting privilege. It uses public order and LP paths and must have sufficient capital and a valid LP redemption epoch. A positive opening VPI and a subsequently paid close rebate are required.

The ordinary route uses a pre-existing independent LP and measures its actual redemption proceeds. The controlled equal-depth variant additionally requires an independently funded replacement LP whose deposit activates in the same LP settlement as the actor's redemption. This restores post-open pool depth without an intervening reduced-depth epoch. The actor cannot force that independent LP's funding, amount, or timing; it is an explicit precondition of the controlled comparison, not an assumption hidden behind the result.

The independent deposit amount equals the actor's publicly simulated redemption proceeds. The test obtains the quote through public calls in a disposable snapshot and reverts before executing the measured branch. This models off-chain transaction simulation; it does not modify storage, grant a role, or create production assets.

### Why this is distinct from the published depth sandwich

Published `AI-SCAN-31`, “Temporary pool-liquidity reduction can amplify a skew-reducing close rebate,” concerns execution-time pool depth increasing a negative VPI quote. The new same-epoch control removes that difference between the early-exit and hold branches. Both branches execute at timestamp `1,709,546,402` with authenticated pool depth `2,003,405.714513`, zero trader claims, close VPI `-1,403.859427`, carry `0.869600`, execution fee `300`, and a fully paid rebate of `1,103.859427`.

The early actor nevertheless finishes at `1,300,100.007091` versus `1,299,548.737595` when holding its LP shares, a `551.269496` advantage. The replacement LP contributes `1,000,702.062418` and subsequently claims `1,000,150.811169`, an actual principal shortfall of `551.251249`. Thus a reduced close-depth quote is not necessary for the remaining cross-cohort transfer. Normalizing the VPI denominator alone does not repair ownership of the refundable opening revenue.

The protocol documentation deliberately leaves VPI outside the pre-close marked-price LP book and describes LP-funded VPI rebates. This report does not misrepresent those design statements as an absent clamp. It asks the program to assess whether the measured transfer of principal between LP cohorts is an unintended theft or an accepted consequence of that design. The clamp is active throughout the proof, and no rebate waiver is used to manufacture the loss.


### Recommended remediation

Keep the refundable part of a positive opening VPI charge outside immediately redeemable LP principal while the originating position can still reclaim it. A position-specific contingent reserve could receive the opening charge, fund later rebates, and release only the nonrefundable residual to LP revenue when the exposure ends. LP entry and exit NAV should consistently account for that reserve so neither an exiting LP nor a new depositor transfers the obligation to another cohort.

The implementation must preserve ordinary deposit and redemption liveness and define partial closes, increases, liquidation, waived rebates, and insufficient pool cash. Regression tests should compare early-exit and hold branches under identical authenticated close inputs, then check reserve conservation and actual LP cash claims. A close-depth-only fix is insufficient for this ownership issue.

## Impact Details

The selected impact is **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**, which the competition maps to **Critical**. The claimed impact is conditional loss of user principal, not merely unpaid yield.

| Measurement | Observed local result | Interpretation and limit |
| --- | ---: | --- |
| Early-exit actor actual cash | `1,300,100.007091` test USDG | `100.007091` above the actor's own `1,300,000` funding |
| Hold-branch actor actual cash | `1,299,548.737595` test USDG | Same post-funding snapshot and same close inputs |
| Early-versus-hold actor advantage | `551.269496` test USDG | Counterfactual advantage from the LP timing choice |
| Replacement LP contribution | `1,000,702.062418` test USDG | Independently funded through public LP request and activation |
| Replacement LP actual redemption | `1,000,150.811169` test USDG | Actual token transfer, not only `convertToAssets` |
| Replacement LP principal shortfall | `551.251249` test USDG | Contribution minus actual redemption; requires the stated same-epoch participant |
| Original incumbent principal shortfall | `596.068500` test USDG | Separate ordinary same-price route: `1,000,000 - 999,403.931500`; its hold branch claims `1,000,160.161384` |

The replacement-LP principal loss and its difference from the actor's early-versus-hold advantage are not additional independent losses to be summed. The separate original-incumbent result is supporting evidence from a different route and must not be added to the controlled variant's loss either. The original incumbent's cash outcomes in the controlled variant differ by one share atom in the recorded run, so that variant identifies the replacement LP as the marginal loss bearer.

The original ordinary route and the new control both execute real public calls and actual local token transfers. The new control additionally withdraws the actor's clearinghouse cash and fully redeems the actor, incumbent, and replacement LP through public entry points. This establishes the measured principal effect without relying on preview values, a storage override, or a fabricated claim.

No protocol insolvency, permanent freeze, unclaimed-yield theft, or universal LP loss is asserted. The observed local amounts do not establish a maximum extraction rate, repeatable real-world yield, production liquidity, or existing mainnet victim.

### Validation limits

The primary package runs the fixed scoped source locally with the official test risk parameters and mock out-of-scope Pyth and USDG dependencies. It uses no RPC, public-chain transaction, administrator attack step, or modification of scoped source. Minting test USDG is limited to local fixture funding. The fixed source commit and source tree are checked by the supplied runner.

The extended research record also contains signed historical-fork evidence, but the package's direct theft proof does not depend on replaying that fork. Production token value, public order inclusion, available independent LP participation, network fees, financing costs, and public-network profitability remain unmeasured. Platform acceptance, final severity, and private duplicate status remain unknown.

## References

- Competition scope: https://immunefi.com/audit-competition/plether/scope/#top
- Fixed source tree: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- Opening revenue settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463
- Close rebate settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L623
- LP epoch settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L887
- Close accounting and lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L117
- Published security notes: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L849-L872
- Published AI scan: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md


## Link to Proof of Concept

https://gist.github.com/Jornason/81e1b9a5517382b12aacd2f8ddf71b4c

## Proof of Concept

## Proof of Concept

### Overview

This PoC runs the fixed scoped source locally and compares an actor who redeems Junior LP shares between a positive opening VPI charge and a later close rebate with the same actor holding those shares through the close. The added same-epoch control requires identical authenticated close depth, timestamp, VPI, carry, execution fees, and paid rebates across the two branches. It then measures actual token withdrawals and LP redemptions rather than relying only on previews or share valuation.

The complete control source is included below and is also supplied as `reviewer-poc/Find010EqualDepth.t.sol`. Its SHA-256 is `0d6b9842f652ecdcc3c7fd76f7209be8ed3d4f4e8691db90ac32326d45724d2d`.

The companion original suite, `reviewer-poc/M04VpiLpTemporalCapture.t.sol`, supplies five same-price tests, including the ordinary incumbent's actual cash loss. Its SHA-256 is `0d32994bb071ead3747ff96370b565dd18691f44c57558d01e763dc7ceecf89b`.

### Step-by-step explanation

1. Prepare the official repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with its pinned submodules. Do not change `packages/perps/src`. The runner verifies that commit, the scoped source tree `fff196b1b2a5bf6e9bfdc94fdb123c7854fe1252`, scoped-source cleanliness, and both test hashes.

2. The fixture deploys the fixed source with VPI factor 1%, directional skew cap 40%, maintenance margin 10 bps, initial margin 20 bps, FAD margin 300 bps, base carry 500 bps, and execution fee 4 bps. MockPyth and mock USDG replace only out-of-scope dependencies. Test-token minting is used only to establish local fixture balances.

3. An independent incumbent Junior LP is already present. The actor publicly funds and activates `1,000,000` test USDG of Junior shares, waits through the deposit cooldown, and separately funds `300,000` test USDG of trader margin. Both measured branches start from the same post-funding snapshot.

4. The actor publicly commits and executes a `750,000`-unit LONG with `200,000` test USDG of margin at the same mock basket price. The authenticated open receipt records positive VPI `1,404.845154`, execution fee `300`, and pre-open pool depth `2,002,000`. This produces the refundable opening charge at issue.

5. In the early-exit branch, the actor requests redemption of all eligible Junior shares. A disposable public-call simulation determines the redemption proceeds used for the independent replacement LP's requested deposit. The snapshot is reverted before the measured branch; no storage or protocol role is changed.

6. The independent replacement LP requests a deposit of `1,000,702.062418` test USDG. Its deposit activates in the same LP settlement as the actor's redemption, avoiding a reduced-depth interval between LP epochs. The actor receives the same `1,000,702.062418` in actual test tokens. This independent participant and same-epoch timing are explicit preconditions; the actor cannot force them.

7. The hold branch restores the common post-funding snapshot and keeps the actor's shares without the replacement deposit. Both branches then execute the public full close at timestamp `1,709,546,402`. Authenticated receipts and runtime state require identical pool depth `2,003,405.714513`, zero trader claims, close VPI `-1,403.859427`, carry `0.869600`, execution fee `300`, and fully paid rebate `1,103.859427`. This rules out a different close-depth quote as the explanation for the A/B value transfer.

8. The actor withdraws its free clearinghouse balance. The actor, original incumbent, and replacement LP then use public redemption requests, epoch settlement, and claims to measure actual token proceeds. The actor finishes at `1,300,100.007091` in the early-exit branch versus `1,299,548.737595` in the hold branch, an advantage of `551.269496`. The early actor also receives `100.007091` more than its own `1,300,000` funding.

9. The independently funded replacement LP contributed `1,000,702.062418` and receives `1,000,150.811169` after its public redemption, an actual principal shortfall of `551.251249`. The original incumbent's two cash outcomes differ by one share atom in this controlled run, so the replacement LP is identified as the marginal loss bearer in this variant.

10. The runner also executes the original five-test suite. Its high-capital same-price route independently shows the original incumbent claiming `999,403.931500`, which is `596.068500` below its own `1,000,000` principal, versus `1,000,160.161384` in the hold branch. That is supporting evidence from a different route and is not added to the controlled variant's loss.

### Reproduction

From a clean official checkout:

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Use the supplied runner from the directory containing `run-poc.sh`:

```bash
PLETHER_REPO_ROOT=/path/to/audit-comp-plether ./run-poc.sh
```

The runner executes the following commands from `packages/perps`, copying only missing tests and removing only files it created:

```bash
forge test --offline --threads 1 \
  --match-path test/perps/accounting/fees/M04VpiLpTemporalCapture.t.sol -vv

forge test --offline --threads 1 \
  --match-path test/perps/accounting/fees/Find010EqualDepth.t.sol \
  --match-test test_Find010EqualDepthEarlyExitVersusHold -vv
```

No RPC, public-chain transaction, private key, online oracle request, or modification of scoped source is required. The expected aggregate result is five original tests and one same-epoch control passing with zero failures and zero skips.

### Expected result

The following is the complete test-result and application-log section for the added same-epoch control. Compiler warnings are omitted; no test result or application log line is omitted. `original-poc-output.txt` supplies the equivalent original five-test section.

```text
Ran 1 test for test/perps/accounting/fees/Find010EqualDepth.t.sol:Find010EqualDepthTest
[PASS] test_Find010EqualDepthEarlyExitVersusHold() (gas: 18379816)
Logs:
  early actor cash 1300100007091
  hold actor cash 1299548737595
  early incumbent cash 1000150820294
  hold incumbent cash 1000150820293
  replacement cost 1000702062418
  replacement cash 1000150811169
  early incumbent NAV at close 1001150975685
  hold incumbent NAV at close 1001150975684
  replacement NAV at close 1000150843108
  early open pool depth 2002000000000
  early close pool depth 2003405714513
  hold close pool depth 2003405714513
  early close net depth 2003405714513
  hold close net depth 2003405714513
  early close timestamp 1709546402
  hold close timestamp 1709546402
  early open VPI 1404845154
  early close VPI -1403859427
  hold close VPI -1403859427
  early carry 869600
  hold carry 869600
  early rebate paid 1103859427
  hold rebate paid 1103859427
  early rebate waived 0
  early close fee 300000000
  hold close fee 300000000
  equal-depth actor advantage 551269496
  replacement principal loss 551251249

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 38.85ms (23.84ms CPU time)

Ran 1 test suite in 43.88ms (38.85ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Complete Solidity source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";
import {BasePerpTest} from "test/perps/BasePerpTest.sol";

contract Find010EqualDepthTest is BasePerpTest {

    address internal constant ACTOR = address(0xA11CE);
    address internal constant REPLACEMENT_LP = address(0xB0B);
    uint256 internal constant LP_DEPOSIT = 1_000_000e6;
    uint256 internal constant TRADER_DEPOSIT = 300_000e6;
    uint256 internal constant LOTS = 750_000e18;
    bytes32 internal constant REBATE_SETTLED = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");

    struct Outcome {
        uint256 actorCash;
        uint256 incumbentCash;
        uint256 replacementCash;
        uint256 replacementCost;
        uint256 incumbentNavAtClose;
        uint256 replacementNavAtClose;
        uint256 openPoolDepth;
        uint256 closePoolDepth;
        uint256 closeClaims;
        uint256 closeNetDepth;
        uint256 closeTimestamp;
        uint256 closeExecutionPrice;
        uint256 rebateAssessed;
        uint256 rebatePaid;
        uint256 rebateWaived;
        int256 openVpi;
        int256 closeVpi;
        int256 pricePnl;
        int256 carry;
        uint256 openFee;
        uint256 closeFee;
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

    function test_Find010EqualDepthEarlyExitVersusHold() public {
        _fundJunior(ACTOR, LP_DEPOSIT);
        vm.warp(juniorVault.lastDepositTime(ACTOR) + juniorVault.DEPOSIT_COOLDOWN());
        _fundTrader(ACTOR, TRADER_DEPOSIT);

        uint256 clean = vm.snapshotState();
        Outcome memory early = _runCohortBranch(true);
        assertTrue(vm.revertToState(clean));
        Outcome memory hold = _runCohortBranch(false);

        console2.log("early actor cash", early.actorCash);
        console2.log("hold actor cash", hold.actorCash);
        console2.log("early incumbent cash", early.incumbentCash);
        console2.log("hold incumbent cash", hold.incumbentCash);
        console2.log("replacement cost", early.replacementCost);
        console2.log("replacement cash", early.replacementCash);
        console2.log("early incumbent NAV at close", early.incumbentNavAtClose);
        console2.log("hold incumbent NAV at close", hold.incumbentNavAtClose);
        console2.log("replacement NAV at close", early.replacementNavAtClose);
        console2.log("early open pool depth", early.openPoolDepth);
        console2.log("early close pool depth", early.closePoolDepth);
        console2.log("hold close pool depth", hold.closePoolDepth);
        console2.log("early close net depth", early.closeNetDepth);
        console2.log("hold close net depth", hold.closeNetDepth);
        console2.log("early close timestamp", early.closeTimestamp);
        console2.log("hold close timestamp", hold.closeTimestamp);
        console2.log("early open VPI", early.openVpi);
        console2.log("early close VPI", early.closeVpi);
        console2.log("hold close VPI", hold.closeVpi);
        console2.log("early carry", early.carry);
        console2.log("hold carry", hold.carry);
        console2.log("early rebate paid", early.rebatePaid);
        console2.log("hold rebate paid", hold.rebatePaid);
        console2.log("early rebate waived", early.rebateWaived);
        console2.log("early close fee", early.closeFee);
        console2.log("hold close fee", hold.closeFee);

        assertEq(early.openPoolDepth, 2_002_000e6);
        assertEq(early.openPoolDepth, hold.openPoolDepth);
        assertGt(early.openVpi, 0);
        assertEq(early.openVpi, hold.openVpi);
        assertEq(early.openFee, hold.openFee);
        assertEq(early.closePoolDepth, hold.closePoolDepth, "same authenticated close pool input");
        assertEq(early.closeClaims, 0);
        assertEq(hold.closeClaims, 0);
        assertEq(early.closeNetDepth, hold.closeNetDepth, "same close VPI economic depth");
        assertEq(early.closeTimestamp, hold.closeTimestamp, "same close wall clock");
        assertEq(early.closeExecutionPrice, 1e8);
        assertEq(early.closeExecutionPrice, hold.closeExecutionPrice);
        assertEq(early.pricePnl, 0);
        assertEq(early.pricePnl, hold.pricePnl);
        assertEq(early.closeVpi, hold.closeVpi, "depth-normalized VPI quote must match");
        assertEq(early.carry, hold.carry, "carry must not explain the difference");
        assertEq(early.closeFee, hold.closeFee);
        assertEq(early.rebatePaid, hold.rebatePaid, "same paid rebate in both cohorts");
        assertEq(early.rebatePaid, early.rebateAssessed);
        assertEq(early.rebateWaived, 0);
        assertGt(early.rebatePaid, 0);
        assertLe(early.rebatePaid, uint256(early.openVpi));
        assertEq(juniorVault.balanceOf(ACTOR), 0);
        assertEq(juniorVault.balanceOf(REPLACEMENT_LP), 0);
        assertEq(juniorVault.maxRequestRedeem(address(this)), 0);
        assertEq(hold.actorCash, settlementToken.balanceOf(ACTOR));
        console2.log("equal-depth actor advantage", early.actorCash - hold.actorCash);
        console2.log("replacement principal loss", early.replacementCost - early.replacementCash);

        assertGt(early.actorCash, LP_DEPOSIT + TRADER_DEPOSIT, "early exit actor profit");
        assertGt(early.actorCash, hold.actorCash, "early exit beats holding at equal close depth");
        assertLt(early.replacementCash, early.replacementCost, "replacement LP loses principal");
        assertEq(hold.replacementCost, 0);
        assertEq(hold.replacementCash, 0);
    }

    function _runCohortBranch(
        bool actorExitsFirst
    ) internal returns (Outcome memory outcome) {
        _startRecordingLogs();
        vm.prank(ACTOR);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, LOTS, 200_000e6, 1e8, false);
        router.executeOrder(openId, _mockPythUpdateData(1e8));
        OrderV3Types.OrderReceipt memory openReceipt = _verifiedReceipt(router.lifecycleBook(), openId);
        outcome.openPoolDepth = openReceipt.poolDepthAssets;
        outcome.openVpi = openReceipt.economics.vpiAssets;
        outcome.openFee = openReceipt.economics.executionFeeAssets;
        uint256 postOpenPoolDepth = pool.totalAssets();
        uint256 redemptionTime = (block.timestamp / pool.LP_EPOCH_DURATION() + 1) * pool.LP_EPOCH_DURATION();

        if (actorExitsFirst) {
            uint256 beforeQuote = vm.snapshotState();
            outcome.replacementCost = _quoteActorExit(redemptionTime);
            assertTrue(vm.revertToState(beforeQuote));
            _startRecordingLogs();
            uint256 depositId = _requestReplacementDeposit(outcome.replacementCost, redemptionTime);
            uint256 shares = juniorVault.maxRequestRedeem(ACTOR);
            uint256 walletBefore = settlementToken.balanceOf(ACTOR);
            vm.prank(ACTOR);
            uint256 requestId = juniorVault.requestRedeem(shares, ACTOR, ACTOR);
            _settlePublicAt(redemptionTime);
            uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ACTOR);
            assertEq(claimable, shares);
            vm.prank(ACTOR);
            juniorVault.claimRedeem(requestId, claimable, ACTOR, ACTOR);
            assertEq(settlementToken.balanceOf(ACTOR) - walletBefore, outcome.replacementCost);
            uint256 claimableDeposit = juniorVault.claimableDepositRequest(depositId, REPLACEMENT_LP);
            assertEq(claimableDeposit, outcome.replacementCost);
            vm.prank(REPLACEMENT_LP);
            juniorVault.claimDeposit(depositId, claimableDeposit, REPLACEMENT_LP, REPLACEMENT_LP);
            assertEq(juniorVault.balanceOf(ACTOR), 0);
        } else {
            vm.warp(redemptionTime);
            router.updateMarkPrice(_mockPythUpdateData(1e8));
        }
        assertEq(pool.totalAssets(), postOpenPoolDepth, "same-epoch restoration must preserve post-open depth");

        outcome.closeClaims = engine.totalTraderClaimBalanceAssets();
        vm.recordLogs();
        vm.prank(ACTOR);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, LOTS, 0, 1e8, true);
        router.executeOrder(closeId, _mockPythUpdateData(1e8));
        Vm.Log[] memory logs = _takeRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length != 0 && logs[i].topics[0] == REBATE_SETTLED) {
                (outcome.rebateAssessed, outcome.rebatePaid, outcome.rebateWaived) =
                    abi.decode(logs[i].data, (uint256, uint256, uint256));
            }
        }
        OrderV3Types.OrderReceipt memory closeReceipt = _verifiedReceipt(router.lifecycleBook(), closeId);
        outcome.closePoolDepth = closeReceipt.poolDepthAssets;
        outcome.closeNetDepth = closeReceipt.poolDepthAssets - outcome.closeClaims;
        outcome.closeTimestamp = block.timestamp;
        outcome.closeExecutionPrice = closeReceipt.executionPrice;
        outcome.closeVpi = closeReceipt.economics.vpiAssets;
        outcome.pricePnl = closeReceipt.economics.realizedPnlAssets;
        outcome.carry = closeReceipt.economics.carryAssets;
        outcome.closeFee = closeReceipt.economics.executionFeeAssets;
        (uint256 remainingSize,,,,,,) = engine.positions(ACTOR);
        assertEq(remainingSize, 0);
        assertEq(closeReceipt.economics.postTraderClaimBalanceAssets, 0);

        outcome.incumbentNavAtClose = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        outcome.replacementNavAtClose = juniorVault.convertToAssets(juniorVault.balanceOf(REPLACEMENT_LP));
        uint256 traderCash = clearinghouse.balanceAssets(ACTOR);
        vm.prank(ACTOR);
        clearinghouse.withdraw(ACTOR, traderCash);
        outcome.actorCash = settlementToken.balanceOf(ACTOR);

        outcome.incumbentCash = _claimAllJuniorAt(address(this), redemptionTime + 7_201);
        outcome.replacementCash = _claimAllJuniorAt(REPLACEMENT_LP, redemptionTime + 14_401);
        outcome.actorCash += _claimAllJuniorAt(ACTOR, redemptionTime + 21_601);
    }

    function _quoteActorExit(
        uint256 redemptionTime
    ) internal returns (uint256 paid) {
        _requestReplacementDeposit(LP_DEPOSIT, redemptionTime);
        uint256 shares = juniorVault.maxRequestRedeem(ACTOR);
        uint256 before = settlementToken.balanceOf(ACTOR);
        vm.prank(ACTOR);
        uint256 requestId = juniorVault.requestRedeem(shares, ACTOR, ACTOR);
        _settlePublicAt(redemptionTime);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ACTOR);
        assertEq(claimable, shares);
        vm.prank(ACTOR);
        juniorVault.claimRedeem(requestId, claimable, ACTOR, ACTOR);
        paid = settlementToken.balanceOf(ACTOR) - before;
    }

    function _settlePublicAt(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
    }

    function _requestReplacementDeposit(
        uint256 amount,
        uint256 activationTime
    ) internal returns (uint256 requestId) {
        settlementToken.mint(REPLACEMENT_LP, amount);
        vm.startPrank(REPLACEMENT_LP);
        settlementToken.approve(address(juniorVault), amount);
        requestId = juniorVault.requestDeposit(amount, REPLACEMENT_LP, REPLACEMENT_LP);
        vm.stopPrank();
        assertEq(juniorVault.depositEpochStart(requestId), activationTime);
    }

    function _claimAllJuniorAt(
        address lp,
        uint256 timestamp
    ) internal returns (uint256 paid) {
        vm.warp(timestamp);
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        if (shares == 0) {
            return 0;
        }
        uint256 before = settlementToken.balanceOf(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        _settlePublicAt(pool.lpEpochStart(requestId));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        assertEq(claimable, shares, "full LP redemption must be funded");
        vm.prank(lp);
        juniorVault.claimRedeem(requestId, claimable, lp, lp);
        paid = settlementToken.balanceOf(lp) - before;
    }

}
```

