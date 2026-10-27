# 58488 sc low tokeautousdstrategy claims rewards to itself automatically when deallocate is called but since reward token is tokemak the rewards remain permanently locked

**Submitted on Nov 2nd 2025 at 17:11:32 UTC by @niroh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58488
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The TokeAutoUSDStrategy collects rewards from the Token Auto Rewarder contract automatically whenever \_deallocate is called. This is done by setting the rewarder.withdraw() last parameter to true:

```solidity
//From TokeAutoUSDStrategy::_deallocate()...

 // withdraw shares, claim any rewards
rewarder.withdraw(address(this), sharesNeeded, true);
```

Internally the rewarder claims the reward to the sender by setting the second parameter of \_processRewards (recipient) to the claiming account

```solidity
//From MainRewarder _withdraw
  if (claim) {
        _processRewards(account, account, true);
    }
```

The reward token of Rewarder is Tokemak (0x2e9d63788249371f1DFC918a52f8d799F4a38C94). Therefore, rewards are claimed as Tokemak balance added to the strategy.

## Vulnerability Details

Since the TokeAutoUSDStrategy doesn't have any swipe function (or low level call functionality) there is no way to move, use or trade the Tokemak tokens accumulating in the strategy. This means all collected rewards remain permanently frozen.

The same is true for the TokeAutoEth strategy, with one difference which is that TokeAutoEth implements \_claimRewards which enables anyone to call claimRewards on it directly. Hoever its implementation sends the gained Tokemak to the MYT vault: (`rewarder.getReward(address(this), address(MYT), false);`). Since it's MYT vault token is Eth, the tokens remain unreachable in this case as well (only in the MYT vault instead of the strategy).

## Impact Details

Permanent Freezing of all rewards collected by the TokeAutoUSDStrategy and TokeAutoEth strategy

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/AaveV3OPUSDCStrategy.sol#L45>

## Proof of Concept

## Proof of Concept

How to run:

1. Copy the code below into the TokeAutoUSDStrategyTest contract in v3-poc/src/test/strategies/AaveV3ARBUSDCStrategy.t.sol
2. Add the following imports and interfaces at the top of the file:

```solidity
import  "../libraries/BaseStrategyTest.sol";
interface IAccessController  {
    function grantRole(bytes32 role, address account) external;
    function getRoleAdmin(bytes32 role) external view returns (bytes32);
    function owner() external view  returns (address);
    function getRoleMember(bytes32 role, uint256 index) external view returns (address);
}

interface IMainRewarder  {
    function rewardToken() external view returns (IERC20);
    function queueNewRewards(uint256 newRewards) external ;
    function accessController() external view returns (IAccessController controller);
}
```

3. run with `FOUNDRY_PROFILE=default forge test --fork-url https://mainnet.gateway.tenderly.co --match-test testRewardUnclaimable -vvv`

```solidity
function testRewardUnclaimable() public {

        uint256 initialDeposit = getTestConfig().vaultInitialDeposit;
        uint256 amountFirstAllocation = initialDeposit / 2;

        vm.startPrank(allocator);
        uint256 oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
        bytes memory dataOldAlloc = abi.encode(oldAllocation);
        VaultV2(vault).allocate(strategy, dataOldAlloc, amountFirstAllocation);
        vm.stopPrank();

        //emulate reward awarding to autoToken
        bytes32  LIQUIDATOR_MANAGER = keccak256("LIQUIDATOR_ROLE");
        IAccessController ac = IMainRewarder(REWARDER).accessController();
        address whitelisted = ac.getRoleMember(LIQUIDATOR_MANAGER,0);


        uint256 rewardAmount = 100e18;
        IERC20 rewardToken = IMainRewarder(REWARDER).rewardToken();

        uint256 rewardBal = rewardToken.balanceOf(strategy);
        console.log("srategy tokemak before: %s",rewardBal);

        deal(address(rewardToken),whitelisted,rewardAmount);
        vm.startPrank(whitelisted);
        rewardToken.approve(REWARDER,rewardAmount);
        IMainRewarder(REWARDER).queueNewRewards(rewardAmount);
        vm.stopPrank();
        vm.warp(block.timestamp + 60*60*24);
        vm.roll(block.number + 60*60*24);


        //deallocate
         vm.startPrank(allocator);
        oldAllocation = VaultV2(vault).allocation(IMYTStrategy(strategy).adapterId());
        dataOldAlloc = abi.encode(oldAllocation);
        VaultV2(vault).deallocate(strategy, dataOldAlloc, amountFirstAllocation);
        vm.stopPrank();

        rewardBal = rewardToken.balanceOf(strategy);
        console.log("srategy tokemak: %s",rewardBal);


    }
```
