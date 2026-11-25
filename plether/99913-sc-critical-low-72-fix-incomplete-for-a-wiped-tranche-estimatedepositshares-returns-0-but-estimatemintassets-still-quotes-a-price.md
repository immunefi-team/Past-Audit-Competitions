# \#99913 [SC-Critical] LOW-72 fix incomplete: for a wiped tranche estimateDepositShares returns 0, but estimateMintAssets still quotes a price

**Submitted on Nov 15th 2026 at 18:01:06 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99913
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

**Severity: Insight.** Filed under the program's Insight Reporting rules. No impact from the program's list is claimed; the Impact section says why.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/TrancheVault.sol`: `estimateMintAssets` (lines 1372-1376) → `_previewMintAssets` (1869-1873) → `_convertToAssetsUsingAssets` (1861-1867). The same unguarded pricing base also serves `convertToShares` (515-519).

**Relation to prior findings:** the fix for Cecuro **LOW-72** ("Pending depositor can receive impossible share quotes for a wiped tranche", pp. 213-214; fix commit `899675594d1e`) is incomplete. It guards the forward estimate only. The program's known-issues rule makes a report on a fixed root valid when it shows that the fix "is incomplete or introduces a new bug".

## Summary

For a tranche whose deposit-pricing assets are zero while its shares survive (a terminal wipe), the share price is undefined. LOW-72 reported that `estimateDepositShares` still returned a large share quote in that state. The fix made it return 0 (`TrancheVault.sol:1366-1368`), and the interface documents the rule: "Returns zero when projected deposit-pricing assets are zero with surviving shares" (`IAsyncTrancheVault.sol:41-42`).

The inverse estimate was not changed. `estimateMintAssets(shares)` prices the same zero assets with no guard, so it returns `ceil(shares × (0 + 1) / (supply + 1000))`: a positive, near-zero price. After a Junior wipe in the PoC:

- `estimateDepositShares(1,000 USDC)` = **0** (the LOW-72 fix);
- `maxRequestDeposit(anyone)` = **0**, and `requestDeposit` reverts `TrancheVault__TerminallyWiped`;
- `estimateMintAssets(1e12)` = **1** (0.000001 USDC), for the 1e12 shares that cost 1,000 USDC before the loss;
- `estimateMintAssets(1e21)` = **1e9** (1,000 USDC), about 1e-9 of the healthy price.

So the vault's two deposit estimates disagree about the same tranche in the same block, and the inverse one quotes a payable entry price that `maxRequestDeposit` and `requestDeposit` refuse. `convertToShares(1,000 USDC)` on the same tranche still returns 1.000000001e21 shares, which is the large forward quote LOW-72 described, on the ERC-7575 surface.

No value moves. No contract in scope calls `estimateMintAssets`. This is filed as an Insight: the LOW-72 rule should hold on the inverse estimate as well.

## Relation to prior findings

**Cecuro LOW-72** (Acknowledged, "Addressed"). Same root: a deposit quote that is not aware of a wiped tranche. Its text, its locations (`TrancheVault.sol:1298-1308`, `:1344-1348`, `:1827-1838` in Cecuro's numbering) and its remediation ("Return zero from `estimateDepositShares()` when tranche deposits are unavailable …") name only the forward estimate and the lens that forwards it. The fix comment says: "estimateDepositShares returns zero when projected deposit-pricing assets are zero and shares survive, including before stored principal has been reconciled." The maintainers' regression test for the fix (`packages/perps/test/perps/spec/lp/DepositEstimate.t.sol`) checks `estimateDepositShares` only. `estimateMintAssets` reads the same `_depositPricingAssets()` (`TrancheVault.sol:1848-1851`) and was left unguarded. This report shows that the fix is incomplete on the inverse path.

Other nearby items, and why they are different:
- **Cecuro LOW-14** ("Integrators can overestimate request capacity because maxRequestDeposit ignores controller"): `maxRequestDeposit(address(0))`. That is a capacity view, not a price quote. In this report `maxRequestDeposit` already returns the right answer (0).
- **Cecuro LOW-43** ("Integrator can query wrong async vault limits through IERC7575"): which `max*` view reports claim capacity versus request capacity. It is about documentation of limits, not about pricing a wiped tranche.
- **Cecuro LOW-56** ("Integrators can mis-handle rejected deposits because the lens says they stay pending"): the lens's request-state wording for rejected deposits. Different root ("Not confirmed").
- The pre-audit report, the Cantina and collaborative audit reports and the AI scan contain nothing on the inverse estimate.

## Vulnerability details

The fixed forward estimate (`TrancheVault.sol:1361-1370`):

```solidity
    function estimateDepositShares(
        uint256 assets
    ) public view returns (uint256) {
        uint256 pricingAssets = _depositPricingAssets();
        uint256 pricingSupply = accruedTotalSupply();
        if (pricingAssets == 0 && pricingSupply != 0) {
            return 0;
        }
        return quoteDepositFromState(assets, pricingAssets, pricingSupply, 0);
    }
```

The unguarded inverse estimate (`TrancheVault.sol:1372-1376`, `:1869-1873`, `:1861-1867`):

```solidity
    function estimateMintAssets(
        uint256 shares
    ) external view returns (uint256) {
        return _previewMintAssets(shares);
    }

    function _previewMintAssets(
        uint256 shares
    ) internal view returns (uint256) {
        return _convertToAssetsUsingAssets(shares, _depositPricingAssets(), Math.Rounding.Ceil);
    }

    function _convertToAssetsUsingAssets(
        uint256 shares,
        uint256 totalAssets_,
        Math.Rounding rounding
    ) internal view returns (uint256) {
        return Math.mulDiv(shares, totalAssets_ + 1, accruedTotalSupply() + VIRTUAL_SHARES, rounding);
    }
```

With `_depositPricingAssets() == 0` and `accruedTotalSupply() = S > 0`, the inverse estimate is `ceil(shares × 1 / (S + 1000))`. The `+ 1` is the virtual-asset term that protects a fresh vault from inflation. For a wiped tranche with surviving supply it is the only thing left in the numerator, so it becomes the quoted price. Any `shares ≤ S + 1000` is quoted at 1 asset-wei, and larger amounts at about `shares / S` asset-wei. That is the mirror image of the impossible quote LOW-72 reported, where the forward estimate divided by the same `+ 1` and returned a huge share count. Both tranche vaults run this code; `IS_SENIOR` only selects which principal `_depositPricingAssets()` reads. The PoC wipes Junior, the tranche that absorbs losses first.

`convertToShares` (`TrancheVault.sol:515-519`) calls `_convertToSharesUsingAssets(assets, _depositPricingAssets(), Floor)` without the guard. It returns `assets × (S + 1000) / 1`, which is the pre-fix forward quote.

Admission is closed in the same state: `maxRequestDeposit` returns 0 when `_isTerminallyWiped()` (`TrancheVault.sol:1319-1321`), and `requestDeposit` reverts `TrancheVault__TerminallyWiped` (`:1585-1587`).

## Impact

**What breaks:** a documented rule of the LOW-72 remediation. The vault's deposit-quote surface says "no quote" for a wiped tranche in one direction (`estimateDepositShares` = 0, and its NatSpec states the rule) and gives a positive price in the other (`estimateMintAssets`). The same holds for the ERC-7575 `convertToShares`.

**Who is affected:** an off-chain consumer that sizes a deposit in shares. `previewMint` reverts by design in this asynchronous vault (`TrancheVault__AsyncPreviewUnavailable`), and the interface points integrations to the `estimate*` functions instead. A UI or script that asks "how much USDC for N shares" therefore gets a near-zero price for a tranche that cannot be entered, while the forward estimate for the same tranche says 0 shares. LOW-72's impact sentence covers the same consumers: "Off-chain consumers can misstate a wiped request as still convertible into shares, even though the only reachable outcome is cancellation and refund."

**Why this is an Insight and not a Low:**
- No value moves. Nobody can act on the quote: `requestDeposit` reverts in the wiped state (shown in the PoC), and a depositor whose request was pending before the wipe gets a full refund through `cancelPendingDeposit` (shown in the PoC).
- No contract in scope calls `estimateMintAssets`. The in-repo lens (`PerpsPublicLens.sol:353`, out of scope) forwards only `estimateDepositShares`.
- The state needs a terminal wipe: trader PnL larger than the tranche's principal. The PoC reaches it through ordinary trading, but it is rare in operation.


## Recommendation

Apply the LOW-72 rule in the shared deposit-pricing path, so that every quote built on `_depositPricingAssets()` sees the wipe, not only `estimateDepositShares`. For the inverse estimate, a revert is clearer than 0, because a 0-asset mint quote can be read as "free":

```solidity
    function _previewMintAssets(
        uint256 shares
    ) internal view returns (uint256) {
        uint256 pricingAssets = _depositPricingAssets();
        if (pricingAssets == 0 && accruedTotalSupply() != 0) {
            revert TrancheVault__TerminallyWiped(); // or return 0 / type(uint256).max, documented as "no quote"
        }
        return _convertToAssetsUsingAssets(shares, pricingAssets, Math.Rounding.Ceil);
    }
```

Document the result in the `estimateMintAssets` NatSpec (`IAsyncTrancheVault.sol:49-51`) next to the forward rule. For `convertToShares`, ERC-4626 says the function must not revert, so return 0 in the same state, as `estimateDepositShares` does. Add the inverse case to `DepositEstimate.t.sol`, which today checks only the forward estimate.


## Link to Proof of Concept

https://gist.github.com/Richter2552/11c3985989b221c0acc7deb11388697e

## Proof of Concept

## Proof of Concept

**When this Insight matters (Insight Reporting rule):** while a tranche is terminally wiped, which is the exact state LOW-72 covers. In practice that is any period after a Junior tranche has been wiped by trader PnL, before it is recapitalized. In that period, any integration that uses `estimateMintAssets` (the documented replacement for `previewMint`) or the ERC-7575 `convertToShares` to price an entry shows a price for an entry that the vault refuses, and contradicts the forward estimate that the LOW-72 fix corrected.

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `BasePerpTest` fixture. Its `setUp` seeds both tranches and activates trading, as the owner does at launch. After that, the test uses only public entry points from unprivileged accounts:
- a Senior LP deposits 100,000 USDC through `TrancheVault.requestDeposit` / `claimDeposit`;
- the LP epoch is settled through the permissionless `HousePool.settleLpEpoch`, and marks are installed through the permissionless `router.updateMarkPrice`;
- ALICE requests a 500 USDC Junior deposit, which is still pending at the wipe;
- trader DAVE opens LONG $30k and closes it after the basket falls to 0.9, using production `OrderRouter.commitOrder` from his own account and `router.executeOrder` from a keeper with no role. His profit, about $3k, exceeds Junior principal ($1k), so Junior's projected principal and deposit-pricing assets become 0 while its 1e12 shares survive. Senior is not wiped (about $99k).

Pyth is the maintainers' `MockPyth` (outside the scope). No protocol contract is pranked, no pool cash is moved by the test, and no storage is written. Both estimates read the pool's projected tranche state, so the test reads them right after the close, as LOW-72's fix comment describes ("including before stored principal has been reconciled").

Each **property** test states the behaviour the code should have and **fails** on the pinned code. It accepts any "no entry" answer from the inverse estimate: a revert, 0 (the value the fix uses for the forward estimate), or `type(uint256).max`. Each **control/characterize** test passes and pins the exact behaviour.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card2260S3WipedQuoteParity.t.sol packages/perps/test/perps/spec/lp/
cd packages/perps
forge test --match-contract '^Card2260S3WipedQuoteParityTest$' --match-test '^test_Property_' -vv
forge test --match-contract '^Card2260S3WipedQuoteParityTest$' --match-test '^test_(Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run against the pinned commit (forge 1.8.1, solc 0.8.35). Result: 3 tests passed, 2 failed (the two property tests). Excerpt (forge `-vv` logs; USDC amounts in 1e6 units, shares in share-wei):

```console
[FAIL: inverse quote must also say entry is impossible] test_Property_WipedInverseQuoteAgreesWithForward_Junior() (gas: 10532388)
Logs:
  junior principal at the open (stored)    999990868
  junior projected principal / pricing     0 0
  senior projected principal / pricing     99000000147 99000000147
  junior share supply                      1000000000000
  ALICE pending deposit (USDC)             500000000
  estimateMintAssets(shares) on the wiped tranche 1 1

[FAIL: inverse quote must also say entry is impossible] test_Property_WipedInverseQuoteForSeedShares_Junior() (gas: 10243051)
Logs:
  ...
  estimateMintAssets(shares) on the wiped tranche 1000000000000 1

[PASS] test_Characterize_NoValuePath() (gas: 10011841)
Logs:
  ...
  ALICE cancel refund (USDC)               500000000

[PASS] test_Characterize_WipedTrancheQuotes() (gas: 10393978)
Logs:
  ...
  wiped: estimateDepositShares(1000 USDC)      0
  wiped: maxRequestDeposit(BOB)                0
  wiped: seed shares                           1000000000000
  wiped: estimateMintAssets(1 share-wei)       1
  wiped: estimateMintAssets(seed shares)       1
  wiped: estimateMintAssets(1000e18)           1000000000
  wiped: convertToShares(1000 USDC)            1000000001000000000000

[PASS] test_Control_HealthyTrancheQuotesBothWays() (gas: 2243617)
Logs:
  healthy: estimateDepositShares(1000 USDC) 1000009132083
  healthy: estimateMintAssets(seed shares)  999990869

Suite result: FAILED. 3 passed; 2 failed; 0 skipped
```

Expected output on a fixed contract: both property tests pass, the control still passes, and the two `test_Characterize_` tests fail, because they pin the current positive inverse quote.

### What the tests show

| step | test | result on the pinned code |
|---|---|---|
| control: before any loss, the Junior forward estimate is positive and the inverse estimate prices the seed's 1e12 shares at ~1,000 USDC; entry is open | `test_Control_HealthyTrancheQuotesBothWays` | passes |
| DAVE's profitable close wipes Junior through trading (projected principal and pricing assets 0, shares 1e12 survive, Senior ~$99k). The forward estimate is 0 and `maxRequestDeposit` is 0, so the inverse estimate of 1 share-wei should also say entry is impossible | `test_Property_WipedInverseQuoteAgreesWithForward_Junior` | **FAILS**: `estimateMintAssets(1)` = 1 |
| same state: the seed's 1e12 shares (1,000 USDC before the loss) should not be quoted as mintable | `test_Property_WipedInverseQuoteForSeedShares_Junior` | **FAILS**: `estimateMintAssets(1e12)` = 1 |
| the exact values side by side, including `estimateMintAssets(1e21)` = 1e9 and `convertToShares(1,000 USDC)` = 1.000000001e21 | `test_Characterize_WipedTrancheQuotes` | passes |
| no value path: a new depositor's `requestDeposit` reverts `TrancheVault__TerminallyWiped`; ALICE's pre-wipe 500 USDC request is refunded in full by `cancelPendingDeposit` | `test_Characterize_NoValuePath` | passes |

### Test file: `packages/perps/test/perps/spec/lp/Card2260S3WipedQuoteParity.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title The LOW-72 fix guards the forward deposit estimate of a wiped tranche, but not the inverse mint estimate
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install in packages/perps/test/perps/spec/lp/ and
///         run with the repository's own foundry.toml.
/// @dev Cecuro LOW-72 ("Pending depositor can receive impossible share quotes for a wiped tranche") was fixed by
///      returning 0 from `estimateDepositShares` when projected deposit-pricing assets are zero and shares survive
///      (TrancheVault.sol:1361-1370). The inverse estimate, `estimateMintAssets` (:1372-1376 -> `_previewMintAssets`
///      :1869-1873 -> `_convertToAssetsUsingAssets` :1861-1867), prices against the same zero assets with no guard:
///      `shares * (0 + 1) / (supply + VIRTUAL_SHARES)`, rounded up. For the wiped tranche it therefore quotes a positive,
///      near-zero price for any share amount, while the forward estimate (0) and `maxRequestDeposit` (0) say entry
///      is impossible. The ERC-7575 `convertToShares` (:515-519) shares the same unguarded deposit-pricing base and
///      still returns the large forward quote LOW-72 described; it is logged in the characterization test.
///      The wipe is reached through ordinary trading: a trader opens a LONG through `OrderRouter.commitOrder` and the
///      keeper path `router.executeOrder`, the basket falls, and the trader closes at a profit larger than Junior
///      principal. LP epochs settle through the permissionless `HousePool.settleLpEpoch`, marks are installed through
///      the permissionless `router.updateMarkPrice`, Pyth is the maintainers' MockPyth (outside the scope). After the
///      fixture's setUp (which seeds both tranches and activates trading, as the owner does at launch) no protocol
///      contract is pranked, no pool cash is moved by the test, and no storage is written.
contract Card2260S3WipedQuoteParityTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant ALICE = address(0xA11CE); // Junior depositor whose request is pending at the wipe
    address internal constant BOB = address(0xB0B); // would-be new Junior depositor
    address internal constant DAVE = address(0xDA7E); // ordinary trader
    address internal constant KEEPER = address(0x5EE5); // no role anywhere; any EOA
    address internal constant REFRESHER = address(0x4EF4); // any EOA

    uint256 internal constant SENIOR_LP_ASSETS = 100_000e6;
    uint256 internal constant ALICE_DEPOSIT = 500e6;
    uint256 internal constant TRADER_FUNDS = 20_000e6;
    uint256 internal constant LONG_SIZE = 30_000e18; // $30k notional at $1; max skew ~0.4 x $102k
    uint256 internal constant LONG_MARGIN = 3_000e6;
    uint256 internal constant OPEN_PRICE = 1e8;
    uint256 internal constant CLOSE_PRICE = 0.9e8; // LONG gains ~$3k > Junior principal ($1k seed)

    uint256 private _clientNonce;

    /// No Junior deposit in the fixture setUp: its helper pranks the router into `engine.updateMarkPrice`.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Public-entry-point helpers
    // ---------------------------------------------------------------------------------------------------------------

    /// A live Pyth tick at the current block time, chained to the last engine mark time.
    function _tickNow(
        uint256 price
    ) private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    /// Anyone installs a fresh accepted basket via the permissionless `router.updateMarkPrice`.
    function _publicRefresh(
        uint256 price
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow(price);
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
    }

    /// Anyone settles the LP epoch through the permissionless HousePool entry point (no open positions here).
    function _publicSettle(
        uint256 price
    ) private {
        _publicRefresh(price);
        vm.prank(REFRESHER);
        pool.settleLpEpoch(0, 0);
    }

    /// An LP funds a tranche through the vault's public request/claim flow.
    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activationTime = vault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _publicSettle(OPEN_PRICE);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "ghost: LP deposit finalized");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card-2260-s3", ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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

    /// `trader` commits through production `OrderRouter.commitOrder`; KEEPER executes on the first tick after commit.
    function _tradeThroughRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 markAtCommit,
        uint256 fillPrice
    ) private {
        vm.warp(vm.getBlockTimestamp() + 10);
        _publicRefresh(markAtCommit);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose);
        vm.prank(trader);
        uint64 orderId = OrderRouter(address(router)).commitOrder(req);

        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        vm.warp(uint256(committedAt) + 2);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(fillPrice)), 0, int32(-8), committedAt + 1, committedAt
        );
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(fillPrice);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    /// Senior is funded by an LP; ALICE requests a Junior deposit that is still pending when DAVE's profitable LONG
    /// close takes more than Junior principal out of the pool.
    function _wipeJuniorThroughTrading() private returns (uint256 aliceRequest) {
        assertTrue(pool.isTradingActive(), "ghost: trading activated");
        _publicDeposit(seniorVault, SENIOR_LP, SENIOR_LP_ASSETS);
        _fundTrader(DAVE, TRADER_FUNDS); // DAVE's own USDC through clearinghouse.deposit

        settlementToken.mint(ALICE, ALICE_DEPOSIT);
        vm.startPrank(ALICE);
        settlementToken.approve(address(juniorVault), ALICE_DEPOSIT);
        aliceRequest = juniorVault.requestDeposit(ALICE_DEPOSIT, ALICE, ALICE);
        vm.stopPrank();

        uint256 juniorBefore = pool.juniorPrincipal();
        _tradeThroughRouter(DAVE, CfdTypes.Side.LONG, LONG_SIZE, LONG_MARGIN, false, OPEN_PRICE, OPEN_PRICE);
        _tradeThroughRouter(DAVE, CfdTypes.Side.LONG, LONG_SIZE, 0, true, CLOSE_PRICE, CLOSE_PRICE);
        (uint256 sizeAfter,,,,,,) = engine.positions(DAVE);
        assertEq(sizeAfter, 0, "ghost: DAVE closed");

        // Both estimates read the pool's projected (pending) tranche state, so the wipe is visible to them at once,
        // before any epoch settlement stores it (LOW-72's comment: "including before stored principal has been
        // reconciled"). (A `settleLpEpoch` attempted at ALICE's epoch start in this state reverted
        // `HousePool__NoLpEpochProgress` while this test was built, so the test reads the projected state directly.)
        (uint256 seniorProjected, uint256 juniorProjected,,) = pool.getPendingTrancheState();
        (uint256 seniorPricing, uint256 juniorPricing) = pool.getPendingDepositTrancheState();
        _publicRefresh(CLOSE_PRICE); // the mark stays fresh for the reads below
        console2.log("junior principal at the open (stored)   ", juniorBefore);
        console2.log("junior projected principal / pricing    ", juniorProjected, juniorPricing);
        console2.log("senior projected principal / pricing    ", seniorProjected, seniorPricing);
        console2.log("junior share supply                     ", juniorVault.totalSupply());
        console2.log("ALICE pending deposit (USDC)            ", juniorVault.pendingDepositRequest(aliceRequest, ALICE));
        assertEq(juniorProjected, 0, "ghost: Junior wiped by trader PnL");
        assertEq(juniorPricing, 0, "ghost: Junior deposit-pricing assets are zero");
        assertEq(juniorVault.totalAssets(), 0, "ghost: Junior totalAssets is zero");
        assertGt(juniorVault.totalSupply(), 0, "ghost: Junior shares survive the wipe");
        assertGt(seniorProjected, 0, "ghost: Senior is not wiped");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Control: a healthy tranche quotes both directions consistently
    // ---------------------------------------------------------------------------------------------------------------

    /// CONTROL: before any loss, the Junior forward and inverse estimates agree (1,000 USDC <-> the seed's shares).
    function test_Control_HealthyTrancheQuotesBothWays() public {
        _publicDeposit(seniorVault, SENIOR_LP, SENIOR_LP_ASSETS);
        uint256 seedShares = juniorVault.balanceOf(address(this));
        uint256 fwd = juniorVault.estimateDepositShares(1_000e6);
        uint256 inv = juniorVault.estimateMintAssets(seedShares);
        console2.log("healthy: estimateDepositShares(1000 USDC)", fwd);
        console2.log("healthy: estimateMintAssets(seed shares) ", inv);
        assertGt(fwd, 0, "forward quote positive");
        assertApproxEqRel(inv, 1_000e6, 0.001e18, "inverse quote ~1,000 USDC for the seed's shares");
        assertGt(juniorVault.maxRequestDeposit(BOB), 0, "entry open");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Property: once the tranche is wiped, the inverse estimate agrees with the forward estimate (entry impossible)
    // ---------------------------------------------------------------------------------------------------------------

    /// The inverse estimate "says entry is impossible" if it reverts, or returns one of the no-quote values: 0 (the
    /// value the LOW-72 fix gives the forward estimate and `maxRequestDeposit`) or type(uint256).max. A finite positive
    /// price is a quote for an entry that cannot happen. The maintainers choose the form; the property accepts any.
    function _inverseQuotesNoEntry(
        uint256 shares
    ) private returns (bool) {
        try juniorVault.estimateMintAssets(shares) returns (uint256 assets) {
            console2.log("estimateMintAssets(shares) on the wiped tranche", shares, assets);
            return assets == 0 || assets == type(uint256).max;
        } catch {
            return true;
        }
    }

    /// PROPERTY (LOW-72 remediation; IAsyncTrancheVault.sol:41-42 "Returns zero when projected deposit-pricing assets
    /// are zero with surviving shares"): for a wiped tranche the inverse estimate must not quote a price either.
    /// FAILS on the pinned code: estimateMintAssets(1) == 1.
    function test_Property_WipedInverseQuoteAgreesWithForward_Junior() public {
        _wipeJuniorThroughTrading();
        uint256 fwd = juniorVault.estimateDepositShares(1_000e6);
        uint256 cap = juniorVault.maxRequestDeposit(BOB);
        assertEq(fwd, 0, "forward suppressed (LOW-72 fix)");
        assertEq(cap, 0, "admission closed");
        assertTrue(_inverseQuotesNoEntry(1), "inverse quote must also say entry is impossible");
    }

    /// PROPERTY, same rule on a realistic amount: the shares the seed bought for 1,000 USDC must not be quoted as
    /// mintable for (almost) nothing. FAILS on the pinned code: estimateMintAssets(1e12) == 1.
    function test_Property_WipedInverseQuoteForSeedShares_Junior() public {
        _wipeJuniorThroughTrading();
        uint256 seedShares = juniorVault.balanceOf(address(this));
        assertTrue(_inverseQuotesNoEntry(seedShares), "inverse quote must also say entry is impossible");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Characterization: the exact values, and that nobody can act on the inverse quote
    // ---------------------------------------------------------------------------------------------------------------

    /// CHARACTERIZE: after the wipe, every quote and gate for the same tranche, side by side.
    function test_Characterize_WipedTrancheQuotes() public {
        _wipeJuniorThroughTrading();
        uint256 seedShares = juniorVault.balanceOf(address(this));
        uint256 fwd = juniorVault.estimateDepositShares(1_000e6);
        uint256 cap = juniorVault.maxRequestDeposit(BOB);
        uint256 inv1 = juniorVault.estimateMintAssets(1);
        uint256 invSeed = juniorVault.estimateMintAssets(seedShares);
        uint256 invBig = juniorVault.estimateMintAssets(1_000e18);
        uint256 conv = juniorVault.convertToShares(1_000e6);
        console2.log("wiped: estimateDepositShares(1000 USDC)     ", fwd);
        console2.log("wiped: maxRequestDeposit(BOB)               ", cap);
        console2.log("wiped: seed shares                          ", seedShares);
        console2.log("wiped: estimateMintAssets(1 share-wei)      ", inv1);
        console2.log("wiped: estimateMintAssets(seed shares)      ", invSeed);
        console2.log("wiped: estimateMintAssets(1000e18)          ", invBig);
        console2.log("wiped: convertToShares(1000 USDC)           ", conv);
        assertEq(fwd, 0, "forward suppressed");
        assertEq(cap, 0, "admission closed");
        assertEq(inv1, 1, "inverse quotes 1 wei for 1 share-wei");
        assertGt(invSeed, 0, "inverse quote positive for the seed's shares");
        assertLt(invSeed, 1_000e6 / 1000, "and below 0.1% of what those shares cost");
    }

    /// CHARACTERIZE: the contradiction has no on-chain consequence. A new depositor cannot act on the quote, and the
    /// pending pre-wipe depositor keeps the cancel/refund path (LOW-72's own scenario).
    function test_Characterize_NoValuePath() public {
        uint256 aliceRequest = _wipeJuniorThroughTrading();
        assertGt(juniorVault.estimateMintAssets(1), 0, "inverse quote positive in the wiped state");
        settlementToken.mint(BOB, 1_000e6);
        vm.startPrank(BOB);
        settlementToken.approve(address(juniorVault), 1_000e6);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        juniorVault.requestDeposit(1_000e6, BOB, BOB);
        vm.stopPrank();

        vm.prank(ALICE);
        uint256 refunded = juniorVault.cancelPendingDeposit(aliceRequest, ALICE, ALICE);
        console2.log("ALICE cancel refund (USDC)              ", refunded);
        assertEq(refunded, ALICE_DEPOSIT, "pre-wipe depositor recovers the full escrow");
        assertEq(settlementToken.balanceOf(ALICE), ALICE_DEPOSIT, "refund paid to ALICE");
    }

}
```

