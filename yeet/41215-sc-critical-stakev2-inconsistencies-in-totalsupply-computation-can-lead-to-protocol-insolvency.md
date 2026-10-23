# #41215 \[SC-Critical] StakeV2: Inconsistencies in totalSupply computation, can lead to protocol insolvency

**Submitted on Mar 12th 2025 at 14:53:27 UTC by @max10afternoon for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41215
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Unstaking will decrease the totalSupply, by the amount being unstaken, but won't transfer the tokens until, the unstake process gets completed. This will effect the computation of accumulatedDeptRewardsYeet. Meaning that the unstaken amount will be distributable as reward, making the protocol insolvent.

## Vulnerability Details

The executeRewardDistributionYeet allows to redistribute Yeet tokens accumulated in the contract as reward. The amount that can be distributed gets computed by the accumulatedDeptRewardsYeet function:

```
   /// @notice The function used to distribute excess rewards to the vault.
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
        uint256 accRevToken0 = accumulatedDeptRewardsYeet();
        require(accRevToken0 > 0, "No rewards to distribute");
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

        stakingToken.approve(address(zapper), accRevToken0);
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        uint256 vaultSharesMinted;
        require(
            address(token0) == address(stakingToken) || address(token1) == address(stakingToken),
            "Neither token0 nor token1 match staking token"
        );

        if (address(token0) == address(stakingToken)) {
            (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams);
        } else {
            (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
        }

        _handleVaultShares(vaultSharesMinted);
        emit RewardsDistributedToken0(accRevToken0, rewardIndex);
    }
```

```
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

The unstaking process is divided in two parts startUnstake and either unstake or rageQuit. During startUnstake, the balance of the user, gets burnt and the totalSupply gets decreased:

```
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;

        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
        emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
    }
```

Than, possibly after a vesting period, the tokens gets transferred to the user:

```
  /// @notice The function used to unstake the tokens
    /// @param index The index of the vesting to unstake
    function _unstake(uint256 index) private {
        Vesting memory vesting = vestings[msg.sender][index];

        (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
        require(unlockedAmount != 0, "No unlocked amount");

        stakingToken.transfer(msg.sender, unlockedAmount);
        stakingToken.transfer(address(0x000000dead), lockedAmount);
        _remove(msg.sender, index);
        if (lockedAmount > 0) {
            emit RageQuit(msg.sender, unlockedAmount, lockedAmount, index);
        } else {
            emit Unstake(msg.sender, unlockedAmount, index);
        }
        stakedTimes[msg.sender]--;
    }
```

This means that after startUnstake gets called, the accumulatedDeptRewardsYeet will account for the unstaked amount as reward that can be distributed. Allowing the tokens to be sent to the underlying vault, and making the the contract insolvent, as it will not have enough liquidity to pay for the unstake finalization, while also distributing the rewards among other users.

#### Note on access control

The executeRewardDistributionYeet function, has a onlyManager modifier, that said, this issue doesn't arise from a wreckless or malicious behavior by the manager, but will also be caused by a regular usage of the function as the accumulatedDeptRewardsYeet function will advertise the wrong value as being available for distribution. In other words, the issue depends on a logical issue in the smart contract and not on an administrative account breaking any trust assumptions.

## Impact Details

Insolvency: The smart contract will advertise and use, more funds that what is has, leading to insolvency.

## Proof of Concept

To run the coded PoC, copy the following file, in the ./test folder, of the contest's github repository:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../src/StakeV2.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockWETH} from "./mocks/MockWBERA.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./mocks/SimpleZapperMock.sol";

contract KodiakVaultV1 {
    IERC20 public token0;
    IERC20 public token1;

    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0;
        token1 = _token1;
    }
}

abstract contract StakeV2_BaseTest {
    StakeV2 public stakeV2;
    MockERC20 public token;
    MockWETH public wbera;
    SimpleZapperMock public mockZapper;

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
        address owner = address(this);
        address manager = address(this);
        mockZapper = new SimpleZapperMock(token, wbera);
        stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
    }

    function test() public {}
}

contract StakeV2_HandleExcessDebt is Test {

    error ERC20InsufficientBalance(address, uint256, uint256);

    MockERC20 public token;
    MockWETH public wbera;

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
    }

   
    function test_handleExcessYeetToken() public {
        address owner = address(this);
        address manager = address(this);
        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        StakeV2 stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));
        mockZapper.setReturnValues(1, 1); 


        token.mint(address(this), 50 ether);
        token.approve(address(stakeV2), 50 ether);
        stakeV2.stake(50 ether);

    
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 0);

        stakeV2.startUnstake(50 ether);

        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 50 ether);
     
        

        assertEq(token.balanceOf(address(stakeV2)), 50 ether);

        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(50 ether, 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );

        assertEq(token.balanceOf(address(stakeV2)), 0);

        
        //Protocol became insolvent
        vm.expectRevert();
        stakeV2.rageQuit(0);


    }


  
}

```
