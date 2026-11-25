# \#98359 [SC-Medium] Closes that earn a VPI rebate credit the treasury none of the execution fee; the fee stays in LP assets

**Submitted on Nov 7th 2026 at 09:33:12 UTC by @janneh2000 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98359
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

On a voluntary close whose VPI rebate is at least the 4 bp execution fee, the trader is still charged the full fee (it is netted out of the rebate), but the treasury is credited nothing and the fee stays in HousePool assets. When I fully closed a 300k position, the treasury got 0 of a 120 USDG fee. The pool held about 1M USDG, and an open in the same state that also nets a rebate tops the treasury up to the full fee. So this is not the documented case where the fee "cannot be cash-credited". I tested at immunefi-team/audit-comp-plether @ a82d4fe, packages/perps.

The two lines (packages/perps/src/libraries/CfdEnginePlanLib.sol @ a82d4fe):

```solidity
// L1067-1068 (_planCloseActionSettlement): the fee is capped to cash withheld/collected, which is 0 on a net rebate
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
// L984 (_planIsolatedCloseSettlement): the top-up uses that already-capped value, so it is 0 whatever free cash exists
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
```

The open path computes its top-up from the nominal fee instead: L399 `delta.executionFeeAssets = openState.executionFeeAssets;` and L737 `feeShortfallAssets = delta.executionFeeAssets - directlyCreditedAssets`. The docs say the fee is treasury revenue:
- WHITEPAPER L412: "the protocol execution fee belongs to treasury rather than LP assets"
- WHITEPAPER L1455: "voluntary closes pay the 4-basis-point treasury execution fee"
- ACCOUNTING_SPEC L1118: "free-cash-funded top-ups become treasury margin"
- ACCOUNTING_SPEC L782 drops only fee that "cannot be cash-credited under this priority"

**Reproduce** (one command; the PoC goes to packages/perps/test/perps/):

```
docker build -t plether-s1 . && docker run --rm plether-s1
# without Docker, from the repo root at a82d4fe:
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/perps/PoC_S1_CloseRebateFee.t.sol -vv
```

The PoC runs the real flow. The trader calls router.commitOrder, an unrelated keeper calls router.executeOrder with a fresh MockPyth tick, the fixture uses the deployment risk parameters (vpiFactor 0.01), and the price stays at 1.00 so there is no price PnL. The A/B control is the same contract with vpiFactor = 0. Results were identical across 3 runs, with via_ir on and off, and in the clean image. Raw 6-decimal atoms:

```
PoC_S1_CloseRebateFee
  open 300k LONG (pays VPI)        fee 120000000  treasury credited 120000000  pool +449101796
  full close (earns VPI rebate)    fee 120000000  treasury credited 0          pool -328900330
  BUG: treasury shortfall on close (atoms) 120000000
  Bob 1k close reducing skew       fee 400000     treasury credited 95610
  BUG: treasury shortfall on close (atoms) 304390
  Bob opens 300k SHORT (net rebate) fee 120000000 treasury credited 120000000  pool -448900596
PoC_S1_Control_NoVpi: every leg credits exactly the nominal fee
Ran 2 test suites: 6 tests passed, 0 failed
```

**Preconditions**: none beyond normal trading. Any close that reduces skew earns a rebate. Once skew is above roughly 4% of depth, the rebate exceeds the fee, so the treasury gets nothing from those closes. That is 400 USDG of fee revenue per 1M of close notional moved to LPs. 



**Prior work**: the sponsor's invariant model (test/perps/invariant/properties/PerpVpiFrozenAccountingInvariant.t.sol L118-124) computes the close credit with the same rebate-offset formula as the code. It does not explain why the close skips the free-cash top-up that the open performs under the same priority. TreasuryFeeCustody's "execution fees accrue to protocol not LP equity" check runs only with vpiFactor = 0. Cecuro MED-6 is the opposite direction: top-ups spending LP reserves.

**Suggested fix:** in _planIsolatedCloseSettlement, compute the top-up from the nominal fee (cs.executionFeeAssets) minus what was credited, still capped by _freePoolCashForProtocolFee. If the netting is intended, the whitepaper and spec lines above should say so.

Proof label: LOCAL-POC-CONFIRMED



## Link to Proof of Concept

https://gist.github.com/janneh2000/0add861d9aa5c925562e4548858b687f

## Proof of Concept

## Proof of Concept

**Reproduce** (one command; the PoC goes to packages/perps/test/perps/):

```
docker build -t plether-s1 . && docker run --rm plether-s1
# without Docker, from the repo root at a82d4fe:
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/perps/PoC_S1_CloseRebateFee.t.sol -vv
```

The PoC runs the real flow. The trader calls router.commitOrder, an unrelated keeper calls router.executeOrder with a fresh MockPyth tick, the fixture uses the deployment risk parameters (vpiFactor 0.01), and the price stays at 1.00 so there is no price PnL. The A/B control is the same contract with vpiFactor = 0. Results were identical across 3 runs, with via_ir on and off, and in the clean image. Raw 6-decimal atoms:

```
PoC_S1_CloseRebateFee
  open 300k LONG (pays VPI)        fee 120000000  treasury credited 120000000  pool +449101796
  full close (earns VPI rebate)    fee 120000000  treasury credited 0          pool -328900330
  BUG: treasury shortfall on close (atoms) 120000000
  Bob 1k close reducing skew       fee 400000     treasury credited 95610
  BUG: treasury shortfall on close (atoms) 304390
  Bob opens 300k SHORT (net rebate) fee 120000000 treasury credited 120000000  pool -448900596
PoC_S1_Control_NoVpi: every leg credits exactly the nominal fee
Ran 2 test suites: 6 tests passed, 0 failed
```
LOCAL CLI Results: 


rivaldo@Alies-MacBook-Air audit-comp-plether % FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/perps/PoC_S1_CloseRebateFee.t.sol -vv
[⠊] Compiling...
[⠆] Compiling 30 files with Solc 0.8.35
[⠰] Solc 0.8.35 finished in 65.07s
Compiler run successful with warnings:



Ran 3 tests for test/perps/PoC_S1_CloseRebateFee.t.sol:PoC_S1_CloseRebateFee
[PASS] test_FullCloseWithRebate_TreasuryGetsNoFee() (gas: 3963741)
Logs:
  open 300k LONG (pays VPI)
    nominal execution fee (atoms) 120000000
    treasury credited (atoms)     120000000
    pool assets delta (atoms)     449101796
  full close (earns VPI rebate)
    nominal execution fee (atoms) 120000000
    treasury credited (atoms)     0
    pool assets delta (atoms)     -328900330
  BUG: treasury shortfall on close (atoms) 120000000

[PASS] test_OpenWithNetRebate_TreasuryGetsFullFee() (gas: 4498243)
Logs:
  Bob opens 300k SHORT (net rebate)
    nominal execution fee (atoms) 120000000
    treasury credited (atoms)     120000000
    pool assets delta (atoms)     -448900596

[PASS] test_SmallCloseReducingSkew_TreasuryShortchanged() (gas: 5757694)
Logs:
  Bob 1k close reducing skew
    nominal execution fee (atoms) 400000
    treasury credited (atoms)     95610
    pool assets delta (atoms)     0
  BUG: treasury shortfall on close (atoms) 304390

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 32.45ms (31.83ms CPU time)

Ran 3 tests for test/perps/PoC_S1_CloseRebateFee.t.sol:PoC_S1_Control_NoVpi
[PASS] test_FullCloseWithRebate_TreasuryGetsNoFee() (gas: 3939987)
Logs:
  open 300k LONG (pays VPI)
    nominal execution fee (atoms) 120000000
    treasury credited (atoms)     120000000
    pool assets delta (atoms)     0
  full close (earns VPI rebate)
    nominal execution fee (atoms) 120000000
    treasury credited (atoms)     120000000
    pool assets delta (atoms)     266

[PASS] test_OpenWithNetRebate_TreasuryGetsFullFee() (gas: 4396849)
Logs:
  Bob opens 300k SHORT (net rebate)
    nominal execution fee (atoms) 120000000
    treasury credited (atoms)     120000000
    pool assets delta (atoms)     0

[PASS] test_SmallCloseReducingSkew_TreasuryShortchanged() (gas: 5693108)
Logs:
  Bob 1k close reducing skew
    nominal execution fee (atoms) 400000
    treasury credited (atoms)     400000
    pool assets delta (atoms)     0

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 33.78ms (31.28ms CPU time)

Ran 2 test suites in 41.40ms (66.23ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)


