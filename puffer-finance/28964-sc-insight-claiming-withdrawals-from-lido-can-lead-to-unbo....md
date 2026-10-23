# 28964 - \[SC - Insight] Claiming withdrawals from Lido can lead to unbo...

Submitted on Mar 3rd 2024 at 14:20:08 UTC by @LokiThe5th for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28964

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Unbounded gas consumption

## Description

## Brief/Intro

`PufferVault::claimWithdrawalsFromLido()` is susceptible to unbounded gas use, as it uses an internal call to `LidoWithdrawalQueue::claimWithdrawal()`, which is itself known to be susceptible to unbounded gas usage. For a single claim, this would not be an issue, but in this case `PufferVault::claimWithdrawalsFromLido()` takes an array of `requestIds` and makes multiple single `claimWithdrawal` calls in a `for loop`, increasing the unbounded gas use risk.

*Please note: even though this is correctly categorized as `Unbounded Gas Use`, the reviewer does not consider this to be a medium severity finding, and LOW would be more appropriate. The reason being that it can (and will) cost the team gas, but the truly unbounded gas scenario will require a low probability event (an extreme growth in the number of checkpoints on Lido's withdrawal queue) as well as the puffer team queueing extremely large arrays*.

## Vulnerability Details

`PufferVault::claimWithdrawalsFromLido()` uses multiple `LidoWithdrawalQueue::claimWithdrawal()` calls in a `for-loop` to finalize claims from Lido. But the underlying `claimWithdrawal` function from Lido warns us that there is a risk of unbounded gas use.

From the `LidoWithdrawalQueue::claimWithdrawal()` function we see:

```
    /// @notice Claim one`_requestId` request once finalized sending locked ether to the owner
    /// @param _requestId request id to claim
    /// @dev use unbounded loop to find a hint, which can lead to OOG
    /// @dev
    ///  Reverts if requestId or hint are not valid
    ///  Reverts if request is not finalized or already claimed
    ///  Reverts if msg sender is not an owner of request
    function claimWithdrawal(uint256 _requestId) external {
        _claim(_requestId, _findCheckpointHint(_requestId, 1, getLastCheckpointIndex()), msg.sender);
        _emitTransfer(msg.sender, address(0), _requestId);
    }
```

As per Lido's comments, we see that this function searches for the appropriate hint using `_findCheckpointHint`, which in turn uses a binary search, with the `min` set to `1` and the `lastCheckpointIndex` being the last checkpoint.

This means that for every item in the array passed to `claimWithdrawals` the entire checkpoint history is traversed with `LidoWithdrawalQueue::claimWithdrawal()`.

## Impact Details

Because the number of checkpoints can only grow higher as time passes, the gas cost will keep increasing slightly over time, even for the same amount of items in an array.

As the number of checkpoints will continue increasing, and the length of the `requestIds` passed into `claimWithdrawalsFromLido` is likely to increase, this becomes more probable to cause unbounded gas use (and thus out of gas errors) in the future.

## References

* Lido's `claimWithdraw`: <https://etherscan.io/address/0xe42c659dc09109566720ea8b2de186c2be7d94d9#code#F22#L277>
* Lido's `_findCheckpointHint`: <https://etherscan.io/address/0xe42c659dc09109566720ea8b2de186c2be7d94d9#code#F23#L385>

## Proof of Concept

Kindly inspect the `LidoWithdrawalQueueMockGasUse` file before running the PoC. As it is difficult to test this on a forked env, the aforementioned file mocks the functionality that is crucial to this PoC. It basically takes the code from Lido and places it in the mock, allowing us to approximate the issue (gas use).

The below files should be placed within the `puffEth` repo.

Place the `UnboundedGasPoC.sol` file in the `test` directory.\
Place the `MockAccessManager.sol`, `LidoWithdrawalQueueMockGasUse.sol` and the `stETHStrategyMock.sol`files in the `test/mocks` directory.

The files can be found in this drive folder: <https://drive.google.com/drive/folders/1dvxLMLvy40g44epde0P6H287pyOshRcR?usp=sharing>

Or can be retrieved from these gists:

* `UnboundedGasPoC.sol`: <https://gist.github.com/lokithe5th/8a1f79c98aa7c915f42d4bcc0f1ce5c1\\>
  stETHStrategyMock.sol: <https://gist.github.com/lokithe5th/90ffb119fd5c1ffb81dd35313b49091a\\>
  MockAccessManager.sol: <https://gist.github.com/lokithe5th/5c6f560b9587796d6d2894a9e1aabf57\\>
  LidoWithdrawalQueueMockGasUse: <https://gist.github.com/lokithe5th/c35f1fd8d147ab27d3a25f08cc18ac00>

Run the test with `forge test --match-test test_gas_Claim --gas-report -vvv`.

At line 61 of `UnboundedGasPoC.sol` you can toggle if we want to use a control scenario (where updated hints are used), or if you want to test an approximation of the current setup, where no hints are used.

**Please ensure you have commented out the `disableInitializers()` calls in the `PufferVault` constructor.**

The console output of the test scenario, note that the figures are approximations due to this not being a fork test:

```
| Function Name                            | min             | avg      | median   | max      | # calls |  

| claimWithdrawalsFromLido                 | 1538801         | 1538801  | 1538801  | 1538801  | 1       |  
```

The console output of the control scenario, using updated hints:

```
| Function Name                            | min             | avg      | median   | max      | # calls |  

| claimWithdrawalsFromLido                 | 1148501         | 1148501  | 1148501  | 1148501  | 1       |
```
