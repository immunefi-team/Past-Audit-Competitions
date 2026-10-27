# 39495 \[SC-Low] flatcfm cannot be resolved in case answer of questionid are in greater or equal to 2

## #39495 \[SC-Low] FlatCFM cannot be resolved in case answer of questionId are in greater or equal to 2^OUTCOME\_COUNT and answer % 2^OUTCOME\_COUNT is 0

**Submitted on Jan 31st 2025 at 07:22:46 UTC by @perseverance for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39495
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Temporary freezing of funds for at least 1 hour

### Description

### Brief/Intro

## Description

The FlatCFM can be resolved by getting the answer from RealityETH Oracle for the questionId. When the answer from RealityETH is finalized, anyone can call resolve() to resolve the FlatCFM. It is important that FlatCFM should be resolved so that users can redeem to get the collateral (money) back.

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/src/FlatCFM.sol#L58>

```solidity

function resolve() external {}

```

The answer from RealityETH Oracle are submitted by anyone and the arbitrator if requested via the RealityETH Contract.

### The vulnerability

#### Vulnerability Details

In the resolve() function, the contract taken into consideration the invalid case or the case answer is 0 then the FlatCFM is resolved as Invalid Position receive Full Payout.

In an edge case, if the answer is in **greater or equal to 2^OUTCOME\_COUNT and answer % 2^OUTCOME\_COUNT is 0**

In this case all the payouts\[i] will be 0 .

```solidity
For example, the outcomeCount = 50. 

answer = 2**50  => all payouts[i] = 0 

or answer = 2**51 => all payout[i] = 0 


the outcomeCount = 255

anwswer = 2**255 => all payouts[i] = 0

```

```solidity

function resolve() external {
        bytes32 answer = oracleAdapter.getAnswer(questionId);
        uint256[] memory payouts = new uint256[](outcomeCount + 1);
        uint256 numericAnswer = uint256(answer);

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

**In this case, the call to conditionalTokens.reportPayouts will be reverted because of errors "payout is all zeroes"**

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/test/integration/vendor/gnosis/conditional-tokens-contracts/ConditionalTokens.sol#L101>

```solidity

function reportPayouts(bytes32 questionId, uint256[] calldata payouts) external {
        uint256 outcomeSlotCount = payouts.length;
        require(outcomeSlotCount > 1, "there should be more than one outcome slot");
        // IMPORTANT, the oracle is enforced to be the sender because it's part of the hash.
        bytes32 conditionId = CTHelpers.getConditionId(msg.sender, questionId, outcomeSlotCount);
        require(payoutNumerators[conditionId].length == outcomeSlotCount, "condition not prepared or found");
        require(payoutDenominator[conditionId] == 0, "payout denominator already set");

        uint256 den = 0;
        for (uint256 i = 0; i < outcomeSlotCount; i++) {
            uint256 num = payouts[i];
            den = den.add(num);

            require(payoutNumerators[conditionId][i] == 0, "payout numerator already set");
            payoutNumerators[conditionId][i] = num;
        }
        require(den > 0, "payout is all zeroes");
        payoutDenominator[conditionId] = den;
        emit ConditionResolution(conditionId, msg.sender, questionId, outcomeSlotCount, payoutNumerators[conditionId]);
    }

```

Since the answer is provided by users and arbitration with different scenarios with different actors, although it might be rare situation but still it can happen that answer can be out of range as described above.

If this happened, then the FlatCFM cannot be resolved. Since the question answer might be already finalized when users or the project notice this error, then nothing can be done to provide to correct answer.

Since the FlatCFM is not resolved, then users cannot redeem to get back the token. It will be very complicated situation to handle.

It is better to take this scenario into consideration and prevent it now to avoid this situation. For these scenarios, the FlatCFM can be resolved as Invalid to receive full payout.

## Impacts

## About the severity assessment

Impact of this bug report is that FlatCFM cannot be resolved then users cannot redeem to get back the token.

It might cause Temporary freezing of funds for at least 1 hours

To get back the money, users can merge the tokens, but it is complicated situation and will be difficult to handle since it is related to many users as the tokens are circulating.

Impact severity: at least High

But since this is an edge case, so the likelyhood of this issue might be Low.

**In total , I think the bug report can be Medium or Low**

I map it into the closet impact listed:

Contract fails to deliver promised returns, but doesn't lose value

But I believe that the severity can be at least Medium, but I let the project and Immunefi team to decide upon that.

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/89d26594d88080b62121184be87ba7a0>

### Proof of Concept

## Proof of concept

Test code to show the bug

```solidity

function testResolveWrongAnswerCallsReportPayoutsWithOutofRange() public {
        uint256[] memory plainAnswer = new uint256[](OUTCOME_COUNT + 2);
        plainAnswer[0] = 0;
        plainAnswer[OUTCOME_COUNT - 1] = 0;
        plainAnswer[OUTCOME_COUNT] = 1;
        bytes32 answer = _toBitArray(plainAnswer);

        uint256[] memory expectedPayout = new uint256[](OUTCOME_COUNT + 1);
        
        vm.mockCall(
            address(oracleAdapter),
            abi.encodeWithSelector(FlatCFMOracleAdapter.getAnswer.selector, QUESTION_ID),
            abi.encode(answer)
        );
        vm.expectRevert("payout is all zeroes");
        cfm.resolve();
    }

```

Explanation:

The OUTCOME\_COUNT is 50

Set the answer is 2^50

Expect the call to resolve to revert with error: "payout is all zeroes"

Copy the test code into the Unit test:

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/test/unit/FlatCFM.t.sol>

Also modify a bit the DummyConditionalTokens to behave like the real conditionalTokens

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/test/unit/dummy/ConditionalTokens.sol#L86-L88>

```solidity
function reportPayouts(bytes32 questionId, uint256[] calldata payouts) external override {
        _test_reportPayouts_caller[questionId] = msg.sender;
        
        uint256 outcomeSlotCount = payouts.length;
        
        require(outcomeSlotCount > 1, "there should be more than one outcome slot");

        uint256 den = 0;
        for (uint256 i = 0; i < outcomeSlotCount; i++) {
            uint256 num = payouts[i];
            den += num;
            
        }
        require(den > 0, "payout is all zeroes");

    }
```

After that, run the test

```bash
forge test --match-test testResolveWrongAnswerCallsReportPayoutsWithOutofRange -vvvvv

```

Test Log:

```log
[91899] TestResolve::testResolveWrongAnswerCallsReportPayoutsWithOutofRange()
    ├─ [0] VM::mockCall(DummyFlatCFMOracleAdapter: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0x8a4599c7736f6d65207175657374696f6e20696400000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000004000000000000)
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(payout is all zeroes)
    │   └─ ← [Return] 
    ├─ [68804] FlatCFM::resolve()
    │   ├─ [0] DummyFlatCFMOracleAdapter::getAnswer(0x736f6d65207175657374696f6e20696400000000000000000000000000000000) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000004000000000000
    │   ├─ [324] DummyFlatCFMOracleAdapter::isInvalid(0x0000000000000000000000000000000000000000000000000004000000000000) [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [36287] DummyConditionalTokens::reportPayouts(0x736f6d65207175657374696f6e20696400000000000000000000000000000000, [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0])
    │   │   └─ ← [Revert] revert: payout is all zeroes
    │   └─ ← [Revert] revert: payout is all zeroes
    └─ ← [Stop] 
```

### Full POC:

1. Replace <https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/test/unit/dummy/ConditionalTokens.sol>

with

<https://gist.github.com/Perseverancesuccess2021/89d26594d88080b62121184be87ba7a0#file-conditionaltokens-sol>

2. Replace <https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/test/unit/FlatCFM.t.sol>

With

<https://gist.github.com/Perseverancesuccess2021/89d26594d88080b62121184be87ba7a0#file-flatcfm-t-sol>

After that, run the test

```bash
forge test --match-test testResolveWrongAnswerCallsReportPayoutsWithOutofRange -vvvvv
```

Full Log:

<https://gist.github.com/Perseverancesuccess2021/89d26594d88080b62121184be87ba7a0#file-test\\_testresolvewronganswercallsreportpayoutswithoutofrange\\_250131\\_1120-log>
