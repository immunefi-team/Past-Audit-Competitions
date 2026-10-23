# 52347 sc high improper handling of yield distribution state in distributeyieldwithlimit leads to revert freezing users yield&#x20;

* Submitted on Aug 10th 2025 at 04:20:49 UTC by @oluwaseyisekoni for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network)
* Report ID: #52347
* Report Type: Smart Contract
* Severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* Impacts: Protocol insolvency

## Description

Brief/Intro

The `distributeYieldWithLimit` function in the ArcToken contract distributes a specified amount of yield (`totalAmount`) among token holders over multiple batched calls. However, the distribution recalculates the effective total supply and each holder's balance/eligibility dynamically for every batch. Because those values are not snapshotted at the start of the distribution process, changes between batches (token transfers, eligibility toggles, etc.) can lead to inaccurate allocations, stuck funds, or reverts that prevent completion of the distribution.

## Root Cause

The function computes `effectiveTotalSupply` and each holder's `holderBalance` at the time of each batch rather than using a snapshot taken when the distribution started. As a result, changes to:

* Token balances (`balanceOf(holder)`)
* Yield eligibility (`_isYieldAllowed(holder)`)

between batched calls change distribution denominators and numerators, producing inconsistent and potentially excessive per-holder shares across batches.

## Vulnerability Details

The simplified normal scenario:

* 10 eligible holders, each with 100 tokens
* `totalAmount = 8000` yield tokens
* If distribution were atomic, each holder gets (8000 \* 100) / 1000 = 800 yield tokens.

The vulnerability manifests when distribution is split into batches and the effective total supply or eligibility changes between batches.

{% stepper %}
{% step %}

### Scenario 1 — Eligibility State Change Between Batches

Setup:

* 10 holders, 100 tokens each → totalSupply = 1000
* `totalAmount = 8000` yield tokens
* All holders initially eligible
* Two batched calls:
  * First call: startIndex = 0, maxHolders = 3
  * Second call: startIndex = 3, maxHolders = 7
* Between the first and second call, holders 0–2 become ineligible

First call:

* EffectiveTotalSupply = 1000
* Each holder’s share = (8000 × 100) / 1000 = 800
* Distributed to holders 0–2: 3 × 800 = 2400

Second call:

* EffectiveTotalSupply recalculated excluding holders 0–2 = 700
* Each remaining holder’s share = (8000 × 100) / 700 ≈ 1142.857
* Distributing to holders 3–9 yields ≈ 8000 in this batch

Total distributed ≈ 2400 + 8000 ≈ 10,400 > totalAmount → the second call will attempt to transfer more than remaining balance and revert, freezing yield distribution.
{% endstep %}

{% step %}

### Scenario 2 — Token Balance Change Between Batches

Setup:

* 10 holders, initially 100 tokens each → totalSupply = 1000
* `totalAmount = 8000`
* All holders yield-eligible
* Two batched calls: first 3 holders, then remaining 7

First call:

* EffectiveTotalSupply = 1000
* Each of holders 0–2 gets 800 → distributed 3 × 800 = 2400

Between calls:

* Holder 4 increases balance from 100 → 1000 (e.g., buys tokens)
* New effective total = 1900

Second call:

* Recomputed effectiveTotalSupply = 1900
* Holder 4’s share ≈ (1000 / 1900) × 8000 ≈ 4210
* Other 6 holders split ≈ 3790 → \~631 each (approx), example numbers in report show \~421 for others based on arithmetic rounding in that PoC
* Second batch distributed ≈ 6736 (or higher depending on rounding)
* Total distributed ≈ 2400 + 6736 = 9136 > totalAmount → the second call will revert due to insufficient balance in the distributing contract, freezing the yield.

Impact: batched distribution can end up attempting to distribute more than `totalAmount` due to dynamic recalculation, causing revert and stuck funds.
{% endstep %}
{% endstepper %}

## Impact Details

* Users’ yield tokens can become stuck (distribution cannot be completed).
* Potential protocol insolvency if yield distributor role funds are exhausted or distributions revert permanently.
* Any actor can manipulate balances/eligibility between batches to cause a revert or change allocation.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466>

## Proof of Concept

The PoC below demonstrates Scenario 2 (balance increase between batches). It triggers a revert when the second batch attempts to transfer more than the contract holds.

```solidity
function test_manipulateYieldDistribution() public{
        
        uint256 totalAmount = 8000e18;

        uint256 effectiveTotalSupply = token.balanceOf(owner)+token.balanceOf(charlie)+token.balanceOf(alice)+token.balanceOf(dan)+token.balanceOf(bob)+token.balanceOf(ayo)+token.balanceOf(user_0)+token.balanceOf(user_1)+token.balanceOf(user_2)+token.balanceOf(user_3);
        console.log("effectiveTotalSupply: ", effectiveTotalSupply);
        
        
        vm.startPrank(owner);
        yieldToken.approve(address(token), YIELD_AMOUNT);
        console.log("Arc token balance before 1st call",totalAmount);
        token.distributeYieldWithLimit(totalAmount, 0, 3); //YIELD TOKEN
        vm.stopPrank();

        vm.startPrank(owner);
        console.log("Arc token balance before 2nd call ",yieldToken.balanceOf(address(token)));
        //increase a user balance in the next call by 900e18 token 
        console.log("dan bal before increase ",token.balanceOf(dan));
        //This was done in place of purchasing an arc token
        token.mint(dan, 900e18);
        console.log("dan bal after increase ",token.balanceOf(dan));
        
        vm.expectRevert(
            abi.encodeWithSelector(
                IERC20Errors.ERC20InsufficientBalance.selector,
                address(token),
                126315789473684210527, //balance left in arc token contract
                421052631578947368421 // required amount
            )
        );
        
        token.distributeYieldWithLimit(totalAmount, 3, 7);
        console.log("Revert Error: ERC20InsufficientBalance");
         vm.stopPrank();

    }
```

Example logs from the test:

```
[PASS] test_manipulateYieldDistribution() (gas: 477323)
Logs:
  effectiveTotalSupply:  1000000000000000000000
  Arc token balance before 1st call 8000000000000000000000
  Arc token balance before 2nd call  5600000000000000000000
  dan bal before increase  100000000000000000000
  dan bal after increase  1000000000000000000000
  Revert Error: ERC20InsufficientBalance
```

## Notes / mitigation ideas (do not add implementation)

* Snapshot effectiveTotalSupply and eligible holders' balances at the start of the distribution, and base all batched calculations on that snapshot.
* Alternatively, perform the distribution atomically (one call) or ensure idempotent per-holder accounting that references remaining undistributed amount rather than recomputing from totalAmount each batch.
* Ensure the distributor contract holds or locks the full `totalAmount` and that per-batch transfers cannot exceed the remaining allocated amount computed from a stable snapshot.
