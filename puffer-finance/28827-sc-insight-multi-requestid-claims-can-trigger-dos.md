# 28827 - \[SC - Insight] Multi requestid claims can trigger DOS

Submitted on Feb 28th 2024 at 03:26:42 UTC by @SentientX for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28827

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Claiming multiple withdrawals from Puffer Vault using multiple requestid's in an array can lead to denial of service triggered by bottleneck race conditions in corresponding function in Lido handling claim request.

## Vulnerability Details

When multisig action initiatesWithdrawal from Lido by calling `function initiateETHWithdrawalsFromLido(uint256[] calldata amounts) external`, users can claim their withdrawals based on requestid's by calling `function claimWithdrawalsFromLido(uint256[] calldata requestIds) external`

This function in turn by using this line `_LIDO_WITHDRAWAL_QUEUE.claimWithdrawal(requestIds[i]);` passing the requestId/s as an array to the Lido WithdrawalQueue.sol address `0x889edC2eDab5f40e902b864aD4d7AdE8E412F9B1`

```
    function claimWithdrawal(uint256 _requestId) external {
        _claim(_requestId, _findCheckpointHint(_requestId, 1, getLastCheckpointIndex()), msg.sender);
        _emitTransfer(msg.sender, address(0), _requestId);
    }
```

This may create bottle necks that can get users funds stuck or create a Denial of Service when a user pass a large number of requestid's at once. Example:

1. Alice mints seperate amounts of puffEth example at 10 different times.
2. Multisig calls initiateWithdraw from Lido
3. Alice attempts to claim all 10 withdrawals and this creates a denial of service. Please see POC.

Consider the following solution:

Lido, has options of bulk claims using `function claimWithdrawalsTo(uint256[] calldata _requestIds, uint256[] calldata _hints, address _recipient)` and their can be a seperate function `function claimWithdrawalFromLido(requestId) external` To handle individual withdrawals from Lido.

This should prevent against the above scenario at improve user experience.

## Impact Details

Denial of Service can be caused if users attempt to claim multiple request at a time.

## References

Result after running POC:

```
    ├─ [31770] 0xD9A442856C234a39a81a089C06451EBAa4306a72::claimWithdrawalsFromLido([27173 [2.717e4], 27174 [2.717e4], 27176 [2.717e4], 27177 [2.717e4], 27178 [2.717e4], 27179 [2.717e4], 27180 [2.718e4], 27181 [2.718e4], 27182 [2.718e4], 27183 [2.718e4]])
    │   ├─ [31316] 0x39Ca0a6438B6050ea2aC909Ba65920c7451305C1::claimWithdrawalsFromLido([27173 [2.717e4], 27174 [2.717e4], 27176 [2.717e4], 27177 [2.717e4], 27178 [2.717e4], 27179 [2.717e4], 27180 [2.718e4], 27181 [2.718e4], 27182 [2.718e4], 27183 [2.718e4]]) [delegatecall]
    │   │   ├─ [6096] 0x889edC2eDab5f40e902b864aD4d7AdE8E412F9B1::claimWithdrawal(27173 [2.717e4])
    │   │   │   ├─ [5512] 0xE42C659Dc09109566720EA8b2De186c2Be7D94D9::claimWithdrawal(27173 [2.717e4]) [delegatecall]
    │   │   │   │   └─ ←
    │   │   │   └─ ←
    │   │   └─ ←
	
```

## Proof of Concept

1. mkdir pufferPOC
2. cd pufferPOC
3. delete test folder and Counter.sol and Counter.t.sol
4. place poc and interfaces in src folder

run with `forge test --contracts ./src/pufferPOC-3.sol -vvvv --evm-version shanghai`
