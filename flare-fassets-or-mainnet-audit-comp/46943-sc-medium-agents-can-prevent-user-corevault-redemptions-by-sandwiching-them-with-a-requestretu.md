# #46943 \[SC-Medium] Agents can prevent user CoreVault redemptions by sandwiching them with a requestReturnFromCoreVault and a cancelReturnFromCoreVault

**Submitted on Jun 6th 2025 at 14:41:02 UTC by @niroh for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46943
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol>
* **Impacts:**
  * Temporary freezing of funds

## Description

## Brief/Intro

Agents can front run a user redemption request from the core vault with a requestReturnFromCoreVault for an amount that does not leave enough available balance in the vault. They can then cancel the request immediately after the user redemption, causing the redemption to fail for almost no cost for them.

## Vulnerability Details

Agents can can requestReturnFromCoreVault to receive some of the vault's underlying balance as part if their Minted. The request enters the cancelableTransferRequests queue and increases the cancelableTransferRequestsAmount value. The request if fulfilled the next time triggerInstructions is called (once per day) and enough funds are available. While the request is pending, agent collateral is reserved for the requested amount and CV funds available for transfer are decreased by the same amount. However, the agent can cancel the request at any time by calling cancelTransferRequestFromCoreVault. This releases the agent collateral and makes the requested amount available again in the vault.

Note that creating and canceling a returnFromCV request is costless and is not timelocked or limited in any way. This enables agents to deliberately lock CV funds to prevent redeemers from redeeming from the CV.

### attack scenario

1. Starting state: agent A with 100 minted lots and 500 free collateral lots. CV has 200 lots available and no pending requests nor escrows.
2. User wants to redeem 100 lots and decides to do so from the CV pool. They send a redemption request to the pool.
3. Agent wants to prevent the user redemption. They frontrun the user transaction with a requestReturnFromCoreVault tx, requesting 110 lots.
4. The Agent's request locks 110 lots from the CV's 200 balance, preventing the redemption from succeeding. This can be seen in this code:

```solidity

//from requestTransferFromCoreVault line 186
if (_cancelable) {
        // only one cancelable request per destination address
        for (uint256 i = 0; i < cancelableTransferRequests.length; i++) {
            TransferRequest storage req = transferRequestById[cancelableTransferRequests[i]];
            require(keccak256(bytes(req.destinationAddress)) != destinationAddressHash, "request already exists");
        }
        cancelableTransferRequestsAmount += _amount;
        cancelableTransferRequests.push(nextTransferRequestId);
        newTransferRequest = true;
    } else {
       .
       .
       .
//line 215 - the total requests amount (both agent returns and redeems) together with the current request
//are checked to not exceed total available funds
uint256 requestsAmount = totalRequestAmountWithFee();
require(requestsAmount <= availableFunds + escrowedFunds, "insufficient funds");
```

5. As is evident from the code above, when the redeem tx is executed, it fails in line 216 because total requests (together with the agent's return and the redemption) exceed total funds
6. The agent then backruns the redemption with a cancelTransferRequestFromCoreVault which releases their reserved collateral and deletes the request:

```solidity
function _deleteReturnFromCoreVaultRequest(Agent.State storage _agent) private {
    assert(_agent.activeReturnFromCoreVaultId != 0 && _agent.returnFromCoreVaultReservedAMG != 0);
    _agent.reservedAMG -= _agent.returnFromCoreVaultReservedAMG;
    _agent.activeReturnFromCoreVaultId = 0;
    _agent.returnFromCoreVaultReservedAMG = 0;
}
```

7. Note that there are no fees imposed on the agent in this process, and their locked collateral is released within the same block, therefore there is virtually no cost for the attack other then transaction fees which are minimal on Flare/Songbird.
8. This procedure can be repeated as many times as needed as there are no limitations on the number of times an agent can request and cancel returns.

## Impact Details

1. For users whose redemptions can not covered by agents Minted, blocking CV redemptions effectively prevents them from redeeming, at least temporarily enough Minted exists with agents to serve their redemption.
2. Agents can use the same technique to: A. block or disrupt other agent returns from the CV. B. Censor specific user mints (by temporarily increasing their reserved amount to prevent the mint)

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CoreVaultManager.sol#L232> <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CoreVaultManager.sol#L215C8-L216C89>

## Proof of Concept

## Proof of Concept

### attack scenario

1. Starting state: agent A with 100 minted lots and 500 free collateral lots. CV has 200 lots available and no pending requests nor escrows.
2. User wants to redeem 100 lots and decides to do so from the CV pool. They send a redemption request to the pool.
3. Agent wants to prevent the user redemption. They frontrun the user transaction with a requestReturnFromCoreVault tx, requesting 110 lots.
4. The Agent's request locks 110 lots from the CV's 200 balance, preventing the redemption from succeeding. This can be seen in this code:

```solidity

//from requestTransferFromCoreVault line 186
if (_cancelable) {
        // only one cancelable request per destination address
        for (uint256 i = 0; i < cancelableTransferRequests.length; i++) {
            TransferRequest storage req = transferRequestById[cancelableTransferRequests[i]];
            require(keccak256(bytes(req.destinationAddress)) != destinationAddressHash, "request already exists");
        }
        cancelableTransferRequestsAmount += _amount;
        cancelableTransferRequests.push(nextTransferRequestId);
        newTransferRequest = true;
    } else {
       .
       .
       .
//line 215 - the total requests amount (both agent returns and redeems) together with the current request
//are checked to not exceed total available funds
uint256 requestsAmount = totalRequestAmountWithFee();
require(requestsAmount <= availableFunds + escrowedFunds, "insufficient funds");
```

5. As is evident from the code above, when the redeem tx is executed, it fails in line 216 because total requests (together with the agent's return and the redemption) exceed total funds
6. The agent then backruns the redemption with a cancelTransferRequestFromCoreVault which releases their reserved collateral and deletes the request:

```solidity
function _deleteReturnFromCoreVaultRequest(Agent.State storage _agent) private {
    assert(_agent.activeReturnFromCoreVaultId != 0 && _agent.returnFromCoreVaultReservedAMG != 0);
    _agent.reservedAMG -= _agent.returnFromCoreVaultReservedAMG;
    _agent.activeReturnFromCoreVaultId = 0;
    _agent.returnFromCoreVaultReservedAMG = 0;
}
```

7. Note that there are no fees imposed on the agent in this process, and their locked collateral is released within the same block, therefore there is virtually no cost for the attack other then transaction fees which are minimal on Flare/Songbird.
8. This procedure can be repeated as many times as needed as there are no limitations on the number of times an agent can request and cancel returns.
