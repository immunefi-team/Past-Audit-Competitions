# 28613 - \[SC - Medium] User will lose funds

Submitted on Feb 22nd 2024 at 14:16:18 UTC by @shadowHunter for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28613

Report type: Smart Contract

Report severity: Medium

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In case of slashing, Lido `claimWithdrawal` will give discounted value which is lesser than expected ETH. This causes huge problem since `$.lidoLockedETH` does not account for discount, causing `totalAssets` to become higher than required. This indirectly causes share prices to become higher since share price increases with increased `totalAssets`

## Vulnerability Details

1. Lets say User deposited 1000 StEth to Puffer Vault (I assume this will be normally done via PufferVaultMainnet)
2. Since this is first deposit, User gets 1000 shares (total asset and supply being 0)

```
function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
        return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
    }
```

3. Now Owner initiate withdrawal for all 1000 StETH using `initiateETHWithdrawalsFromLido` function, transferring all 1000 StEth to Lido
4. This makes `$.lidoLockedETH=1000` and returns a request id say 1

```
for (uint256 i = 0; i < amounts.length; ++i) {
            lockedAmount += amounts[i];
        }
        $.lidoLockedETH += lockedAmount;
```

5. Owner claims the request id using `claimWithdrawalsFromLido` function
6. Slashing occurs and withdrawals get discounted by Lido (<https://github.com/lidofinance/lido-dao/blob/master/contracts/0.8.9/WithdrawalQueueBase.sol#L472C8-L477C49>)

```
 uint256 ethWithDiscount = _calculateClaimableEther(request, _requestId, _hint);
        // because of the stETH rounding issue
        // (issue: https://github.com/lidofinance/lido-dao/issues/442 )
        // some dust (1-2 wei per request) will be accumulated upon claiming
        _setLockedEtherAmount(getLockedEtherAmount() - ethWithDiscount);
        _sendValue(_recipient, ethWithDiscount);
```

6. Lets say due to slashing 10% got deducted which means 900 eth gets returned for the claim made in Step 5
7. So `$.lidoLockedETH` is updated as 1000-900=100eth

```
receive() external payable virtual {
        // If we don't use this pattern, somebody can create a Lido withdrawal, claim it to this contract
        // Making `$.lidoLockedETH -= msg.value` revert
        VaultStorage storage $ = _getPufferVaultStorage();
        if ($.isLidoWithdrawal) {
            $.lidoLockedETH -= msg.value;
        }
    }
```

8. `getPendingLidoETHAmount` still gives 100 since $.lidoLockedETH is still 100. Also, If we check `totalAssets`, it still gives 1000 amount (900 eth+100 steth pending) even though nothing is pending on Lido now
9. This becomes a problem when next depositor deposit since if he deposits 1000 eth, he gets 1000 shares instead of 1111

```
Original:
1000*1000/1000=1000

Expected:
1000*1000/900= 1111
```

## Impact Details

User will pay more for shares than required.

## Recommendation

Track the delta slashing amount and deduct the same from `$.lidoLockedETH` for correct accounting

## Proof of Concept

```
function test_poc() public {
        stETHMock(address(stETH)).mint(address(this), 2000);
        stETH.approve(address(pufferVault), type(uint256).max);
		assertEq(pufferVault.deposit(1000, address(this)), 1000, "deposit");
		uint256[] memory arr=new uint256[](1);
		arr[0]=1000;
		// Removed restricted modifier for simplicity
		pufferVault.initiateETHWithdrawalsFromLido(arr);
		
		// Lets say due to slashing Lido claim gave 0 amount
		// $.lidoLockedETH still remains 1000 wei causing below issue
		
		// Fails since totalAssets still gives 2000 wei (mock lido request withdrawal does not actually transfer steth) even though they were lost due to slashing
		// Below gave 500 shares instead of 1000
		assertEq(pufferVault.deposit(1000, address(this)), 1000, "deposit");

    }
```
