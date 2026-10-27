# 58088 sc low inadequate enforcement of global cap enables cumulative over allocation

**Submitted on Oct 30th 2025 at 15:14:39 UTC by @Arkindyo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58088
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistStrategyClassifier.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `AlchemistStrategyClassifier` defines `globalCap` as an MYT‑wide, per‑risk‑class maximum allocation ceiling intended to bound total exposure over time. However, `PerpetualGauge` enforces allocations against the current `totalIdleAssets` and a per‑execution accumulator `totalRiskAllocated` inside a single allocation loop. Because enforcement is scoped to the current execution context instead of the cumulative total allocated across all prior rounds, repeated executions can incrementally exceed the intended global cap, bypassing governance limits and increasing systemic risk.

## Vulnerability Details

The system’s cap semantics and enforcement are misaligned: `globalCap` is documented as a global, cumulative ceiling per risk class across the entire MYT scope, yet `PerpetualGauge` only constrains the amount allocated within a single execution against `totalIdleAssets`. Over multiple executions, the per‑round constraint fails to account for already allocated assets, allowing total risk‑class allocation to drift above governance’s upper bound. In production, this can lead to concentration of funds beyond approved limits, magnifying the blast radius of downstream protocol failures or malicious upgrades.

The `globalCap` in `AlchemistStrategyClassifier` is described as “the aggregate maximum allocation for this risk class within MYT,” implying a cumulative, cross‑round, cross‑time constraint on outstanding allocations.

[src/AlchemistStrategyClassifier.sol:RiskClass#L20](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistStrategyClassifier.sol#L20)

```solidity
    struct RiskClass {
        uint256 globalCap; // Max allocation for all strategies in this class combined
        uint256 localCap; // Max allocation for this single strategy in the class
    }
```

However, allocation is governed by values derived from `totalIdleAssets` (the current idle base) and a local per‑loop accumulator `totalRiskAllocated` for the running execution in `PerpetualGauge`. The check ensures the loop’s incremental allocations don’t exceed a bound computed from idle assets and per‑iteration totals, but it does not subtract previously allocated amounts nor compare against the risk class’s outstanding allocation across the system.

[src/PerpetualGauge.sol:executeAllocation#L152-L161](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/PerpetualGauge.sol#L152C1-L161)

```solidity

    function executeAllocation(uint256 ytId, uint256 totalIdleAssets) external nonReentrant {
...
        for (uint256 i = 0; i < sIds.length; i++) {
            uint8 risk = stratClassifier.getStrategyRiskLevel(sIds[i]);
            uint256 indivCap = stratClassifier.getIndividualCap(sIds[i]);
            uint256 globalCap = stratClassifier.getGlobalCap(risk);

            uint256 target = (weights[i] * totalIdleAssets) / 1e18;

            // Individual cap
            uint256 capIndiv = (indivCap * totalIdleAssets) / 1e4;
            if (target > capIndiv) target = capIndiv;

            // Global cap for risk group
            if (risk > 0) {
                uint256 capGlobalLeft = (globalCap * totalIdleAssets) / 1e4 - totalRiskAllocated;
                if (target > capGlobalLeft) target = capGlobalLeft;
                totalRiskAllocated += target;
            }
```

Consequence: On the first execution, the loop may correctly keep the per‑round increment under `globalCap`. On subsequent executions, `totalRiskAllocated` is reset for the loop, and the base remains tied to `totalIdleAssets`. As idle replenishes (e.g., deposits, yield, deallocations to idle), the loop can allocate again up to the per‑round bound, cumulatively pushing the aggregate outstanding allocation for the risk class beyond `globalCap`.

## Impact Details

Governance policy bypass: The system can hold more exposure than permitted for a risk class, nullifying governance‑mandated ceilings.

Amplified loss potential: If any strategy or downstream protocol within an over‑allocated risk class fails (rug, exploit, malicious upgrade), losses scale with the excess exposure above `globalCap`.

Up to the sum of all per‑round increments accumulated above `globalCap`. In the worst case, repeated executions can push exposure to the entire vault capacity for that risk class if idle continues to refill, making potential losses unbounded relative to the intended cap.

## Recommendations

Enforce cumulative cap: Maintain a persistent counter `riskClassOutstanding[RiskClass]` representing total currently allocated assets per risk class across the system. Before allocating `delta`, enforce `riskClassOutstanding[risk] + delta <= globalCap[risk]`.

## References

Contract semantics: `src/AlchemistStrategyClassifier.sol` (global cap meaning and per‑risk configuration). [src/AlchemistStrategyClassifier.sol:RiskClass#L20](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistStrategyClassifier.sol#L20)

Enforcement logic: `src/PerpetualGauge.sol` (allocation loop using `totalIdleAssets` and `totalRiskAllocated` per execution). [src/PerpetualGauge.sol:executeAllocation#L152-L161](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/PerpetualGauge.sol#L152C1-L161)

## Proof of Concept

## Proof of Concept

Creates a focused test src/test/PerpetualGaugeGlobalCapPoC.t.sol that proves globalCap is enforced per execution only, allowing cumulative over-allocation across rounds.

* Sets globalCap for risk class 1 to 3000 bps and indivCap to 10000 bps.
* Calls executeAllocation four times. Each run allocates 300,000 (30%), totaling 1,200,000 , exceeding the intended global cap if interpreted as cumulative outstanding exposure.

```bash
Ran 1 test for src/test/PerpetualGaugeGlobalCapPoC.t.sol:PerpetualGaugeGlobalCapPoC
[PASS] test_poc_globalCap_cumulative_bypass() (gas: 403160)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.93ms (3.34ms CPU time)

Ran 1 test suite in 329.48ms (13.93ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Test.sol";
import {PerpetualGauge} from "../PerpetualGauge.sol";
import {IERC20Metadata} from "@openzeppelin/contracts/interfaces/IERC20Metadata.sol";
import {IStrategyClassifier} from "../interfaces/IStrategyClassifier.sol";

contract MockERC20 is IERC20Metadata {
    string public name = "Mock Token";
    string public symbol = "MCK";
    uint8 public decimals = 18;
    uint256 public totalSupply = 1e24;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor() {
        balanceOf[msg.sender] = totalSupply;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient");
        require(allowance[from][msg.sender] >= amount, "Not allowed");
        balanceOf[from] -= amount;
        allowance[from][msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

contract MockStrategyClassifier {
    mapping(uint256 => uint8) public risk;
    mapping(uint256 => uint256) public indivCap;
    mapping(uint8 => uint256) public globalCap;

    function setRisk(uint256 stratId, uint8 _risk) external { risk[stratId] = _risk; }
    function setIndivCap(uint256 stratId, uint256 cap) external { indivCap[stratId] = cap; }
    function setGlobalCap(uint8 riskLevel, uint256 cap) external { globalCap[riskLevel] = cap; }
    function getStrategyRiskLevel(uint256 stratId) external view returns (uint8) { return risk[stratId]; }
    function getIndividualCap(uint256 stratId) external view returns (uint256) { return indivCap[stratId]; }
    function getGlobalCap(uint8 riskLevel) external view returns (uint256) { return globalCap[riskLevel]; }
}

contract TrackingAllocatorProxy {
    event Allocated(uint256 strategyId, uint256 amount);

    IStrategyClassifier private immutable _classifier;
    mapping(uint8 => uint256) public riskTotal;
    mapping(uint256 => uint256) public stratTotal;

    constructor(address classifier) { _classifier = IStrategyClassifier(classifier); }

    function allocate(uint256 strategyId, uint256 amount) external {
        stratTotal[strategyId] += amount;
        uint8 r = _classifier.getStrategyRiskLevel(strategyId);
        riskTotal[r] += amount;
        emit Allocated(strategyId, amount);
    }
}

contract GaugeHarness is PerpetualGauge {
    constructor(address c, address a, address v) PerpetualGauge(c, a, v) {}
    function addStrategy(uint256 ytId, uint256 sid) external { strategyList[ytId].push(sid); }
}

contract PerpetualGaugeGlobalCapPoC is Test {
    GaugeHarness gauge;
    MockERC20 token;
    MockStrategyClassifier classifier;
    TrackingAllocatorProxy allocator;

    address alice = address(0xA11CE);

    function setUp() public {
        token = new MockERC20();
        classifier = new MockStrategyClassifier();
        allocator = new TrackingAllocatorProxy(address(classifier));

        gauge = new GaugeHarness(address(classifier), address(allocator), address(token));

        token.transfer(alice, 1e21);

        // Risk class 1, no individual cap (10000 bps), global cap 30% (3000 bps)
        classifier.setRisk(1, 1);
        classifier.setIndivCap(1, 10_000);
        classifier.setGlobalCap(1, 3_000);

        gauge.addStrategy(1, 1);
    }

    function test_poc_globalCap_cumulative_bypass() public {
        // Single strategy gets full weight
        vm.prank(alice);
        gauge.vote(1, _arr(1), _arr(100));

        uint256 idle = 1_000_000;
        uint256 perRoundCap = (3_000 * idle) / 10_000; // 30% of idle

        // Execute allocation multiple times; each run respects per-round cap
        gauge.executeAllocation(1, idle);
        gauge.executeAllocation(1, idle);
        gauge.executeAllocation(1, idle);
        gauge.executeAllocation(1, idle);

        uint256 cumulative = allocator.riskTotal(1);
        assertEq(cumulative, perRoundCap * 4);

        // Cumulative exposure exceeds intended global cap semantics
        assertGt(cumulative, perRoundCap);
    }

    function _arr(uint256 v) internal pure returns (uint256[] memory) {
        uint256[] memory arr = new uint256[](1);
        arr[0] = v;
        return arr;
    }
}

```
