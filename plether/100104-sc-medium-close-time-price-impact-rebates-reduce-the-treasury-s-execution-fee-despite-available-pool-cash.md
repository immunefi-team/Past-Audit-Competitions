# \#100104 [SC-Medium] Close-time price-impact rebates reduce the treasury’s execution fee despite available pool cash

**Submitted on Nov 16th 2026 at 08:23:37 UTC by @Shubadu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100104
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
A live close that earns a price-impact (VPI) rebate can credit the protocol treasury less than the assessed execution fee, even when pool cash is available to fund the difference. The equivalent rebate on an open leaves the treasury fee intact. In the local PoC, a 100,000 USD round trip incurs a 40 USD fee on each leg, but the treasury receives 40 USD for the open and zero for the close. The missing close fee remains in the LP pool; no trader funds are stolen.

## Vulnerability Details
The open path preserves the assessed execution fee in CfdEnginePlanLib and uses a pool-cash top-up in CfdEngineSettlementSidecar to credit the treasury when the trader’s action charge does not cover it.

The live-close path behaves differently. In CfdEnginePlanLib._planCloseActionSettlement, it sets delta.executionFeeAssets to the lesser of the assessed close fee and the non-VPI charge recovered from the trader:

    delta.executionFeeAssets =
        cs.executionFeeAssets < executionFeeEligibleAssets
            ? cs.executionFeeAssets
            : executionFeeEligibleAssets;

When a close earns a rebate that offsets the action charge, executionFeeEligibleAssets falls below the assessed fee and can reach zero. The close path then calculates its pool-cash top-up from this already reduced delta.executionFeeAssets. Consequently, the top-up cannot restore the fee portion offset by the rebate, even when free pool cash is available.

The trader’s net charge is unaffected. The discrepancy is in how that charge and the rebate are allocated between the treasury and LP pool. The whitepaper describes the execution fee as treasury revenue and voluntary closes as paying a 4-basis-point treasury fee. It also qualifies treasury credit on available cash. This report concerns a case where cash is available and the open path uses it to pay the full fee, but the close path does not.

The runnable PoC uses the frozen audit commit and the release VPI factor. It shows the affected close, a skew-reducing open that credits its full fee, and a close without a rebate that credits its full fee. All three tests pass.

## Impact Details
For an affected live close, the treasury fee shortfall is up to the lesser of the VPI rebate and the assessed close execution fee. At the tested 4-basis-point rate, the maximum shortfall is 4 USD per 10,000 USD of closed notional. That amount remains in the LP pool.

In the PoC, opening a 100,000 USD position credits the treasury 40 USD. Closing it at the same price assesses another 40 USD fee but credits the treasury zero because the close earns a rebate. The treasury receives 40 USD rather than 80 USD across the round trip. The trader does not receive the missing treasury amount as an additional payment.

The selected Low impact is “Contract fails to deliver promised returns, but doesn't lose value.” The finding concerns protocol fee revenue, not theft, insolvency, or frozen funds.

## References
Close fee clamp and top-up calculation:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L984-L987
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1064-L1068

Open fee and pool-funded top-up:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L399
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L498

Documented fee allocation:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L810-L811
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1455


## Link to Proof of Concept

https://gist.github.com/0xulver/30e00c6ee57e92cbfbc5013365c785ca

## Proof of Concept

## Proof of Concept
Runnable Foundry test (secret Gist):
https://gist.github.com/0xulver/30e00c6ee57e92cbfbc5013365c785ca

The test runs locally against the frozen audit commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. It uses the project's BasePerpTest fixture, sets the release VPI factor to 0.01e18, and asserts the 4-basis-point execution fee. It sends no transactions to a public network.

1. Open a 100,000 USD long position. The treasury receives the full 40 USD opening execution fee.
2. Close that position at the same price. The close reduces skew and earns a VPI rebate. Although the assessed closing execution fee is 40 USD, the treasury receives zero. The 40 USD shortfall remains in the LP pool.
3. As controls, open an opposite-side position that earns a rebate and confirm its full 20 USD execution fee reaches the treasury; then close a position without a rebate and confirm its full 40 USD fee reaches the treasury.

To reproduce from the root of a clean checkout at the frozen commit:

mkdir -p packages/perps/test/perps/poc
curl -fL 'https://gist.githubusercontent.com/0xulver/30e00c6ee57e92cbfbc5013365c785ca/raw/56b09f27d9cd6cf1f3c3d670bcc216c04b3f6499/CloseFeeNettedAgainstRebate.t.sol' -o packages/perps/test/perps/poc/CloseFeeNettedAgainstRebate.t.sol
cd packages/perps
forge test --offline --threads 1 --skip script --match-path 'test/perps/poc/CloseFeeNettedAgainstRebate.t.sol' -vv

The recorded run completed with 3 tests passed, 0 failed and 0 skipped. Its affected-close output was: fee assessed 40.000000 USD, treasury credited 0.000000 USD, and shortfall retained by the pool 40.000000 USD.

