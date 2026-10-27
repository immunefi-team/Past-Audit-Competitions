# 29186 - \[SC - Insight] ValidationLogicvalidateBorrow - L-L Incorrect i...

## `ValidationLogic::validateBorrow()` - L227-L231: Incorrect implementation of the intended (AAVE) protocol logic prevents all users with a health factor (HF) equal to `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` from borrowing.

Submitted on Mar 10th 2024 at 00:25:30 UTC by @OxSCSamurai for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29186

Report type: Smart Contract

Report severity: Insight

Target: <https://immunefi.com/>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

### Brief/Intro

`ValidationLogic::validateBorrow()` - L227-L231: Incorrect implementation of the intended (AAVE) protocol logic prevents all users with a health factor (HF) equal to `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` from borrowing.

### Vulnerability Details

Description:

Users won't be able to borrow or borrow more whenever their health factor (HF) value is EQUAL to the `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`. This is not a correct representation of intended protocol logic. Intended protocol logic makes it possible for users to borrow/borrow more whenever their `HF >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD`.

To add even more weight to my argument, see below where it's made clear that there are two different "minimum" HF liquidation thresholds, the absolute minimum allowed one, `0.95e18`, where immediate liquidation probably occurs for any HF value below it, and the other more healthy "minimum" HF liq threshold with a value of `1e18`:

```solidity
  // Minimum health factor allowed under any circumstance
  // A value of 0.95e18 results in 0.95
  uint256 public constant MINIMUM_HEALTH_FACTOR_LIQUIDATION_THRESHOLD = 0.95e18;

  /**
   * @dev Minimum health factor to consider a user position healthy
   * A value of 1e18 results in 1
   */
  uint256 public constant HEALTH_FACTOR_LIQUIDATION_THRESHOLD = 1e18;
```

There shouldn't exist any valid logical reason why `HEALTH_FACTOR_LIQUIDATION_THRESHOLD = 1e18` should be excluded as a valid minimum value as is incorrectly done in the below code snippet:

```solidity
    require(
      vars.healthFactor > HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
      Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD
    );
```

Here it's clear that the `require` check will revert whenever `vars.healthFactor <= HEALTH_FACTOR_LIQUIDATION_THRESHOLD` is true. The `<` is correct. It's the `=` in `<=` that's wrong, because the `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` is a VALID minimum (healthy) HF value, and to exclude it like the current implementation does, is in fact completely wrong.

Not only that, but check the error description: `Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD`. It's clear what it says, but the code implementation doesn't reflect this.

### Impact Details

POTENTIAL IMPACT IN SCOPE:

* Griefing -- no attacker or attack necessary, instead it's the bug itself causing griefing to the user/borrower, unfairly preventing them from borrowing when they should be able to borrow.

In case this contract is not in scope, it affects the results of at least the following contracts & functions in scope, therefore primacy of impact applies: `WrappedTokenGatewayV3::borrowETH()` and `Pool::flashLoan()` for both zksync and manta.

IMPACTS:

* Users won't be able to borrow or borrow more whenever their health factor (HF) value is EQUAL to the `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`.
* Some users who might check out your smart contracts will notice that your intended protocol logic should allow for borrowing in the case where `vars.healthFactor == HEALTH_FACTOR_LIQUIDATION_THRESHOLD`, but that the expected behaviour is different from the actual behaviour, where it doesn't allow this.
* Specifically, the following functions will revert when called under above conditions, for both zksync and manta: -- `WrappedTokenGatewayV3::borrowETH()` -- `Pool::flashLoan()`

### References

<https://github.com/zerolend/core-contracts/blob/2448f46b6b472ba0f83a615f68aa8614866a8321/contracts/protocol/libraries/logic/ValidationLogic.sol#L227-L231> <https://github.com/zerolend/core-contracts/blob/2448f46b6b472ba0f83a615f68aa8614866a8321/contracts/protocol/libraries/logic/GenericLogic.sol#L172-L176>

### Proof of Concept

PoC:

For some reason I couldnt get the tests to work in your local github repos, so I decided to run the same tests in AAVE's github repo, which should suffice.

And just for your records, here is the test error I get when I try run `npm run test` or `yarn test` in your repo:

```solidity
Generating typings for: 134 artifacts in dir: types for target: ethers-v5
Successfully generated 296 typings!
Compiled 129 Solidity files successfully (evm target: london).
[BASH] Testnet environment ready
WARNING: You are currently using Node.js v21.6.2, which is not supported by Hardhat. This can lead to unexpected behavior. See https://hardhat.org/nodejs-versions


(node:29266) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)


  1) "before all" hook in "{root}"
·------------------------|---------------------------|----------------|----------------------------·
|  Solc version: 0.8.12  ·  Optimizer enabled: true  ·  Runs: 100000  ·  Block limit: 6718946 gas  │
·························|···························|················|·····························
|  Methods                                                                                         │
··············|··········|·············|·············|················|·············|···············
|  Contract   ·  Method  ·  Min        ·  Max        ·  Avg           ·  # calls    ·  usd (avg)   │
·-------------|----------|-------------|-------------|----------------|-------------|--------------·

  0 passing (3s)
  1 failing

  1) "before all" hook in "{root}":
     ProviderError: Method not found
      at HttpProvider.request (/mnt/DATA/DEV/WEB3_SEC/BUG_HUNTING/BUG_BOUNTIES/IMMUNEFI/CURRENT/BUG_BOUNTY_BOOSTS/zerolendxyz/zerolendxyz_REPO/core-contracts/node_modules/hardhat/src/internal/core/providers/http.ts:90:21)
      at processTicksAndRejections (node:internal/process/task_queues:95:5)
      at DeploymentsManager.saveSnapshot (/mnt/DATA/DEV/WEB3_SEC/BUG_HUNTING/BUG_BOUNTIES/IMMUNEFI/CURRENT/BUG_BOUNTY_BOOSTS/zerolendxyz/zerolendxyz_REPO/core-contracts/node_modules/hardhat-deploy/src/DeploymentsManager.ts:1418:22)
      at Object.fixture (/mnt/DATA/DEV/WEB3_SEC/BUG_HUNTING/BUG_BOUNTIES/IMMUNEFI/CURRENT/BUG_BOUNTY_BOOSTS/zerolendxyz/zerolendxyz_REPO/core-contracts/node_modules/hardhat-deploy/src/DeploymentsManager.ts:323:9)
      at Context.<anonymous> (/mnt/DATA/DEV/WEB3_SEC/BUG_HUNTING/BUG_BOUNTIES/IMMUNEFI/CURRENT/BUG_BOUNTY_BOOSTS/zerolendxyz/zerolendxyz_REPO/core-contracts/test-suites/__setup.spec.ts:5:3)



error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

## THE AAVE TESTS:

Command used for all tests: `npm run test`

## TESTS:

* Test 1: With bug not fixed
* Test 2: With bug fixed
* Test 3: Control test.

## Test 1:

* With bug not fixed and explicitly passing in `vars.healthFactor = 1e18` via a slight modification in `GenericLogic::calculateUserAccountData()` - L172-L177:

```solidity
    vars.healthFactor = 1e18; /// @audit added for PoC/testing purposes
    // vars.healthFactor = (vars.totalDebtInBaseCurrency == 0)
    //   ? type(uint256).max
    //   : (vars.totalCollateralInBaseCurrency.percentMul(vars.avgLiquidationThreshold)).wadDiv(
    //     vars.totalDebtInBaseCurrency
    //   ); /// @audit NOTE: this is required for some of the protocol tests to execute successfully...
```

## Test 1 results:

```solidity
  AToken: Mint and Burn Event Accounting
    ✔ User 1 supplies DAI
    ✔ User 1 supplies DAI on behalf of user 2

    1) User 2 supplies ETH,and borrows DAI
    .
    .
    .
    .
    .
    .
  63 passing (23s)
  1 failing

  1) AToken: Mint and Burn Event Accounting
       User 2 supplies ETH,and borrows DAI:
     Error: VM Exception while processing transaction: reverted with reason string '35'
    at BorrowLogic.validateBorrow (contracts/protocol/libraries/logic/ValidationLogic.sol:145)
    at BorrowLogic.executeBorrow (contracts/protocol/libraries/logic/BorrowLogic.sol:85)
    at Pool.liquidationCall (contracts/protocol/pool/Pool.sol:366)
    at Pool.borrow (contracts/protocol/pool/Pool.sol:219)
    at InitializableImmutableAdminUpgradeabilityProxy._delegate (contracts/dependencies/openzeppelin/upgradeability/Proxy.sol:42)
    at InitializableImmutableAdminUpgradeabilityProxy._fallback (contracts/dependencies/openzeppelin/upgradeability/Proxy.sol:71)
    at InitializableImmutableAdminUpgradeabilityProxy.<fallback> (contracts/dependencies/openzeppelin/upgradeability/Proxy.sol:18)
    at EdrProviderWrapper.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:415:41)
    at EthersProviderWrapper.send (node_modules/@nomiclabs/hardhat-ethers/src/internal/ethers-provider-wrapper.ts:13:20)
```

In above test result with one failed test the reason string `35` is:

```solidity
string public constant HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD = '35'; // 'Health factor is lesser than the liquidation threshold'
```

I explicitly passed in a HF exactly equal to the constant `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`, and the buggy require check saw this as an invalid HF.

The next test was run with the bug fixed and still explicitly passing in `vars.healthFactor = 1e18` as the borrower's HF.

## Test 2: with bug fixed

```solidity
    require(
      //vars.healthFactor > HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
      vars.healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD, /// @audit added for PoC/testing purposes
      Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD
    );
```

## Test 2 results (see my two @audit tags below):

```solidity
  AToken: Mint and Burn Event Accounting
    ✔ User 1 supplies DAI
    ✔ User 1 supplies DAI on behalf of user 2
    ✔ User 2 supplies ETH,and borrows DAI 						/// @audit failed in test 1, passes here
    ✔ User 2 borrows more DAI - confirm mint event includes accrued interest
    ✔ User 1 - supplies more DAI - confirm mint event includes accrued interest
    ✔ User 1 supplies more DAI again - confirm mint event includes accrued interest
    ✔ User 2 repays all remaining DAI
    ✔ User 1 withdraws all deposited funds and interest
    ✔ User 2 borrows, pass time and repay DAI less than accrued debt
    ✔ User 1 withdraws amount less than accrued interest
    .
    .
    .
    .
    .
    .
  AToken: Transfer
    ✔ User 0 deposits 1000 DAI, transfers 1000 to user 0
    ✔ User 0 deposits 1000 DAI, disable as collateral, transfers 1000 to user 1
    ✔ User 0 deposits 1000 DAI, transfers 5 to user 1 twice, then transfer 0 to user 1
    ✔ User 0 deposits 1000 DAI, transfers to user 1
    ✔ User 0 deposits 1 WETH and user 1 tries to borrow the WETH with the received DAI as collateral

    1) User 1 tries to transfer all the DAI used as collateral back to user 0 (revert expected) /// @audit this is OK, it's because of the code snippet I commented out so that I could pass in `vars.healthFactor = 1e18` explicitly. Uncomment this code snippet and this test will pass.
    .
    .
    .
    .
    .
    .
  104 passing (27s)
  1 failing

  1) AToken: Transfer
       User 1 tries to transfer all the DAI used as collateral back to user 0 (revert expected):
     AssertionError: Expected transaction to be reverted with reason '35', but it didn't revert
      at Context.<anonymous> (test-suites/atoken-transfer.spec.ts:228:5)
```

Here the bugfixed worked, the user was able to borrow successfully >>> `✔ User 2 supplies ETH,and borrows DAI`. Ignore the failed test, it's expected because I commented out the code snippet necessary for this failed test to pass...

## Test 3: Control test.

* using test command: `npm run test`
* Here I removed all my changes that I made to the codebase for testing purposes, but I left my bugfix in there, and ran all the tests again, to see if my bugfix causes any issues with any of your tests. The bugfix again:

```solidity
    require(
      //vars.healthFactor > HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
      vars.healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD, /// @audit added for PoC/testing purposes
      Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD
    );
```

## Test 3 test result:

```solidity
.
.
.
  WadRayMath
    ✔ Plain getters
    ✔ wadMul()
    ✔ wadDiv()
    ✔ rayMul()
    ✔ rayDiv()
    ✔ rayToWad()
    ✔ wadToRay()

·--------------------------------------------------------------------------------------------|---------------------------|----------------|-----------------------------·
|                                    Solc version: 0.8.10                                    ·  Optimizer enabled: true  ·  Runs: 100000  ·  Block limit: 12450000 gas  │
········
.
.
.
.
.
.
·······
|  ZeroReserveInterestRateStrategy                                                           ·          -  ·          -  ·        232430  ·        1.9 %  ·          -  │
·--------------------------------------------------------------------------------------------|-------------|-------------|----------------|---------------|-------------·

  813 passing (3m)
```

NO issues, all AAVE tests ran successfully with my bugfix implemented, as expected.

## THE BUGFIX:

`ValidationLogic::validateBorrow()` - L227-L231:

```diff
        require(
-           vars.healthFactor > HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
+           vars.healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
            Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD
        );
```
