# 26204 - \[SC - Insight] DeGate Operator has capability to disable balan...

## DeGate Operator has capability to disable balance checks (EOA Control Risks in DepositContractProxy Deployment)

Submitted on Nov 28th 2023 at 05:39:37 UTC by @ongrid for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26204

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Permanent freezing of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.
* Theft of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.

### Description

### Bug Description

The current deployment of `DepositContractProxy` at `0x54D7aE423Edb07282645e740C046B9373970a168` exhibits improper ownership, being controlled by an Externally Owned Account (EOA). This setup introduces significant security vulnerabilities due to the inherent risks of centralized control over system configuration.

### Severity

The [Secbit's report](https://github.com/degatedev/protocols/commit/180138015197c886ec3c87efa8bf0031b653359f#commitcomment-132582143) emphasized the elimination of centralization in managing proxy admin. While the report did not explicitly mention ownership of business logic, the current use of an EOA for contract ownership contradicts the underlying intention of these recommendations.

### Impact

#### Disabling Balance Checks

The DeGate Operator, through an EOA, has the capability to disable crucial ERC-20 balance checks. This functionality, intended to validate balance changes for non-standard tokens with custom transfer logic, can be bypassed, undermining transactional security.

#### Risks with Special Tokens

Rebased tokens, common in liquid staking, are particularly vulnerable. Manipulating `checkBalance` could lead to incorrect calculations in deposit amounts, causing financial discrepancies based on the variance between the expected and actual balance increments.

#### Theft of Funds Potential

A critical risk is the theft from the Default Deposit Contract. With disabled balance checks, specially crafted tokens could alter their balances, enabling improper `amountReceived` variable on `deposit()`.

### Conclusion

The centralized control over this configuration poses serious security threats, including potential Operator misuse and token-based exploits. Immediate remedial action is necessary to secure the platform and protect user assets.

### Risk Breakdown

Difficulty to Exploit: Easy

### Recommendations

Ownership of `DepositContractProxy` should align with the auditor's guidance on avoiding centralized control.

1. **Transfer Ownership to Multisig**: Shift the ownership of the contracts to a Gnosis Multisig wallet to ensure decentralized governance and mitigate risks.
2. **Optionally Use Timelock**: Consider transferring ownership to a Timelock contract to add an extra layer of security and transparency in the upgrade process.

### References

### Proof of concept

To demonstrate the potential misuse of an Externally Owned Account (EOA) by the DeGate Operator, I have created a specific branch in the repository with a forge test, functioning on an Ethereum mainnet fork using the address of the live contract.

Test `test/DepositCentralizedControl.t.sol::testDepositOwner` checks the owner's address bytecode, which is found to be zero, indicating that the address is indeed an EOA.

Test `test/DepositCentralizedControl.t.sol::testDepositCentralizedControlSetCheckBalance` demonstrates that the entity owning this address can alter the `checkBalance` setting of the deposit contract.

Repository: <https://leprosorium.testxyz.work/DeGate-06bfde5c-8293-4c7f-b0d0-b9674fc29a51/deGate-3c6d1cda/-/tree/31430\\_DepositContractProxy\\_owner\\_centralized>

CI with test: <https://leprosorium.testxyz.work/DeGate-06bfde5c-8293-4c7f-b0d0-b9674fc29a51/deGate-3c6d1cda/-/jobs/29>

Test output:

```
$ forge test -vvvv
installing solc version "0.8.23"
Successfully installed solc 0.8.23
Compiling 25 files with 0.8.23
Solc 0.8.23 finished in 5.59s
Compiler run successful!
Running 2 tests for test/DepositCentralizedControl.t.sol:DrainDepositExchangeTest
[PASS] testDepositCentralizedControlSetCheckBalance() (gas: 36869)
Traces:
  [41082] DrainDepositExchangeTest::testDepositCentralizedControlSetCheckBalance()
    ├─ [0] VM::prank(0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82)
    │   └─ ← ()
    ├─ [0] VM::expectEmit(true, true, true, true)
    │   └─ ← ()
    ├─ emit CheckBalance(token: FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], checkBalance: true)
    ├─ [31268] 0x54D7aE423Edb07282645e740C046B9373970a168::setCheckBalance(FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], true)
    │   ├─ [26245] 0x8CCc06C4C3B2b06616EeE1B62F558f5b9C08f973::setCheckBalance(FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], true) [delegatecall]
    │   │   ├─ emit CheckBalance(token: FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], checkBalance: true)
    │   │   └─ ← ()
    │   └─ ← ()
    ├─ [0] VM::prank(0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82)
    │   └─ ← ()
    ├─ [0] VM::expectEmit(true, true, true, true)
    │   └─ ← ()
    ├─ emit CheckBalance(token: FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], checkBalance: false)
    ├─ [2295] 0x54D7aE423Edb07282645e740C046B9373970a168::setCheckBalance(FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], false)
    │   ├─ [1876] 0x8CCc06C4C3B2b06616EeE1B62F558f5b9C08f973::setCheckBalance(FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], false) [delegatecall]
    │   │   ├─ emit CheckBalance(token: FakeToken: [0x3ACBcf03Fd92448e2631D094673fBfE50F37D08e], checkBalance: false)
    │   │   └─ ← ()
    │   └─ ← ()
    └─ ← ()
[PASS] testDepositOwner() (gas: 5022)
Traces:
  [5022] DrainDepositExchangeTest::testDepositOwner()
    └─ ← ()
Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.13s
 
Ran 1 test suites: 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## Further PoC Development

Should this issue be deemed as having CRITICAL severity, I am prepared to invest additional time to develop a comprehensive PoC aimed to demonstrate the incorrect calculation of `amountReceived` on deposit highlighting the potential financial damage. Given that the live contracts lack deposit states, this would require a complex setup with fixtures to simulate the environment accurately.
