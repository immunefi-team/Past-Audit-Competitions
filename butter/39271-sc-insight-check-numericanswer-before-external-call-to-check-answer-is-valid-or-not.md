# #39271 \[SC-Insight] Check \`numericAnswer\` before external call to check answer is valid or not

**Submitted on Jan 26th 2025 at 19:17:39 UTC by @iehnnkta for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39271
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Unbounded gas consumption

## Description

## Vulnerability Detail

In the function `FlatCFM::resolve`, updates the payouts array, which further ne reported into the conditional token contract.

```solidity
    function resolve() external {
        bytes32 answer = oracleAdapter.getAnswer(questionId);
        uint256[] memory payouts = new uint256[](outcomeCount + 1);
        uint256 numericAnswer = uint256(answer);

        //@audit - gas: numericAnswer is already typecasted answer. So check numericAnswer first to 0, such that there is no external call required if numericAnswer is 0 as oracle updater going to return false for answer if its uint is zero.
        if (oracleAdapter.isInvalid(answer) || numericAnswer == 0) {
            // 'Invalid' receives full payout
            payouts[outcomeCount] = 1;
        } else {
            // Each bit (i-th) in numericAnswer indicates if outcome i is 1 or 0
            for (uint256 i = 0; i < outcomeCount; i++) {
                payouts[i] = (numericAnswer >> i) & 1;
            }
        }
        conditionalTokens.reportPayouts(questionId, payouts);
    }
```

Here as `numericAnswer` is already typecasted `answer` which is in bytes32. So instead of checking `answer` is invalid or not, it is highly recommended to check `numericAnswer` is zero or not. Because if `numericAnswer` is zero, function `oracleAdapter::isInvalid` function returns false, and then `numericAnswer` returns true. Which is waste of heavy amount of gas, the function making external call.

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

contract A {

    B oracleAdapter;

    constructor (address _b) {
        oracleAdapter = B(_b);
    }

    function gasCheck() external view {
        bytes32 answer = bytes32(uint256(0));
        uint outcomeCount = 2; // let outcomeCount = 2 (yes or no)
        uint256[] memory payouts = new uint256[](outcomeCount + 1);
        uint256 numericAnswer = uint256(answer);

        if (numericAnswer == 0 || oracleAdapter.isInvalid(answer)) {
        // if (numericAnswer == 0 || oracleAdapter.isInvalid(answer)) {
            payouts[outcomeCount] = 1;
        } else {
            for (uint256 i = 0; i < outcomeCount; i++) {
                payouts[i] = (numericAnswer >> i) & 1;
            }
        }
    }
}

contract B {
    function isInvalid(bytes32 answer) public pure returns (bool) {
        return (uint256(answer) ==
            0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff);
    }
}
```

Now as you can see, same function I have recreated in the remix, to check the gas consumption of each case, i.e., `numericAnswer` is being cheked first and last. In attached images also,

1. we can see that when `numericAnswer` checked first before checking `answer` is invalid outcome or nor results in -- `653 gas`.
2. Similarly when `numericAnswer` checked after `answer` is invalid or not resluts in -- `6526 gas`
