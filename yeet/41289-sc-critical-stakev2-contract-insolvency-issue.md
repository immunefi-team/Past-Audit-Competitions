# #41289 \[SC-Critical] StakeV2 Contract Insolvency Issue

**Submitted on Mar 13th 2025 at 10:59:06 UTC by @rajkaur for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41289
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

During unstaking, totalSupply immediately decreases, but tokens remain in the contract, resulting in incorrect rewards. Users can claim these as "rewards," making the contract insolvent and preventing some users from withdrawing funds.

## Vulnerability Details

The main issue is in how contract calculate rewards. Look at this function:\
When user call startUnstake(), the function reduce totalSupply but tokens still in contract:

```
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return token.balanceOf(address(this)) - totalSupply;
}
```

This create big problem because contract think it has extra tokens (rewards) but actually these token are owed to users who are unstaking. Then manager can call executeRewardDistributionYeet() and take these "rewards" out of contract.

```
function startUnstake(uint256 amount) external {
    _burn(msg.sender, amount);  // This reduces totalSupply
}
```

\##POC

Initial state:\
Contract balance: 100 tokens\
Total supply: 100 tokens\
Accumulated rewards: 0 tokens

After 8 users start unstaking:\
Contract balance: 100 tokens\
Total supply: 20 tokens\
Accumulated rewards: 80 tokens

After distributing rewards:\
Contract balance: 40 tokens\
Total supply: 20 tokens\
Accumulated rewards: 20 tokens

User 0 unstaked successfully\
User 1 unstaked successfully\
User 2 unstaked successfully\
User 3 unstaked successfully

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "../src/StakeV2.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockWETH} from "./mocks/MockWBERA.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./mocks/SimpleZapperMock.sol";
import "../src/interfaces/IZapper.sol";

contract KodiakVaultV1Mock {
    IERC20 public token0;
    IERC20 public token1;

    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0;
        token1 = _token1;
    }
}

contract testAccountingBug is Test {
    StakeV2 public stakeV2;
    MockERC20 public token;
    MockWETH public wbera;
    SimpleZapperMock public mockZapper;
    KodiakVaultV1Mock public kodiakVault;
    
    address public owner;
    address public manager;
    address[] public users;
    
    uint256 constant NUM_USERS = 10;
    uint256 constant STAKE_AMOUNT = 10 ether;
    
    function setUp() public {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
        owner = address(this);
        manager = address(this);
        mockZapper = new SimpleZapperMock(token, wbera);
        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
        kodiakVault = new KodiakVaultV1Mock(token, IERC20(address(wbera)));
        
        mockZapper.setReturnValues(1, 1);
        
        for (uint256 i = 0; i < NUM_USERS; i++) {
            users.push(address(uint160(0x1000 + i)));
            token.mint(users[i], STAKE_AMOUNT);
            
            vm.startPrank(users[i]);
            token.approve(address(stakeV2), STAKE_AMOUNT);
            vm.stopPrank();
        }
    }
    
    function test_unstakingVulnerability() public {
        for (uint256 i = 0; i < NUM_USERS; i++) {
            vm.startPrank(users[i]);
            stakeV2.stake(STAKE_AMOUNT);
            vm.stopPrank();
        }
        
        assertEq(token.balanceOf(address(stakeV2)), NUM_USERS * STAKE_AMOUNT, "Contract should have all staked tokens");
        assertEq(stakeV2.totalSupply(), NUM_USERS * STAKE_AMOUNT, "Total supply should match staked amount");
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 0, "No excess rewards initially");
        
        console.log("Initial state:");
        console.log("  Contract balance:", token.balanceOf(address(stakeV2)) / 1 ether, "tokens");
        console.log("  Total supply:", stakeV2.totalSupply() / 1 ether, "tokens");
        console.log("  Accumulated rewards:", stakeV2.accumulatedDeptRewardsYeet() / 1 ether, "tokens");
        
        console.log("\nStep 2: 8 users start unstaking");
        for (uint256 i = 0; i < 8; i++) {
            vm.startPrank(users[i]);
            stakeV2.startUnstake(STAKE_AMOUNT);
            vm.stopPrank();
        }
        
        assertEq(token.balanceOf(address(stakeV2)), NUM_USERS * STAKE_AMOUNT, "Contract should still have all tokens");
        assertEq(stakeV2.totalSupply(), NUM_USERS * STAKE_AMOUNT - 8 * STAKE_AMOUNT, "Total supply should be reduced");
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 8 * STAKE_AMOUNT, "80% of tokens now appear as rewards");
        
        console.log("After 8 users start unstaking:");
        console.log("  Contract balance:", token.balanceOf(address(stakeV2)) / 1 ether, "tokens");
        console.log("  Total supply:", stakeV2.totalSupply() / 1 ether, "tokens");
        console.log("  Accumulated rewards:", stakeV2.accumulatedDeptRewardsYeet() / 1 ether, "tokens");
        
        console.log("\nStep 3: Manager distributes rewards");
        
        uint256 distributionAmount = stakeV2.accumulatedDeptRewardsYeet() * 75 / 100;
        
        IZapper.SingleTokenSwap memory swap = IZapper.SingleTokenSwap({
            inputAmount: distributionAmount,
            outputQuote: 0,
            outputMin: 0,
            executor: address(0),
            path: ""
        });
        
        IZapper.KodiakVaultStakingParams memory stakingParams = IZapper.KodiakVaultStakingParams({
            kodiakVault: address(kodiakVault),
            amount0Max: 0,
            amount1Max: 0,
            amount0Min: 0,
            amount1Min: 0,
            amountSharesMin: 0,
            receiver: address(0)
        });
        
        IZapper.VaultDepositParams memory vaultParams = IZapper.VaultDepositParams({
            vault: address(token),
            receiver: address(this),
            minShares: 0
        });
        
        stakeV2.executeRewardDistributionYeet(swap, stakingParams, vaultParams);
        
        uint256 expectedRemainingBalance = NUM_USERS * STAKE_AMOUNT - distributionAmount;
        assertEq(token.balanceOf(address(stakeV2)), expectedRemainingBalance, "Contract should have distributed rewards");
        
        console.log("After distributing rewards:");
        console.log("  Contract balance:", token.balanceOf(address(stakeV2)) / 1 ether, "tokens");
        console.log("  Total supply:", stakeV2.totalSupply() / 1 ether, "tokens");
        console.log("  Accumulated rewards:", stakeV2.accumulatedDeptRewardsYeet() / 1 ether, "tokens");
        
        vm.warp(block.timestamp + 11 days);
        
        console.log("\nStep 4: Users try to unstake");
        
        uint256 successfulUnstakes = 0;
        uint256 failedUnstakes = 0;
        
        for (uint256 i = 0; i < 8; i++) {
            vm.startPrank(users[i]);
            
            try stakeV2.unstake(0) {
                successfulUnstakes++;
                console.log(string(abi.encodePacked("User ", vm.toString(i), " unstaked successfully")));
                console.log("  User balance:", token.balanceOf(users[i]) / 1 ether, "tokens");
            } catch Error(string memory reason) {
                failedUnstakes++;
                console.log(string(abi.encodePacked("User ", vm.toString(i), " FAILED to unstake: ", reason)));
            }
            
            vm.stopPrank();
        }
        
        console.log("\nFinal state:");
        console.log("  Contract balance:", token.balanceOf(address(stakeV2)) / 1 ether, "tokens");
        console.log("  Total supply:", stakeV2.totalSupply() / 1 ether, "tokens");
        console.log("  Users who successfully unstaked:", successfulUnstakes);
        console.log("  Users who couldn't unstake:", failedUnstakes);
        
        uint256 remainingDebt = (8 - successfulUnstakes) * STAKE_AMOUNT;
        bool isInsolvent = token.balanceOf(address(stakeV2)) < remainingDebt;
        
        console.log("  Tokens owed to remaining users:", remainingDebt / 1 ether, "tokens");
        console.log("  Tokens available in contract:", token.balanceOf(address(stakeV2)) / 1 ether, "tokens");
        console.log("Contract is insolvent:", isInsolvent ? "YES" : "NO");
        
        assertTrue(failedUnstakes > 0, "Some unstaking operations should fail");
        assertTrue(isInsolvent, "Contract should be insolvent");
        assertLt(token.balanceOf(address(stakeV2)), remainingDebt, "Contract should have fewer tokens than it owes");
    }
} 
```
