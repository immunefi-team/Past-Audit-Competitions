# 28788 - \[SC - Critical] Slash during a withdrawal from EigenLayer will ...

Submitted on Feb 27th 2024 at 08:04:12 UTC by @dontonka for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28788

Report type: Smart Contract

Report severity: Critical

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

There is an edge case that can occur during EigenLayer withdrawal process that can `break some key invariants` of the PufferVault contract. Essentially, if the `queuedWithdrawal` is being **slashed** in the middle of the withdrawal process (which is a 2-step process), PufferVault's accounting will be in a `broken state` and not recoverable, which can lead to unexpected results and most likely to `protocol insolvency` and seem to warrant `Critical` severity.

## Vulnerability Details

The EigenLayer withdrawal process is a 2-step process, so first a call to `initiateStETHWithdrawalFromEigenLayer` is done and later on `claimWithdrawalFromEigenLayer` to actually complete the withdrawal and receive the funds. If somehow `_EIGEN_STRATEGY_MANAGER.slashQueuedWithdrawal` is called successfully against the `queuedWithdrawal` generated when calling initiateStETHWithdrawalFromEigenLayer, this will cause the problem, as `claimWithdrawalFromEigenLayer` will always revert afterward for this queuedWithdrawal, which mean `$.eigenLayerPendingWithdrawalSharesAmount` will be inflated, as the funds being withdrawn are gone already (slashed, so transfered to an external recipient), and there will be no way to apply this correction against eigenLayerPendingWithdrawalSharesAmount. Granted that this is an edge case as mention earlier, but it can definately happen as `slashQueuedWithdrawal` is not something Puffer team can control, it's an external event that can always happen.

## Impact Details

`getELBackingEthAmount()` and `totalAssets()` will return an `inflated value` compare to the `real underlying value of the vault` once slash occurs in the middle of a withdrawal, which means translate into `protocol insolvency`. This gap will compounds overtime as those issues occurs.

## References

Here are the code snipet that helps understand the problem.

```diff
    function slashQueuedWithdrawal(address recipient, QueuedWithdrawal calldata queuedWithdrawal, IERC20[] calldata tokens, uint256[] calldata indicesToSkip)
        external
        onlyOwner
        onlyFrozen(queuedWithdrawal.delegatedAddress)
        nonReentrant
    {
        require(tokens.length == queuedWithdrawal.strategies.length, "StrategyManager.slashQueuedWithdrawal: input length mismatch");

        // find the withdrawalRoot
        bytes32 withdrawalRoot = calculateWithdrawalRoot(queuedWithdrawal);

        // verify that the queued withdrawal is pending
        require(
            withdrawalRootPending[withdrawalRoot],
            "StrategyManager.slashQueuedWithdrawal: withdrawal is not pending"
        );

        // reset the storage slot in mapping of queued withdrawals
+       withdrawalRootPending[withdrawalRoot] = false; // <------------ THIS will make claimWithdrawalFromEigenLayer always revert!

        // keeps track of the index in the `indicesToSkip` array
        uint256 indicesToSkipIndex = 0;

        uint256 strategiesLength = queuedWithdrawal.strategies.length;
        for (uint256 i = 0; i < strategiesLength;) {
            // check if the index i matches one of the indices specified in the `indicesToSkip` array
            if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
                unchecked {
                    ++indicesToSkipIndex;
                }
            } else {
                if (queuedWithdrawal.strategies[i] == beaconChainETHStrategy){
                     //withdraw the beaconChainETH to the recipient
                    _withdrawBeaconChainETH(queuedWithdrawal.depositor, recipient, queuedWithdrawal.shares[i]);
                } else {
                    // tell the strategy to send the appropriate amount of funds to the recipient
+                   queuedWithdrawal.strategies[i].withdraw(recipient, tokens[i], queuedWithdrawal.shares[i]); // <------------ THIS will transfer the funds to the an external recipient which is not PufferVault
                }
            }
            unchecked {
                    ++i;
                }
        }
    }
```

```diff
    function claimWithdrawalFromEigenLayer(
        IEigenLayer.QueuedWithdrawal calldata queuedWithdrawal,
        IERC20[] calldata tokens,
        uint256 middlewareTimesIndex
    ) external virtual {
        VaultStorage storage $ = _getPufferVaultStorage();

        bytes32 withdrawalRoot = _EIGEN_STRATEGY_MANAGER.calculateWithdrawalRoot(queuedWithdrawal);
        bool isValidWithdrawal = $.eigenLayerWithdrawals.remove(withdrawalRoot);
        if (!isValidWithdrawal) {
            revert InvalidWithdrawal();
        }

        $.eigenLayerPendingWithdrawalSharesAmount -= queuedWithdrawal.shares[0];

+       _EIGEN_STRATEGY_MANAGER.completeQueuedWithdrawal({  // <------------ THIS call will always revert, see the below.
            queuedWithdrawal: queuedWithdrawal,
            tokens: tokens,
            middlewareTimesIndex: middlewareTimesIndex,
            receiveAsTokens: true
        });
    }
```

```diff
    function _completeQueuedWithdrawal(QueuedWithdrawal calldata queuedWithdrawal, IERC20[] calldata tokens, uint256 middlewareTimesIndex, bool receiveAsTokens) onlyNotFrozen(queuedWithdrawal.delegatedAddress) internal {
        // find the withdrawalRoot
        bytes32 withdrawalRoot = calculateWithdrawalRoot(queuedWithdrawal);

        // verify that the queued withdrawal is pending
        require(
+           withdrawalRootPending[withdrawalRoot], // <------------ THIS will always revert as slashQueuedWithdrawal will have reset this flag
            "StrategyManager.completeQueuedWithdrawal: withdrawal is not pending"
        );

        require(
            slasher.canWithdraw(queuedWithdrawal.delegatedAddress, queuedWithdrawal.withdrawalStartBlock, middlewareTimesIndex),
            "StrategyManager.completeQueuedWithdrawal: shares pending withdrawal are still slashable"
        );

        ...
	}
```

## Recommendation

There is no magic mitigation for this issue. Since at that point `isValidWithdrawal` passed, we know the queuedWithdrawal is valid and initiateStETHWithdrawalFromEigenLayer was called, so we try our best effort to get the funds, so if we put this into a try/catch at least the PufferVault accounting will remain in good state. It's not perfect either as `completeQueuedWithdrawal` can revert for multiple reasons, some might be only temporary, which would work in the near future if we would retry (but not the current edge case which would be a permanent revert), so the mitigation I'm proposing also have downsides. The problem is also that `slashQueuedWithdrawal` is not even emitting a log. Otherwise, you could leave claimWithdrawalFromEigenLayer as is, but add `another restricted function` that could correct `eigenLayerPendingWithdrawalSharesAmount` manually in case the edge case reported here is detected (manually I guess), that would not be perfect either as there will be a window where the vault accounting will be broken. So there is no perfect mitigation to this issue.

```diff
    function claimWithdrawalFromEigenLayer(
        IEigenLayer.QueuedWithdrawal calldata queuedWithdrawal,
        IERC20[] calldata tokens,
        uint256 middlewareTimesIndex
    ) external virtual {
        VaultStorage storage $ = _getPufferVaultStorage();

        bytes32 withdrawalRoot = _EIGEN_STRATEGY_MANAGER.calculateWithdrawalRoot(queuedWithdrawal);
        bool isValidWithdrawal = $.eigenLayerWithdrawals.remove(withdrawalRoot);
        if (!isValidWithdrawal) {
            revert InvalidWithdrawal();
        }

        $.eigenLayerPendingWithdrawalSharesAmount -= queuedWithdrawal.shares[0];

+       try _EIGEN_STRATEGY_MANAGER.completeQueuedWithdrawal({
            queuedWithdrawal: queuedWithdrawal,
            tokens: tokens,
            middlewareTimesIndex: middlewareTimesIndex,
            receiveAsTokens: true
+       }) returns (string memory) {
+		} catch {
+		}
    }
```

## Proof of Concept

Unfortunatelly, I cannot have a coded PoC for this report as the `slashQueuedWithdrawal` is owned by EigenLayer (`onlyOwner`), so I cannot really simulate this with the current integration test suite, but we can `reason about it in a clear way` with the code indicated previously, so I don't think it is really required.

So if a normal withdrawal is attempted from EigenLayer but inadvertently such queuedWithdrawal is being slashed successfully before the call to claimWithdrawalFromEigenLayer, we will have this problematic edge case.

1. initiateStETHWithdrawalFromEigenLayer is called.
2. \_EIGEN\_STRATEGY\_MANAGER.slashQueuedWithdrawal is called againts the queuedWithdrawal genered previously.
3. claimWithdrawalFromEigenLayer is called, but `always revert`.
4. call to `getELBackingEthAmount()` and `totalAssets()` will result in an `inflated value`, which is the problem.
