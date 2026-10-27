# #39528 \[SC-Insight] Lack of Validation for Min and Max Values in FlatCFMFactory leads to wrong payou

**Submitted on Jan 31st 2025 at 22:06:32 UTC by @NHristov for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39528
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Min and max values are not validated when creating a FlatCFM, allowing minValue == maxValue. This results in the scalar market incorrectly resolving all payout to the short tokens if the numeric answer equals that boundary.

## Vulnerability Details

Because both min and max can be the same when creating a FlatCFM generic question params for the conditional scalar markets the metric question resolution interprets that numeric answer as valid for short tokens.

```solidity
function createFlatCFM(
        FlatCFMOracleAdapter oracleAdapter,
        uint256 decisionTemplateId,
        uint256 metricTemplateId,
        FlatCFMQuestionParams calldata flatCFMQParams,
        GenericScalarQuestionParams calldata genericScalarQuestionParams,
        IERC20 collateralToken,
        string calldata metadataUri
    ) external payable returns (FlatCFM cfm) {
        uint256 outcomeCount = flatCFMQParams.outcomeNames.length;
        if (outcomeCount == 0 || outcomeCount > MAX_OUTCOME_COUNT) {
            revert InvalidOutcomeCount();
        }
        for (uint256 i = 0; i < outcomeCount; i++) {
            string memory outcomeName = flatCFMQParams.outcomeNames[i];
            if (bytes(outcomeName).length > MAX_OUTCOME_NAME_LENGTH) revert InvalidOutcomeNameLength(outcomeName);
        }

        cfm = FlatCFM(flatCfmImplementation.clone());

        bytes32 decisionQuestionId =
            oracleAdapter.askDecisionQuestion{value: msg.value}(decisionTemplateId, flatCFMQParams);

        // +1 for 'Invalid' slot.
        bytes32 decisionConditionId =
            conditionalTokens.getConditionId(address(cfm), decisionQuestionId, outcomeCount + 1);
        if (conditionalTokens.getOutcomeSlotCount(decisionConditionId) == 0) {
            conditionalTokens.prepareCondition(address(cfm), decisionQuestionId, outcomeCount + 1);
        }

        paramsToDeploy[cfm] = DeploymentParams({
            collateralToken: collateralToken,
            metricTemplateId: metricTemplateId,
            genericScalarQuestionParams: genericScalarQuestionParams,
            decisionConditionId: decisionConditionId,
            outcomeNames: flatCFMQParams.outcomeNames
        });

        cfm.initialize(oracleAdapter, conditionalTokens, outcomeCount, decisionQuestionId, metadataUri);

        emit FlatCFMCreated(address(cfm), decisionConditionId);
    }
```

When we call `FlatCFMFactory::createConditionalScalarMarket` to create the conditional scalar market everything will work as expected. However, once the answer is provided by the oracle and if the numeric answer equals the same value as the min and max (since they are equal), the payouts will be received only from the short tokens.

```solidity
function resolve() external {
        bytes32 answer = oracleAdapter.getAnswer(ctParams.questionId);
        uint256[] memory payouts = new uint256[](3);

        if (oracleAdapter.isInvalid(answer)) {
            // 'Invalid' outcome receives full payout
            payouts[2] = 1;
        } else {
            uint256 numericAnswer = uint256(answer);
            if (numericAnswer <= scalarParams.minValue) {
                payouts[0] = 1; // short
            } else if (numericAnswer >= scalarParams.maxValue) {
                payouts[1] = 1; // long
            } else {
                payouts[0] = scalarParams.maxValue - numericAnswer;
                payouts[1] = numericAnswer - scalarParams.minValue;
            }
        }
        conditionalTokens.reportPayouts(ctParams.questionId, payouts);
    }
```

## Impact Details

Funds can be diverted solely to the short outcome, causing mispricing and potentially incorrect compensation for other positions if exploited to the bug.

## References

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/FlatCFMFactory.sol#L103C5-L144C6>

<https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/ConditionalScalarMarket.sol#L72C5-L91C6>

## Proof of Concept

From the unit test folder, in the `CondtionalScalarMarket.t.sol` paste the following code to prove that if the min and max values are the same and only short tokens will receive payouts

```solidity

contract WrongMinMaxTest is Test {
    FlatCFMOracleAdapter oracleAdapter;
    IConditionalTokens conditionalTokens;
    IWrapped1155Factory wrapped1155Factory;

    ConditionalScalarMarket csm;

    IERC20 collateralToken;
    IERC20 shortToken;
    IERC20 longToken;
    IERC20 invalidToken;

    address constant USER = address(0x1111);

    uint256 constant DEAL = 10;
    bytes32 constant QUESTION_ID = bytes32("some question id");
    bytes32 constant CONDITION_ID = bytes32("some condition id");
    bytes32 constant PARENT_COLLECTION_ID = bytes32("someParentCollectionId");
    // set the min valud to be equal to the max value
    uint256 constant MIN_VALUE = 11000;
    uint256 constant MAX_VALUE = 11000;

    function setUp() public virtual {
        // 1. Deploy or mock the external dependencies
        oracleAdapter = new DummyFlatCFMOracleAdapter();
        conditionalTokens = new DummyConditionalTokens();
        wrapped1155Factory = new DummyWrapped1155Factory();
        collateralToken = new DummyERC20("Collateral", "COL");
        shortToken = new DummyERC20("Short", "ST");
        longToken = new DummyERC20("Long", "LG");
        invalidToken = new DummyERC20("Invalid", "XX");

        // 2. Deploy the ConditionalScalarMarket
        csm = new ConditionalScalarMarket();
        csm.initialize(
            oracleAdapter,
            conditionalTokens,
            wrapped1155Factory,
            ConditionalScalarCTParams({
                questionId: QUESTION_ID,
                conditionId: CONDITION_ID,
                parentCollectionId: PARENT_COLLECTION_ID,
                collateralToken: collateralToken
            }),
            ScalarParams({minValue: MIN_VALUE, maxValue: MAX_VALUE}),
            WrappedConditionalTokensData({
                shortData: "",
                longData: "",
                invalidData: "",
                shortPositionId: 1,
                longPositionId: 2,
                invalidPositionId: 2,
                wrappedShort: shortToken,
                wrappedLong: longToken,
                wrappedInvalid: invalidToken
            })
        );
    }

    function testResolveWhenMinAndMaxAreEqualAndResultIsTheSame() public {
        uint256 answer = MIN_VALUE;

        uint256[] memory expectedPayout = new uint256[](3);
        // even when the result is the same as the max value
        // only the short position will receive payout
        // the long position will receive 0 as payout
        expectedPayout[0] = 1;
        expectedPayout[1] = 0;
        expectedPayout[2] = 0;

        vm.mockCall(
            address(oracleAdapter),
            abi.encodeWithSelector(FlatCFMOracleAdapter.getAnswer.selector, QUESTION_ID),
            abi.encode(answer)
        );

        vm.expectCall(
            address(conditionalTokens),
            abi.encodeWithSelector(IConditionalTokens.reportPayouts.selector, QUESTION_ID, expectedPayout)
        );
        csm.resolve();
    }
}
```

and to prove that a cfm can be created with equal min and max generic scalar parameters paste this at the bottom of the file `FlatCFMFactory.t.sol`

```solidity

contract CreateMarketWithBadMinMaxAmount is Base {
    string[] outcomeNames;
    uint256 constant DECISION_TEMPLATE_ID = 42;
    uint256 constant METRIC_TEMPLATE_ID = 442;
    uint32 constant DECISION_OPENING_TIME = 1739577600; // 2025-02-15
    string constant ROUND_NAME = "round";
    string constant METRIC_NAME = "metric";
    string constant START_DATE = "2025-02-16";
    string constant END_DATE = "2025-06-16";
    uint256 constant MIN_VALUE = 1000000;
    uint256 constant MAX_VALUE = 1000000;
    uint32 constant METRIC_OPENING_TIME = 1750118400; // 2025-06-17
    string METADATA_URI = "ipfs://sfpi";

    bytes32 constant DECISION_QID = bytes32("decision question id");
    //bytes32 constant DECISION_CID = bytes32("decision condition id");
    bytes32 constant METRIC_QID = bytes32("conditional question id");
    //bytes32 constant METRIC_CID = bytes32("conditional condition id");
    bytes32 constant OUTCOME1_PARENT_COLLEC_ID = bytes32("cond 1 parent collection id");
    bytes32 constant SHORT_COLLEC_ID = bytes32("short collection id");
    uint256 constant SHORT_POSID = uint256(bytes32("short position id"));
    bytes32 constant LONG_COLLEC_ID = bytes32("long collection id");
    uint256 constant LONG_POSID = uint256(bytes32("long position id"));
    bytes32 constant INVALID_COLLEC_ID = bytes32("invalid collection id");
    uint256 constant INVALID_POSID = uint256(bytes32("invalid position id"));

    FlatCFMQuestionParams decisionQuestionParams;
    GenericScalarQuestionParams genericScalarQuestionParams;

    event FlatCFMCreated(address indexed market);
    event ConditionalMarketCreated(
        address indexed decisionMarket, address indexed conditionalMarket, uint256 outcomeIndex
    );

    function setUp() public virtual override {
        super.setUp();

        outcomeNames.push("Project A");
        outcomeNames.push("Project B");
        outcomeNames.push("Project C");
        outcomeNames.push("Project D");

        decisionQuestionParams = FlatCFMQuestionParams({outcomeNames: outcomeNames, openingTime: DECISION_OPENING_TIME});

        genericScalarQuestionParams = GenericScalarQuestionParams({
            scalarParams: ScalarParams({minValue: MIN_VALUE, maxValue: MAX_VALUE}),
            openingTime: METRIC_OPENING_TIME
        });
    }

    function testCreatesConditionalMarketsFromCFMWithMaxAndMinEqual() public {
        FlatCFM cfm = factory.createFlatCFM(
            oracleAdapter,
            DECISION_TEMPLATE_ID,
            METRIC_TEMPLATE_ID,
            decisionQuestionParams,
            genericScalarQuestionParams,
            collateralToken,
            METADATA_URI
        );

        vm.recordLogs();
        for (uint256 i = 0; i < outcomeNames.length; i++) {
            ConditionalScalarMarket result = factory.createConditionalScalarMarket(cfm);
            (uint256 minValue, uint256 maxValue) = result.scalarParams();

            assertEq(minValue, maxValue);
        }
    }
```
