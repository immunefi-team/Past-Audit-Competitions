# 28946 - \[SC - Low] The assets accounting of the vault can become o...

Submitted on Mar 2nd 2024 at 18:26:27 UTC by @LokiThe5th for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28946

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `PufferVault`'s `receive` function does not accommodate a black swan scenario where executing queued Lido withdrawal requests can return lower than expected amounts of ETH (due to `stETH`slashing). This leads to desynchronization of the assets accounting and results in overreporting of the vault's assets during `deposit` and `mint` functions (and in the future `withdraw` and `redeem`).

## Vulnerability Details

The slashing of queued withdrawals is a known property of the Lido withdrawal process. In the event of such a slashing of queued withdrawals, the `PufferVault::totalAssets()` function will overstate the amount of `stETH` backing `pufETH`.

This is because when a withdrawal is queued with `initiateETHWithdrawalsFromLido` the function records the amount of `stETH` locked in the withdrawalqueue in the `lidoLockedETH` variable. When the withdrawal is claimed via `claimWithdrawalsFromLido`, the ETH is sent to the vault and the amount of ETH received is subtracted from the `lockedLidoAmount`. But in the event of slashing while the stETH is locked in the queue, the amount received from Lido will be less than the amount recorded in `lidoLockedETH`. Consequently the `lidoLockedETH` is not decreased by the expected amount.

## Impact Details

As this `lidoLockedETH` is used in the `totalAssets` calculation during deposits/mints (and withdrawals/redeems in the future), this would lead to users receiving less shares than they should for a given amount of assets.

There is no impact on withdrawals/redemptions yet, as this functionality is not yet present. If it was it would cause users to receive more assets in exchange for their shares than they should (as underlying assets are overcounted).

The Lido docs acknowledge that slashing of queued withdrawals can happen (although it hasn't happened yet and is considered low probability), hence the LOW severity submission.

Technically this would mean that `pufETH` could become undercollateralized, although the degree to which will be dependent on the magnitude of the event.

## Recommended Mitigation

This is a LOW severity issue, especially because the contract is not losing user assets immediately and appropriate action from the team can mitigate the issue.

In the event of a rebase such as this (which would be headline news in our industry), the team should pause deposits/mints to avoid users losing assets when entering the vault. Then an additional access restricted function can be added that allows the team to directly decrease the `lidoLockedETH` amount safely.

An alternative would be to keep track of all requested withdrawals and adjust the `lidoLockedETH` appropriately for any returned ETH that is lower than expected. Considering the contracts are already live, maybe this is something to consider in V2.

It is, however, strongly recommended to put mitigation plans in place for handling such an event, despite the low probability.

## References

* Lido docs on possible slashing events: <https://stake.lido.fi/withdrawals/request> (see "What is slashing?")

As per Lido docs: <https://help.lido.fi/en/articles/7858292-faq-ethereum-withdrawals>:

> The amount you can claim may differ from your initial request due to slashings and penalties. For these reasons, the total claimable reward amount could be lower than the amount withdrawn.

## Proof of Concept

The below files should be placed within the `puffEth` repo. The files can be found here: <https://drive.google.com/drive/folders/1M1dsrQ5plIr8xge9jL90-Q1PXiy5JGfl?usp=sharing>

Or can be retrieved from these secret gists:

* PoCSlashQueue.sol: <https://gist.github.com/lokithe5th/b57898156c37d7178e239199238da2af>
* stETHStrategyMock.sol: <https://gist.github.com/lokithe5th/90ffb119fd5c1ffb81dd35313b49091a>
* MockAccessManager.sol: <https://gist.github.com/lokithe5th/5c6f560b9587796d6d2894a9e1aabf57>
* LidoWithdrawalQueueMockExpanded: <https://gist.github.com/lokithe5th/7b606be1fbe2d25b3acfa5d3c52b4577>

Place the `PoCSlashQueue.sol` file in the `test` directory.\
Place the `MockAccessManager.sol`, `LidoWithdrawalQueueMockExpanded.sol` and the `stETHStrategyMock.sol`files in the `test/mocks` directory.

**Please ensure you comment out the `disableInitializers` line in the `PufferVault` constructor. We don't need to use a proxy for this PoC**

Run the test with `forge test --match-contract PoCSlashQueue -vvv`.

The console output:

```
Max Assets in system:  1000000000000000000000
Total Assets in vault before slash event and withdraw:  1000000000000000000000
Total stETH Vault thinks is still locked in Lido:  5000000
Expected Total Assets in vault after slash event and withdraw:  1000000000000000000000
Actual Total Assets in vault after slash event and withdraw:  999999999999995000000
```

Kindly note that the reduced amount is completely arbitrary. It will depend entirely on the magnitude of the slashing event.
