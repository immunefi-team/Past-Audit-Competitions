# #42020 \[SC-Critical] Inaccurate calculation in \`accumulatedDeptRewardsYeet()\` causes double counting of vesting tokens as excess, leading to permanent loss of user funds

**Submitted on Mar 20th 2025 at 04:47:18 UTC by @nnez for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42020
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Description

The `StakeV2` contract contains a vulnerability in the calculation of excess staking tokens available for reward distribution. The function `accumulatedDeptRewardsYeet()` incorrectly calculates the amount of excess staking tokens by simply subtracting `totalSupply` from the contract's token balance:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;
}
```

The core issue is that when users initiate unstaking via the `startUnstake()` function, the `totalSupply` is immediately reduced, but the tokens remain in the contract pending vesting:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    // ... validation checks ...
    balanceOf[msg.sender] -= unStakeAmount;
    totalSupply -= unStakeAmount;
    
    uint256 start = block.timestamp;
    uint256 end = start + VESTING_PERIOD;
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
    // ...
}
```

This creates a discrepancy where tokens that are already allocated for vesting (and thus belong to specific users) are incorrectly considered as "excess" by the `accumulatedDeptRewardsYeet()` function. When a manager calls `executeRewardDistributionYeet()` using this value, these vesting tokens may be swapped to vault tokens, effectively taking more tokens that belongs to staking balance not excess.

### Example scenario

1. The contract has 10,000 YEET tokens with `totalSupply` also at 10,000 YEET
2. A user initiates unstaking of 5,000 YEET, reducing `totalSupply` to 5,000 while the contract still holds 10,000 YEET
3. `accumulatedDeptRewardsYeet()` now returns 5,000 YEET (10,000 - 5,000)
4. A manager calls `executeRewardDistributionYeet()` and swaps these 5,000 tokens, the contract now has 5,000 YEET tokens
5. The vesting period ends, a user withdraw their unlocked tokens, now the contract has 5,000-5,000 = 0 YEET tokens
6. The remaining stakers won't be able to unstake their position because YEET tokens are depleted.

## Impact

This vulnerability creates contract insolvency as the system incorrectly treats vesting tokens as excess rewards. Over time, as tokens in vesting are double-counted and distributed as rewards, the contract will have insufficient funds to honor all unstaking requests. Early unstakers may receive their tokens, but later unstakers will find the contract depleted leaving remaining stakers with permanent loss of their funds.

## Recommended Mitigations

* Keep track of the total unvesting amount
* Use total unvesting amount in excess calculation

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - (totalSupply + totalUnvesting);
}
```

## Proof of Concept

## Proof-of-Concept

The following test demonstrate the described scenario in which remaining stakers cannot unstake due to insufficient balance of staking tokens in the contract.

### Steps

1. Add the following test in `StakeV2.test.sol`

```solidity
contract StakeV2_IncorrectExcess is Test {
    MockERC20 public token;
    MockWETH public wbera;

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
    }

    
    function test_incorrectExcessCalculation() public {
        address owner = address(this);
        address manager = address(this);
        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        StakeV2 stakeV2 = new StakeV2(token, mockZapper, owner, manager, IWETH(wbera));

        token.mint(address(this), 100 ether);
        token.approve(address(stakeV2), 100 ether);
        stakeV2.stake(100 ether);

        //zapper
        mockZapper.setReturnValues(1, 1);

        assertEq(100 ether, token.balanceOf(address(stakeV2)));
        
        // Assert that there is no excess
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 0);
        
        // Start unstaking, totalSupply is reduced but staking tokens remain in the contract
        stakeV2.startUnstake(50 ether);
        // Assert that staking tokens still remain
        assertEq(100 ether, token.balanceOf(address(stakeV2)));
        // Assert that totalSupply is reduced  
        assertEq(50 ether, stakeV2.totalSupply());
        
        // Assert that excess is incorrectly calculated from unstaking process  
        // (increase by reducing totalSupply)
        assertEq(stakeV2.accumulatedDeptRewardsYeet(), 50 ether);
        
        // Manager detects an excess and proceed to zap in as a vault token  
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(stakeV2.accumulatedDeptRewardsYeet(), 0, 0, address(0), ""),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );
        
        // Warp to end vesting time
        vm.warp(block.timestamp + 10 days + 1);  
        
        // Unstake vesting amount  
        stakeV2.unstake(0);
        // Assert that staking tokens are all depleted from the contract  
        assertEq(0 ether, token.balanceOf(address(stakeV2)));
        // Assert that there is still remaining stakers  
        assertEq(50 ether, stakeV2.totalSupply());
        
        // Remaining stakers try to unstake  
        stakeV2.startUnstake(50 ether);
        vm.warp(block.timestamp + 10 days + 1);  
        // Observe that unstake process fails due to insufficient balance of staking token within the contract  
        vm.expectRevert();
        stakeV2.unstake(0);
    }
}
```

2. Run `forge t --mc StakeV2_IncorrectExcess --mt test_incorrectExcessCalculation -vvvv`
3. Observe that the last unstake fails due to insufficent balance despite a positive totalSupply (indicating that there is remaining stakers in the contract).
