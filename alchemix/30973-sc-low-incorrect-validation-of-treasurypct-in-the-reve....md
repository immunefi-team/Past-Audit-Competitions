# 30973 - \[SC - Low] Incorrect Validation of treasuryPct in the Reve...

Submitted on May 9th 2024 at 23:24:07 UTC by @The\_Seraphs for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30973

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Smart contract unable to operate due to lack of token funds
* Protocol insolvency

## Description

## Brief/Intro

The constructor of the `RevenueHandler` contract incorrectly checks an uninitialised state variable (`treasuryPct`) instead of the parameter (`_treasuryPct`). This issue could lead to setting an unintended treasury percentage without proper validation.

**Note:** *I've set as a medium severity, due to setting an incorrect treasury percentage could lead to financial discrepancies in revenue distribution between the Treasury and veALCX holders - Ultimately affecting the protocol's economic model. However, I do understand that this is part of a constructor, so the likelihood is **low** and the `setTreasuryPct()` can be used to fix once noticed.*

## Vulnerability Details

### Components Affected

* Contract Name: `RevenueHandler`
* Functionality: Constructor and Treasury Percentage Setup

## Impact Details

The constructor of the `RevenueHandler` contract is intended to initialise the treasury percentage (`treasuryPct`) used to calculate the portion of revenues sent to the treasury. The code currently checks the uninitialised state variable `treasuryPct` instead of the input parameter `_treasuryPct`. The correct line should validate `_treasuryPct` as it's the input provided during contract deployment and affects subsequent financial calculations.

If not corrected, this bug could allow the initialisation of the `RevenueHandler` with a `treasuryPct` exceeding 100%. This could lead to errors in calculating the treasury's share of revenues, potentially resulting in economic losses or unintended distribution of funds.

**Current implementation:**

```solidity
    constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {
        veALCX = _veALCX;
        require(_treasury != address(0), "treasury cannot be 0x0");
        treasury = _treasury;
        require(treasuryPct <= BPS, "treasury pct too large"); // incorrect check
        treasuryPct = _treasuryPct;
    }
```

**Proposed fix:**

```solidity
    constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {
        veALCX = _veALCX;
        require(_treasury != address(0), "treasury cannot be 0x0");
        treasury = _treasury;
        require(_treasuryPct <= BPS, "treasury pct too large"); // corrected
        treasuryPct = _treasuryPct;
```

## References

* Link to code (contract): <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L73C1-L79C6>
* Link to code (test file): <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/test/BaseTest.sol#L151>

## Proof of Concept

* Using the existing `src/test/RevenueHandler.t.sol` test contract
* Add the following test function

```solidity
    function testRevenueHandlerTreasuryPct() public {
        uint256 treasuryPct = revenueHandler.treasuryPct();
        console.log("Treasury percentage: %s", treasuryPct);
    }
```

* Go to the `BaseTest.sol` test contract and adjust setUp() which initiates the `RevenueHandler` with the required parameters.

```solidity=153
        revenueHandler = new RevenueHandler(address(veALCX), admin, 50000); // @audit change to higher than BPT
```

* Run the test from the terminal `forge test --mt testRevenueHandlerTreasuryPct -vv`

**Results:**

```shell
[⠒] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/RevenueHandler.t.sol:RevenueHandlerTest
[PASS] testRevenueHandlerTreasuryPct() (gas: 10892)
Logs:
  Treasury percentage: 50000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.25s (70.92ms CPU time)

Ran 1 test suite in 6.62s (6.25s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
