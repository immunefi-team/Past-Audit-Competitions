# #39524 \[SC-Insight] Incorrect Outcome Formatting in Reality Adapter Leads to Wrong Number of Outcome

**Submitted on Jan 31st 2025 at 21:39:28 UTC by @NHristov for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39524
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

When outcome names are passed as an array to the Reality adapter, they are transformed into a comma-separated string with quotation marks. The current string-format logic can incorrectly parse or escape quotes. This can cause Reality.eth to interpret outcomes differently than intended, resulting in extra outcomes in the question.

## Vulnerability Details

The vulnerability is in the `FlatCFMRealityAdapter::_formatDecisionQuestionParams` function. This function formats an array of outcome names into a JSON-like string for Reality.eth. If the outcome names contain special characters like commas with quotes, the formatted string may be misinterpreted by Reality.eth, leading to an incorrect number of outcomes.

```solidity
function _formatDecisionQuestionParams(FlatCFMQuestionParams calldata flatCFMQuestionParams)
        private
        pure
        returns (string memory)
    {
        bytes memory formattedOutcomes = abi.encodePacked('"', flatCFMQuestionParams.outcomeNames[0], '"');
        for (uint256 i = 1; i < flatCFMQuestionParams.outcomeNames.length; i++) {
            formattedOutcomes = abi.encodePacked(formattedOutcomes, ',"', flatCFMQuestionParams.outcomeNames[i], '"');
        }
        return string(abi.encodePacked(formattedOutcomes));
    }
```

Example:

Input outcome names: \["Like Two "C", "B"", "No"] Formatted string: "Like Two "C", "B"","No" Reality.eth interprets this as three outcomes: "Like Two ", "C", "B"","No"

## Impact Details

Incorrect Outcome Count: Reality.eth may interpret the formatted string as having more or fewer outcomes than intended. Invalid Market Configuration: The conditional tokens may be configured incorrectly, leading to invalid market behavior.

## References

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/FlatCFMRealityAdapter.sol#L84-L94>

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/FlatCFMRealityAdapter.sol#L109C5-L130C6>

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/FlatCFMRealityAdapter.sol#L109C5-L130C6>

## Proof of Concept

The following code snippet can be appended at the bottom of the `AskQuestionTest` contract in `FlatCFMRealityAdapter.t.sol`

```solidity
    event QuestionAsked(string question);

    function testAskDecisionQuestionWithOutcomesContainingInvalidChars() public {
        FlatCFMQuestionParams memory flatCFMQuestionParams =
            FlatCFMQuestionParams({outcomeNames: new string[](2), openingTime: uint32(block.timestamp + 1000)});
        flatCFMQuestionParams.outcomeNames[0] = "Like Two \"C\", \"B\"";
        flatCFMQuestionParams.outcomeNames[1] = "No";

        vm.expectEmit(true, false, false, true, address(reality));

        emit QuestionAsked("\"Like Two \"C\", \"B\"\",\"No\"");
        realityAdapter.askDecisionQuestion(decisionTemplateId, flatCFMQuestionParams);
    }
```

Also, the reality eth dummy contract should emit the appropriate event with the question data so we need to override the askQuestionWithMinBod as this

```solidity
    event QuestionAsked(string question);

    function askQuestionWithMinBond(
        uint256 template_id,
        string memory question,
        address arbitrator,
        uint32 timeout,
        uint32 opening_ts,
        uint256 nonce,
        uint256 min_bond
    ) external payable virtual returns (bytes32) {
        bytes32 content_hash = keccak256(abi.encodePacked(template_id, opening_ts, question));
        bytes32 question_id =
            keccak256(abi.encodePacked(content_hash, arbitrator, timeout, min_bond, address(this), msg.sender, nonce));
        require(questions[question_id].timeout == 0, "question must not exist");
        questions[question_id].content_hash = content_hash;
        questions[question_id].arbitrator = arbitrator;
        questions[question_id].opening_ts = opening_ts;
        questions[question_id].timeout = timeout;

        emit QuestionAsked(question);
        return question_id;
    }
```

Run the test snippet with `forge test --mt testAskDecisionQuestionWithOutcomesContainingInvalidChars`
