# 28660 - \[SC - Insight] pufETHsrcTimelock\_setDelay - L State constant M...

## `pufETH/src/Timelock::_setDelay()` - L256: State constant `MINIMUM_DELAY` is incorrectly treated as an invalid minimum delay value, as can be seen here where `newDelay <= MINIMUM_DELAY` is used instead of `newDelay < MINIMUM_DELAY`.

Submitted on Feb 23rd 2024 at 08:47:31 UTC by @OxSCSamurai for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28660

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

### Brief/Intro

## Summary:

* The hardcoded constant value for timelock minimum delay is *incorrectly* treated as an *invalid* timelock delay value.

## Impact in scope:

* Contract fails to deliver promised returns, but doesn't lose value
* In this context the promised "return" is related to the hardcoded minimum timelock delay value of `7 days`.

### Vulnerability Details

Buggy function:

```solidity
    function _setDelay(uint256 newDelay) internal {
        if (newDelay <= MINIMUM_DELAY) {
            revert InvalidDelay(newDelay);
        }
        emit DelayChanged(delay, newDelay);
        delay = newDelay;
    }
```

If for any reason it's not clear yet what the bug is, let me clarify for you:

```solidity
    /**
     * @notice Minimum delay enforced by the contract
     */
    uint256 public constant MINIMUM_DELAY = 7 days;
```

It should be clear enough from the above that the value for `MINIMUM_DELAY` is a VALID minimum value for the timelock delay period. Therefore the `<=` in `newDelay <= MINIMUM_DELAY` should be `<`, because a `newDelay` value of 7 days should be 100% valid, but anything LESS than 7 days is INVALID, therefore we should use `newDelay < MINIMUM_DELAY` instead, so that it translates to all values of `newDelay` where `newDelay >= MINIMUM_DELAY`, are all VALID values.

### Impact Details

* Can never change the timelock delay value to the hardcoded VALID minimum delay period of 7 days as is intended protocol functionality because the buggy function was called during constructor/deployment time, as well as called whenever they want to change the delay again after deployment.
* The lowest possible value(in days) for the timelock delay is 8 days, contrary to the "promised" minimum delay of 7 days.
* Investors/VCs, regulators, users, protocol DAO/governance, multisigs, external integrating projects and external devs will all expect the timelock's minimum possible delay to be 7 days(as promised by the hardcoded `MINIMUM_DELAY` constant), but this is not the case. Anyone who plans/strategizes or builds according to this expectation, will be disappointed to learn that a delay of 7 days is not possible.
* it seems there might have been some confusion during protocol tests and test setup and especially during deployment contract setup & testing as to "weird" behaviour when a minimum delay of 7 days was selected, so the response was to modify it to `7 days + 1` in order to bypass the "weird"/revert behaviour... when in fact it was simply the bug in the internal `_setDelay()` function.

## Recommendations:

* Ensure that the hardcoded value of constant `MINIMUM_DELAY` is treated correctly throughout the codebase. Here correctly means that `7 days` timelock delay is still a valid value to be used in the protocol and that only anything strictly less than `7 days` is invalid.
* Ensure that during tests setup & testing that assumptions are properly validated. It seems the intention here was to deploy with `7 days` as the initial timelock delay value, but your tests/deployment transactions reverted, due to this bug. The tests/deploy script was then modified to bypass this revert, by using a modified initial timelock delay value of `7 days + 1`.
* Correct understanding of minimum & maximum boundaries in terms of valid vs invalid values is crucial, and the implementation of this correct understanding must be consistently applied throughout the codebase.

## BUGFIX:

```diff
    function _setDelay(uint256 newDelay) internal {
-       if (newDelay <= MINIMUM_DELAY) {
+       if (newDelay < MINIMUM_DELAY) {
            revert InvalidDelay(newDelay);
        }
        emit DelayChanged(delay, newDelay);
        delay = newDelay;
    }
```

### References

<https://github.com/PufferFinance/pufETH/blob/3e76d02c7b54323d347c8277327d3877bab591f5/src/Timelock.sol#L255-L262> <https://github.com/PufferFinance/pufETH/blob/3e76d02c7b54323d347c8277327d3877bab591f5/src/Timelock.sol#L256>

### Proof of Concept

## TEST1:

<https://github.com/PufferFinance/pufETH/blob/3a9f943b3a9133cb2ee9bbf8c39e876c4170ead7/script/DeployPuffETH.s.sol#L95>

```solidity
initialDelay: 7 days
```

<https://github.com/PufferFinance/pufETH/blob/3a9f943b3a9133cb2ee9bbf8c39e876c4170ead7/script/DeployPuffETH.s.sol#L190>

```solidity
uint256 delayInSeconds = 7 days;
```

With the bug *not yet fixed*(`newDelay <= MINIMUM_DELAY`) and the protocol tests modified to use `7 days` instead of (`0` or `7 days + 1`) as per above code snippets, we get an `InvalidDelay` error/revert in the test results using forge command `ETH_RPC_URL=https://mainnet.infura.io/v3/YOUR_KEY forge test --contracts script/DeployPuffETH.s.sol -vvvvv`:

```solidity
    ├─ [2801070] DeployPuffETH::run()
    │   ├─ [0] VM::startBroadcast(<pk>)
    │   │   └─ ← ()
    │   ├─ [2092564] → new AccessManager@0x652Fa863F2476bE5606f0611b622AE51f411BF05
    │   │   ├─ emit RoleGranted(roleId: 0, account: 0xC4a2E012024d4ff28a4E2334F58D4Cc233EB1FE1, delay: 0, since: 1702902095 [1.702e9], newMember: true)
    │   │   └─ ← 10323 bytes of code
    │   ├─ [337052] → new NoImplementation@0x844FA36d460c4F0c4e14d900Ad9B294f223936c7
    │   │   └─ ← 1683 bytes of code
    │   ├─ [56850] → new PufferDepositor@0x1bA145F1BE82d7285FFfAee5f79dc7744ffD0a9F
    │   │   ├─ emit Upgraded(implementation: NoImplementation: [0x844FA36d460c4F0c4e14d900Ad9B294f223936c7])
    │   │   └─ ← 163 bytes of code
    │   ├─ [0] VM::label(PufferDepositor: [0x1bA145F1BE82d7285FFfAee5f79dc7744ffD0a9F], "PufferDepositor")
    │   │   └─ ← ()
    │   ├─ [56850] → new PufferVault@0x989e351E99b9c7Ff8a31042521C772b0ECED39D3
    │   │   ├─ emit Upgraded(implementation: NoImplementation: [0x844FA36d460c4F0c4e14d900Ad9B294f223936c7])
    │   │   └─ ← 163 bytes of code
    │   ├─ [0] VM::label(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3], "PufferVault")
    │   │   └─ ← ()
    │   ├─ [23096] → new <unknown>@0x9696768d5e2B611BD89181D54AeB3259Bab9616F
    │   │   └─ ← 0 bytes of code
    │   └─ ← InvalidDelay(604800 [6.048e5])
    └─ ← InvalidDelay(604800 [6.048e5])

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.81s

Ran 5 test suites in 5.43s: 9 tests passed, 3 failed, 0 skipped (12 total tests)

Failing tests:
Encountered 1 failing test in test/Integration/PufferTest.integration.t.sol:PufferTest
[FAIL. Reason: setup failed: ] setUp() (gas: 0)

Encountered 1 failing test in test/unit/PufETH.t.sol:PufETHTest
[FAIL. Reason: setup failed: ] setUp() (gas: 0)

Encountered 1 failing test in test/unit/Timelock.t.sol:TimelockTest
[FAIL. Reason: setup failed: ] setUp() (gas: 0)

Encountered a total of 3 failing tests, 9 tests succeeded
```

> Result: Function reverts & test fails. Timelock delay of `7 days` is *not accepted*.

## TEST2:

Same test as above, but now with the bug *fixed*(`newDelay < MINIMUM_DELAY`) and using a valid `7 days` for the timelock delay duration:

```solidity
    ├─ [3539] PufferVault::totalAssets() [staticcall]
    │   ├─ [3155] PufferVaultImplementation::totalAssets() [delegatecall]
    │   │   ├─ [283] stETHStrategyMock::userUnderlyingView(PufferVault: [0x82c8Ea3945357f79Ca276eDd08a216ba2dCf48c0]) [staticcall]
    │   │   │   └─ ← 0
    │   │   ├─ [259] stETHStrategyMock::sharesToUnderlyingView(0) [staticcall]
    │   │   │   └─ ← 0
    │   │   ├─ [582] stETHMock::balanceOf(PufferVault: [0x82c8Ea3945357f79Ca276eDd08a216ba2dCf48c0]) [staticcall]
    │   │   │   └─ ← 94506801511339732177893208105523307560345234525891122052510 [9.45e58]
    │   │   └─ ← 94506801511339732177893208105523307560345234525891122052510 [9.45e58]
    │   └─ ← 94506801511339732177893208105523307560345234525891122052510 [9.45e58]
    └─ ← ()

[PASS] test_withdraw((address[4],uint256[4],uint256[4],int256),uint256,uint256) (runs: 256, μ: 2369, ~: 2369)
Traces:
  [2369] PufETHTest::test_withdraw(Init({ user: [0x37C85e8501C85678DD78C2e0efD289aFCa4247C7, 0xB96dd076492F49544e2Ef2D9076B858E655dB047, 0xEbA49D5E5cEAA64Fe4B27adfEf7C0698732711F2, 0x60e520Ec502962BbF2b7D2C77c40e64c0fE02c1C], share: [201392022 [2.013e8], 115792089237316195423570985008687907853269984665640564039457584007913129639933 [1.157e77], 136575469017119510220658243480759860199685581935592905362192466467983506 [1.365e71], 426953058856943339976487681355759295678073420707694526 [4.269e53]], asset: [77338257301509117895204304330339740461772430350267748 [7.733e52], 115792089237316195423570985008687907853269984665640564039457584007913129639934 [1.157e77], 244644773103235396247098 [2.446e23], 456479280374691886728263 [4.564e23]], yield: -121899787668446801097984673461 [-1.218e29] }), 3, 986255612626336828686695110186512734 [9.862e35])
    └─ ← ()

Test result: ok. 28 passed; 0 failed; 0 skipped; finished in 4.34s

Ran 5 test suites in 10.91s: 73 tests passed, 0 failed, 0 skipped (73 total tests)
```

> Result: Function does not revert & test passes. Timelock delay of `7 days` is *accepted*.

## TEST3 (control test):

Same test as above with the bug *fixed*, but now using an invalid `6 days` for the timelock delay duration:

```solidity
    ├─ [2801073] DeployPuffETH::run()
    │   ├─ [0] VM::startBroadcast(<pk>)
    │   │   └─ ← ()
    │   ├─ [2092564] → new AccessManager@0x652Fa863F2476bE5606f0611b622AE51f411BF05
    │   │   ├─ emit RoleGranted(roleId: 0, account: 0xC4a2E012024d4ff28a4E2334F58D4Cc233EB1FE1, delay: 0, since: 1702902095 [1.702e9], newMember: true)
    │   │   └─ ← 10323 bytes of code
    │   ├─ [337052] → new NoImplementation@0x844FA36d460c4F0c4e14d900Ad9B294f223936c7
    │   │   └─ ← 1683 bytes of code
    │   ├─ [56850] → new PufferDepositor@0x1bA145F1BE82d7285FFfAee5f79dc7744ffD0a9F
    │   │   ├─ emit Upgraded(implementation: NoImplementation: [0x844FA36d460c4F0c4e14d900Ad9B294f223936c7])
    │   │   └─ ← 163 bytes of code
    │   ├─ [0] VM::label(PufferDepositor: [0x1bA145F1BE82d7285FFfAee5f79dc7744ffD0a9F], "PufferDepositor")
    │   │   └─ ← ()
    │   ├─ [56850] → new PufferVault@0x989e351E99b9c7Ff8a31042521C772b0ECED39D3
    │   │   ├─ emit Upgraded(implementation: NoImplementation: [0x844FA36d460c4F0c4e14d900Ad9B294f223936c7])
    │   │   └─ ← 163 bytes of code
    │   ├─ [0] VM::label(PufferVault: [0x989e351E99b9c7Ff8a31042521C772b0ECED39D3], "PufferVault")
    │   │   └─ ← ()
    │   ├─ [23099] → new <unknown>@0x9696768d5e2B611BD89181D54AeB3259Bab9616F
    │   │   └─ ← 0 bytes of code
    │   └─ ← InvalidDelay(518400 [5.184e5])
    └─ ← InvalidDelay(518400 [5.184e5])

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.90s

Ran 5 test suites in 5.69s: 9 tests passed, 3 failed, 0 skipped (12 total tests)

Failing tests:
Encountered 1 failing test in test/Integration/PufferTest.integration.t.sol:PufferTest
[FAIL. Reason: setup failed: ] setUp() (gas: 0)

Encountered 1 failing test in test/unit/PufETH.t.sol:PufETHTest
[FAIL. Reason: setup failed: ] setUp() (gas: 0)

Encountered 1 failing test in test/unit/Timelock.t.sol:TimelockTest
[FAIL. Reason: setup failed: ] setUp() (gas: 0)

Encountered a total of 3 failing tests, 9 tests succeeded
```

> Result: Function reverts & test fails. Timelock delay of `6 days` is *not accepted*.

One final control test:

## TEST4 (control test):

Same test as above with the bug *fixed*, but now using a valid `8 days` for the timelock delay duration:

```solidity
[PASS] test_withdraw((address[4],uint256[4],uint256[4],int256),uint256,uint256) (runs: 256, μ: 2369, ~: 2369)
Traces:
  [2369] PufETHTest::test_withdraw(Init({ user: [0x03A82056bAB04f51C7d6E532c32b4b3B11188e46, 0x0b5d116b67820c614cbd7aEf3297688167C9a0dF, 0xDbB4259652ca40D69C512C4D5fac86b5804a11F8, 0x6E4EA3c76Dc73DBDbd489f34FbF72337A971EC08], share: [38593803265743856358143737028896761580205631369 [3.859e46], 0, 17620440897629012719824909732430709341823 [1.762e40], 115792089237316195423570985008687907853269984665640564039457584007913129639934 [1.157e77]], asset: [5136927862229800454922703926439179510821543455777654697287529967893985191365 [5.136e75], 71257621897496127261948174471705669 [7.125e34], 1, 574310754599693725601821427145485271805407772430040 [5.743e50]], yield: 1704 }), 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77], 14793923 [1.479e7])
    └─ ← ()

Test result: ok. 28 passed; 0 failed; 0 skipped; finished in 3.97s

Ran 5 test suites in 11.91s: 73 tests passed, 0 failed, 0 skipped (73 total tests)
```

> Result: Function does not revert & test passes. Timelock delay of `8 days` is *accepted*.
