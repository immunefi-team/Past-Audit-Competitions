# 28665 - \[SC - Low] Underflow risk in receive function due to discr...

Submitted on Feb 23rd 2024 at 10:40:26 UTC by @grobelr for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28665

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Temporary freezing of funds for at least 1 hour
* Permanent freezing of funds

## Description

## Brief/Intro

The identified vulnerability arises when initiating withdrawals from Lido's stETH and the actual ETH received during the claim process.

Due to stETH's rebasing nature, which can lead to an increase or decrease in its value relative to ETH, and potential rounding errors, there's a mismatch risk between the anticipated and actual received ETH amounts. This discrepancy, particularly following a negative rebase, could lead to an underflow in the contract's `receive()` function when it attempts to adjust `$.lidoLockedETH` based on the received `msg.value`.

## Vulnerability Details

The core of the vulnerability lies in the contract's handling of ETH withdrawals claimed from Lido's stETH positions. When a user initiates a withdrawal, the contract locks an equivalent amount of ETH in `$.lidoLockedETH`, based on the current stETH to ETH conversion rate. However, since stETH rebases periodically to reflect staking rewards or penalties, the actual ETH value of stETH can change by the time the withdrawal is claimed. Lido's documentation on withdrawals outlines that the amount of ETH claimable is fixed upon the withdrawal request's finalization. Importantly, the stETH/ETH rate at the request's finalization cannot exceed the rate at the time of request creation. Users can claim:

* Normally – the ETH amount equivalent to the stETH amount at the request's placement time.
* Discounted – a reduced ETH amount, corresponding to the oracle-reported share rate, if the protocol experiences significant losses (such as slashings and penalties).

Reference: [Lido's Guide on Tokens Integration](https://docs.lido.fi/guides/lido-tokens-integration-guide/#withdrawals-unsteth) Although such a scenario is highly unlikely, the occurrence of even a single event could result in the Vault being unable to claim the expected rewards due to an underflow, thereby freezing the funds.

## Impact Details

```solidity
function receive() external payable {
    if ($.isLidoWithdrawal) {
        $.lidoLockedETH -= msg.value;
    }
}
```

This code snippet assumes a direct correlation between the locked amount in $.lidoLockedETH and the ETH received (msg.value). However, if you take a look on Lido [WithdrawalQueueBase.sol](https://etherscan.io/address/0xe42c659dc09109566720ea8b2de186c2be7d94d9#code#F23#L472) L472, the function \_claim calls \_calculateClaimableEther which can create rounding errors. Besides that, an underflow on receive() function can completely freeze claiming eth from Lido, if one negative rebase happens between [initiateETHWithdrawalsFromLido](https://etherscan.io/address/0x39ca0a6438b6050ea2ac909ba65920c7451305c1#code#F1#L159) and [claimWithdrawalsFromLido](https://etherscan.io/address/0x39ca0a6438b6050ea2ac909ba65920c7451305c1#code#F1#L106)

This function presupposes a one-to-one correspondence between the amount initially "locked" in $.lidoLockedETH and the ETH amount received (msg.value). However, a closer inspection of Lido's [WithdrawalQueueBase.sol](https://etherscan.io/address/0xe42c659dc09109566720ea8b2de186c2be7d94d9#code#F23#L472) at line 472 reveals that the \_claim function, through \_calculateClaimableEther, may introduce rounding discrepancies. Moreover, an underflow in the receive() function could severely disrupt the ability to claim ETH from Lido, particularly if a negative rebase occurs between the stages of [initiateETHWithdrawalsFromLido](https://etherscan.io/address/0x39ca0a6438b6050ea2ac909ba65920c7451305c1#code#F1#L159) and [claimWithdrawalsFromLido](https://etherscan.io/address/0x39ca0a6438b6050ea2ac909ba65920c7451305c1#code#F1#L106)

## Proof of Concept

Add this function to [PufferTest.integration.t.sol](https://github.com/PufferFinance/pufETH/blob/main/test/Integration/PufferTest.integration.t.sol)

```solidity
    function test_lido_rebase_withdrawal()
    public
    giveToken(BLAST_DEPOSIT, address(stETH), address(pufferVault), 1000 ether) // Blast got a lot of stETH
{
    // Initiate Withdraw
    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 0.1 ether; // steth Amount
    vm.startPrank(OPERATIONS_MULTISIG);
    uint256[] memory requestIds = pufferVault.initiateETHWithdrawalsFromLido(amounts);

    _finalizeWithdrawals(requestIds[0]);
    // Claim
    pufferVault.claimWithdrawalsFromLido(requestIds);

    //Verify
    assertEq(pufferVault.getPendingLidoETHAmount(), 0);
}
```

```
forge test --mt test_lido_rebase_withdrawal -vv
  Error: a == b not satisfied [uint]
        Left: 228234961450
       Right: 0
```

Given the complexity of simulating a negative rebase without altering the entire storage, this snippet serves merely as an illustration of how the claimWithdrawal function from the WithdrawQueue could produce different outcomes.
