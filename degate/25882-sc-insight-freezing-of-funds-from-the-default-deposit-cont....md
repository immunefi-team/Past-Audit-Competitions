# 25882 - \[SC - Insight] Freezing of funds from the Default Deposit Cont...

Submitted on Nov 20th 2023 at 22:25:24 UTC by @infosec\_us\_team for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25882

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Permanent freezing of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.

## Description

## Background

DeGate's guarantee self custody of their assets with the Exodus Mode and escaping censorship via forced withdrawals.

The request must be served within a defined time period. If this does not happen, the system will halt regular operation and permit trustless withdrawal of funds.

## Bug summary

Our team discovered an attack vector that allows a malicious operator to censor a target and also freeze/seize his funds at a low cost. The victim is unable to request a forced withdraw, and the exchange will not get into Exodus Mode.

The attack costs up to 0.02 ETH ($40) every 30 days with the funds seized, per victim. It is a low cost, the victims could be whales with a substantial amount of assets deposited, the malicious operator would ask for a fee to release their funds.

With a cost lower than $500 a malicious node operator can seize for an entire year all the deposited funds of a specific user.

In the "Permanent Fix" section of this report we'll share how adding 2 lines of code to `forceWithdraw(...)`, prevent this and other types of attack vectors from happening, and additionally make the `forceWithdraw(...)` function 100% trustless and censorship resistant.

## Attack vector

In `ExchangeV3.sol` the `forceWithdraw(...)` function accepts an arbitrary `accountID` and will store the **forceWithdraw** request in:

```javascript
S.pendingForcedWithdrawals[accountID][tokenID]
```

The next time the function `forceWithdraw(...)` is called with the same `accountID` and `tokenID` will revert if the pending request has not been processed by the operator.

Here's how this can be abused:

**Step 1-** Operator censors a whale and stops processing his orders.

**Step 2-** Whale tries to exit the exchange by sending a `forceWithdraw(...)` request, but the operator front-runs his transaction and he sends the `forceWithdraw(...)` using the whale's `accountID` and `tokenID`. Whale's transaction will fail.

**Step 3-** The operator can now wait 14 days, and after that, include the pending forced withdraw in a block and submit it to L1 to prevent the exchange from entering a Exodus Mode.

**Step 4-** Because the address of the owner for that `accountID` (the victim's address) is different from the address that initiated the pending forced withdraw (operator's address) the pending forced withdraw will be invalidated and removed from the storage:

> Snippet of code extracted from *WithdrawTransaction.sol*, function `process(...)`, used in *ExchangeV3.sol* to process withdrawals.

```javascript
// withdrawType = 2: onchain valid forced withdrawals (owner and accountID match),
if (withdrawal.withdrawalType == 2) {
    require(withdrawal.from == forcedWithdrawal.owner, "INCONSISENT_OWNER");
} else {
    //withdrawalType == 3: onchain invalid forced withdrawals (owner and accountID mismatch)
    require(withdrawal.from != forcedWithdrawal.owner, "INCONSISENT_OWNER");
    require(withdrawal.amount == 0, "UNAUTHORIZED_WITHDRAWAL");
}
// delete the withdrawal request and free a slot
delete S.pendingForcedWithdrawals[withdrawal.fromAccountID][withdrawal.tokenID];
S.numPendingForcedTransactions--;
```

**Step 5-** If now, the victim decides to submit another `forceWithdraw(...)` request, the operator can front-run him again to freeze his funds for another 14 days, and process the pending request just before it expires.

If by any chance, all other users in the protocol decide to submit force exit requests, the operator can process them as normal and let them go, but leave the whale's funds there, seized forever.

To 100% guarantee self-custody of funds, escape censorship and forbid a malicious node or a griefer from submitting invalid **forceWithdraw** requests with our `accountID` to prevent us from withdrawing our funds, read the next section.

## Permanent fix

DeGate's documentation explains that the reason any `accountID` is accepted, and the validity of the request is checked offline instead of onchain, is because the `ExchangeV3.sol` smart contract doesn't know who the `owner` of a specific `accountID` is.

There's an elegant and simple solution to know if `msg.sender` is the owner of the `accountID` or the Agent of the owner, leading to always generating valid `forceWithdraw` pending requests, and permanently stopping this function from being front-run.

In the extraordinary event where a user its been censored and he wants to send a request to exit the exchange in a trustless, unstoppable manner, he can submit a *forcedWithdrawal* request with MerkleProof *(similar to what users already have to do in the `withdrawFromMerkleTree(...)` function)*, and `ExchangeV3.sol` will calculate the MerkleRoot using `msg.sender` as the value for the `merkleProof.accountLeaf.owner` variable.

Now, the only way the `forcedWithdrawal(...)` transaction can succeed, is if `msg.sender` is equal to the `owner` of the `accountID` or the Agent of the account.

There's no way to maliciously bypass this check.

This is an example in pseudo code for the sake of simplicity (we share the full code for the production-ready solution later on):

```solidity
// Given a merkleProof "ExchangeData.MerkleProof calldata merkleProof;"
// Replace the "owner" field in the merkle proof with the address of the msg.sender
merkleProof.accountLeaf.owner = address(msg.sender);

// If the rest of the parameters for the merkleProof (like "accountID", etc)
// Are not valid, the calculated root will not be equal to the merkleRoot stored
// in the smart contract.

// This forces users calling the `forcedWithdrawal(...)` function to use the correct `accountID`

// ... to calculate the merkleRoot use the exact same logic that is used in `isAccountBalanceCorrect(...)`
uint calculatedRoot = getBalancesRoot(
    merkleProof.balanceLeaf.tokenID,
    merkleProof.balanceLeaf.balance,
    merkleProof.balanceMerkleProof
);
calculatedRoot = getAccountInternalsRoot(
    merkleProof.accountLeaf.accountID,
    merkleProof.accountLeaf.owner,
    merkleProof.accountLeaf.pubKeyX,
    merkleProof.accountLeaf.pubKeyY,
    merkleProof.accountLeaf.nonce,
    calculatedRoot,
    merkleProof.accountMerkleProof
);
// Check against the expected Merkle root (the one that is already stored inside ExchangeV3.sol)
return (calculatedRoot == merkleRoot);
```

This can only succeed if the caller (`msg.sender`) passed the correct `accountID`.

Now - examples aside - here's the final implementation for a `forceWithdraw(...)` function that is 100% trustless and censorship resistant:

> In `ExchangeV3.sol`:

```solidity
// Updated the name from "forceWithdraw(...)" to "forceWithdrawFromMerkleTree(...)"
function forceWithdrawFromMerkleTree(

    // Take an arbitrary `from` address that will be checked in the
    // onlyFromUserOrAgent(from) modifier below, to make sure it
    // is equal to msg.sender or msg.sender is an agent
    address from,

    // Token to withdraw
    address token,

    // MerkleProof to verify ownership of the accountID
    // The accountID variable itself is part of the "merkleProof" param
    ExchangeData.MerkleProof calldata merkleProof,
    )
    external
    override
    nonReentrant
    payable
    onlyFromUserOrAgent(from)
{
    state.forceWithdrawFromMerkleTree(from, token, merkleProof);
}
```

> In the `ExchangeWithdrawals.sol` library:

```solidity
function forceWithdrawFromMerkleTree(
    ExchangeData.State storage S,
    address owner,
    address token,
    ExchangeData.MerkleProof calldata merkleProof,
    )
    public
{

    // We will only add 2 lines of code. These verify the caller and the accountID are valid,
    // and the rest of the code will be exactly the same as in `forceWithdraw(...)`
    // but replacing every instance of "accountID" with "merkleProof.accountLeaf.accountID"

    // STEP 1- Replace the owner of the received merkle proof with the caller
    merkleProof.accountLeaf.owner = owner;
    // STEP 2- Verify that the provided Merkle tree data is valid by calculating the merkleRoot
    ExchangeBalances.verifyAccountBalance(
        uint(S.merkleAssetRoot),
        merkleProof
    );

    // if the `owner` param is not the owner of the `accountID` inside "merkleProof.accountLeaf.accountID"
    // the transaction will revert
    // --------------------------- The logic below this is the same as in `forceWithdraw(...)`

    require(!S.isInWithdrawalMode(), "INVALID_MODE");
    require(S.getNumAvailableForcedSlots() > 0, "TOO_MANY_REQUESTS_OPEN");
    require(merkleProof.accountLeaf.accountID < ExchangeData.MAX_NUM_ACCOUNTS, "INVALID_ACCOUNTID");
    require(merkleProof.accountLeaf.accountID != 0, "INVALID_ACCOUNTID");

    uint32 tokenID = S.getTokenID(token);

    uint withdrawalFeeETH = S.loopring.forcedWithdrawalFee();

    require(msg.value >= withdrawalFeeETH, "INSUFFICIENT_WITHDRAW_FEE");

    uint feeSurplus = msg.value.sub(withdrawalFeeETH);
    if (feeSurplus > 0) {
        msg.sender.sendETHAndVerify(feeSurplus, gasleft());
    }

    require(
        S.pendingForcedWithdrawals[merkleProof.accountLeaf.accountID][tokenID].timestamp == 0,
        "WITHDRAWAL_ALREADY_PENDING"
    );

    S.pendingForcedWithdrawals[merkleProof.accountLeaf.accountID][tokenID] = ExchangeData.ForcedWithdrawal({
        owner: owner,
        timestamp: uint64(block.timestamp)
    });

    S.numPendingForcedTransactions++;

    emit ForcedWithdrawalRequested(
        owner,
        token,
        merkleProof.accountLeaf.accountID
    );
}
```

## Proof of concept

A proof of concept demonstrating how a forced withdrawal requested from an incorrect owner will be ignored, is in the `testExchangeDepositWithdraw.ts` test file. Here's the test:

```javascript
it("Forced withdrawal (incorrect owner)", async () => {
  await createExchange();

  const ownerA = exchangeTestUtil.testContext.orderOwners[0];
  const ownerB = exchangeTestUtil.testContext.orderOwners[1];
  const balance = new BN(web3.utils.toWei("7", "ether"));
  const token = exchangeTestUtil.getTokenAddress("LRC");

  const deposit = await exchangeTestUtil.deposit(ownerA, ownerA, token, balance);
  await exchangeTestUtil.submitTransactions();
  await exchangeTestUtil.submitPendingBlocks();

  // Do the request
  await exchangeTestUtil.requestWithdrawal(ownerA, token, balance, "ETH", new BN(0), {
    authMethod: AuthMethod.FORCE,
    signer: ownerB
  });

  // Commit the withdrawal
  await exchangeTestUtil.submitTransactions();

  // Submit the block
  const expectedResult = { ...deposit };
  expectedResult.amount = new BN(0);
  await submitWithdrawalBlockChecked([expectedResult]);
});
```
