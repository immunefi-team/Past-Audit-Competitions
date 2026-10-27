# #39181 \[SC-Insight] Bond Fund will be Lost When Question is Asked Again

**Submitted on Jan 24th 2025 at 08:08:06 UTC by @Topmark for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39181
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Asking a MetricQuestion on the FlatCFMRealityAdapter contract has a flaw when MetricQuestion is ask again after the first time for the purpose of confirmation or any other necessary reasons. The functions returns the call without refunding the caller since the call has previously gone through thereby locking caller's fund in contract

## Vulnerability Details

```solidity
    function askMetricQuestion(
        uint256 metricTemplateId,
        GenericScalarQuestionParams calldata genericScalarQuestionParams,
        string memory outcomeName
    ) public payable override returns (bytes32) {
        string memory formattedMetricQuestionParams = _formatMetricQuestionParams(outcomeName);
>>>        return _askQuestion(metricTemplateId, formattedMetricQuestionParams, genericScalarQuestionParams.openingTime);
    }
```

The function above from the FlatCFMRealityAdapter contract shows how metric question is asked typically during creation of conditional scalar market in the FlatCFMFactory contract. askMetricQuestion(...) function is a payable function and the msg.value is used to call the askQuestionWithMinBond function, It can be noted from the pointer above that a call is made to the \_askQuestion(...) function in the same contract. <https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/src/FlatCFMRealityAdapter.sol#L121-L123>

```solidity
    function _askQuestion(uint256 templateId, string memory formattedQuestionParams, uint32 openingTime)
        private
        returns (bytes32)
    {
        // See RealityETH reference for how question IDs are derived.
        bytes32 contentHash = keccak256(abi.encodePacked(templateId, openingTime, formattedQuestionParams));
        bytes32 questionId = keccak256(
            abi.encodePacked(
                contentHash, arbitrator, questionTimeout, minBond, address(oracle), address(this), uint256(0)
            )
        );

>>>        // If already asked, return existing questionId.
        if (oracle.getTimeout(questionId) != 0) {
>>>            return questionId;
        }

        // Otherwise ask a new question with the provided parameters.
        return oracle.askQuestionWithMinBond{value: msg.value}(
            templateId, formattedQuestionParams, arbitrator, questionTimeout, openingTime, 0, minBond
        );
    }
```

The implementation of the \_askQuestion(...) function as noted in the two pointers above shows that when question has already being asked the code returns without the need to call askQuestionWithMinBond with the fund instead of reverting with the implication that fund attached to this call would be locked in contract since no refund mechanism is in place in the function.

## Impact Details

Callers Fund will be completely lost to the FlatCFMRealityAdapter contract when call is made to the askMetricQuestion(...) function subsequent times after the first call

## Mitigation

Protocol should either revert the ask Metric call in subsequent calls to revert fund used or msg.value used to make the call can be refunded in the code directly

## Proof of Concept

## Proof of Concept

```solidity
function testAskMetricQuestionLostFund(uint256 sendValue) public {
    vm.assume(sendValue >= 1 ether);
    deal(address(this), sendValue * 2);

    // Store the initial balance before the first call
    uint256 initialBalance = address(this).balance;

    // Expect the first call to askMetricQuestion with the specified sendValue
    vm.expectCall(
        address(reality), sendValue, abi.encodeWithSelector(IRealityETHCore.askQuestionWithMinBond.selector)
    );
    realityAdapter.askMetricQuestion{value: sendValue}(metricTemplateId, genericScalarQuestionParams, "A");

    // Assert the balance after the first call (balance should have decreased by sendValue)
    uint256 balanceAfterFirstCall = address(this).balance;
    assertEq(balanceAfterFirstCall, initialBalance - sendValue, "Balance after first call should decrease by sendValue");

    // Prepare for the second call
    uint256 secondSendValue = sendValue ;
    uint256 balanceBeforeSecondCall = address(this).balance;

    // Ensure the balance is enough for the second call
    assert(balanceBeforeSecondCall >= secondSendValue, "Insufficient balance for second call");

    // Expect the second call to askMetricQuestion with the second sendValue
    vm.expectCall(
        address(reality), secondSendValue, abi.encodeWithSelector(IRealityETHCore.askQuestionWithMinBond.selector)
    );
    realityAdapter.askMetricQuestion{value: secondSendValue}(metricTemplateId, genericScalarQuestionParams, "B");

    // the balance after the second call would have decreased and lost
    uint256 balanceAfterSecondCall = address(this).balance;
    assertEq(balanceAfterSecondCall, balanceBeforeSecondCall , " fund lost to contract after second call ");
}
```

code can be copied and pasted to the FlatCFMRealityAdapter.t.sol test contract. The test will fail as callers balance have been lost to the contract after the second call
