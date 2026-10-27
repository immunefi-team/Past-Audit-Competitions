# 26275 - \[SC - Insight] Bad implementation of executeTransaction functi...

Submitted on Nov 30th 2023 at 02:13:44 UTC by @v0id for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26275

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Block stuffing for profit

## Description

DeGate designs its proxy upgrade implementation by the following rule : a multisig proposal will first get Initiated to to store the upgrade implementation data in a timelock contract . the timelock contract meets a special requirement where it can only upgrade contract and change the proxy address when it has been 45 days between the day that the request has been initiiated and it will get executed . another multisig proposal would then get involved to call timelock whenever it was possible (after 45 days) to let the upgrade process happen. while taking a look at multisig smart contract we can understand that : for a proposal to get executed successfully it has to go through some situations . an owner has to first call submitTransaction function which would get the destination address and the data that it has to attach to the transaction from the caller , and then it would call confirmTransaction function internally , at this function the owner/caller would get added to the array of owners that have signed the transaction to get executed . and in the end the executeTransaction function would get called internally . the executeTransaction function is where the transaction would get executed , but to execute that transaction and the tx.executed to get successful set as true , two conditions have to happen , first number of owners who have signed the transaction has to meet the prescribed amount , and second the transaction call to the destination address has to get successfully executed or else the tx.executed would get a false value .

the issue here is that the contract would either let another person execute their signed transaction or it fails to implement an Appropriate Environment where the someone would not be able to execute their transaction .

let say the multisig requires 4 signers to sign a proposal .those 4 may decide to call the confirmTransaction function before the timelock's settled time , what will happen here is that at 4th call after the first condition would become pass and tx.destination would get called which will get reverted because it has noy yet meet the mentioned time . and because contract uses low level call method .call , the whole execution would not get reverted and so in the next line : tx.executed = false and after that anyone can call executeTransaction . or in Low probability of occurrence situations 4th call would get executed after timelock's settled time but this time the call it self would get reverted for some reasons ... .

now lets say what is the impact of letting anyone call a function that will change address of our proxy's implementation . in defi proxies play an important role , they will have one storage but multiple contract . that means that the storage will always be the same but its only the logic and source code of the proxy that will get changed by changing the address of where it will delegate call its calls to . having the above condition its possible for a maliciuse user to call a function with logic of smart contract A and store/change/delete the storage C value and then call the same function or another one with logic of smart contract B and store/change/delete value storage C value , with same block.timestamp .

for example depositing and withdrawing funds in current implementation looks like :

```
deposit> wexchange proxy > exchange implementation > depositcontract proxy > deposit contract implementation  
^^^^^^^^^^^^
withdraw > exchange proxy > exchange implementation > depositcontract proxy > deposit contract implementation  
```

now if someone change logic of depositcontract implementation at ^^^^^^^^^^^^ they will get a different result , that idea may not cause a huge loss of funds by it self but when we can do all of them in one transaction we can probably use flashloans . we also have block.timestamp that is set to same value before and after the upgrade . this is important because in many smart contracts its not possible to take a profit by eg depositing your finds in and withdrawing them out in one block .

the above scenario is just one example to help to understand the issue. and because of that there is no vision of future implementation 's source code or the current one in this contest we cant 100% say how this can happen or even this can happen or not . we have to also mention that the mentioned issue is being fixed in original gnosis multisig wallets by adding a Stealth Safe Guard (<https://mirror.xyz/yearn-finance-engineering.eth/9uInM\\_sCrogPBs5qkFSNF6qe-32-0XLN5bty5wKLVqU>)

to fix the issue , first of all executetransaction function can contain a requirement where it will only accept calls from owners/admins/executors and/or revert a confirmation transaction if it got reverted at executetransaction call .

because of some Low probability of occurrence conditions that this vulnerability requires we somehow agree if project wants to change the severity .

## Proof of concept

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.4;

contract MultiSigWalletWithDailyLimit {
    function callExecuteTransaction(uint transactionId) public {
        this.executeTransaction(transactionId);
    }
}

contract MyInteractingContract {
    MultiSigWalletWithDailyLimit public multiSigWallet;

    constructor() public {
        multiSigWallet = 0x2028834B2c0A36A918c10937EeA71BE4f932da52;
    }

    function executeTransactionFromOtherContract(uint transactionId) public {
        multiSigWallet.callExecuteTransaction(transactionId);
    }
}
//for the above contract transaction execution to be successful its required that all 4 mutlsig owners sign the transaction, but the execution call gets reverted . 
//when owners sign the transaction , the multisig contract will try to execute it as well as confirming it by the owner . the execute may fail due to unexpeted conditions and/or timelock's //cooldown delay , but the confirmation would still happen and that is where the issue lies 
```
