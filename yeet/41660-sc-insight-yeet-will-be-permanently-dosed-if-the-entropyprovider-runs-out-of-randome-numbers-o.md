# #41660 \[SC-Insight] Yeet will be permanently DOSED if the entropyProvider runs out of randome numbers or gets blacklisted

**Submitted on Mar 17th 2025 at 11:36:14 UTC by @BenR for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41660
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

To determine the 10 winners of the yeetback, the Yeetback contract calls the Entropy service to get a randome number. For this call to the Entropy service the address of an `entropyProvider` is given by the Yeetback contract. If the provided `entropyProvider` is blacklisted or stops providing random numbers the whole Yeet protocol will be DOSED because both the `entropyProvider` state variable in the `Yeetback` contract and the `yeetback` state variable in the `Yeetcontract` cannot be changed.

## Vulnerability Details

After a yet round is over, a new round is restarted by calling `Yeet::restart()`. This function calls the `addYeetback()` function of the contract stored in the state variable `yeetback`.\
This in turn calls `entropy.requestWithCallback` to acquire a random number which is used to determine the winners of the `yeetbacks`.\
The function call to `entropy.requestWithCallback()` take the address of an entropy provider as an argument which is stored in the state variable `entropyProvider`. Once this variable is set in the constructor it cannot be changed.

The issue arises from the fact that the call to `entropy.requestWithCallback()` can revert if the provided `entropyProvider` does not exist or does not have any randomness left. This can happen if the entropyProvider is blacklisted (<https://docs.pyth.network/entropy/protocol-design>) or all provided random numbers are already used up since the provider only provides a limited amount of random numbers. (requestWithCallback() => requestHelper() <https://github.com/pyth-network/pyth-crosschain/blob/c905fbe1a8bc1d78bcae483e9bca6009db6d9be1/target\\_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L202C4-L251C6>)

```solidity
function requestHelper(
        address provider,
        bytes32 userCommitment,
        bool useBlockhash,
        bool isRequestWithCallback
    ) internal returns (EntropyStructs.Request storage req) {
…
       if (_state.providers[provider].sequenceNumber == 0)
@>             revert EntropyErrors.NoSuchProvider();

        // Assign a sequence number to the request
        uint64 assignedSequenceNumber = providerInfo.sequenceNumber;
        if (assignedSequenceNumber >= providerInfo.endSequenceNumber)
@>             revert EntropyErrors.OutOfRandomness();
…
    } 
```

## Impact Details

Because neither the `entropyProvider` state variable in the `Yeetback` contract nor the `yeetback` state variable in the `Yeetcontract` can be changed, if the `entropyProvider` runs out of randomness, the Yeet protocol will not be able to start a new round and distribute the winnings from the last round until the `entropyProvider` supplies new random numbers. In case the `entropyProvider` get blacklisted or decides to quit being a provider, the Yeet protocol will be DOSed permanently.

## Recommendation

Add a function to `Yeetback` which allows the owner to change the `entropyProvider` state variable.\
Also consider adding a function to the `Yeet` contract which allows the owner to change the `yeetback` state variable.

## References

<https://github.com/pyth-network/pyth-crosschain/blob/c905fbe1a8bc1d78bcae483e9bca6009db6d9be1/target\\_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L202C4-L251C6>

<https://docs.pyth.network/entropy/protocol-design>

## Proof of Concept

## POC (step by step)

Scenario 1:

* Yeetback is deployed and `entropyProvider` is set to the address of the `initalEntroyProvider`
* The `initalEntroyProvider` supplies 100 random numbers on chain which are all used up by calls from users of the Entropy protocol
* One yeet round of the Yeet protocol is over and `restart()` is called
* since all random numbers of the `entropyProvider` are used up, the call to `restart()` reverts and the Yeet protocol can not start a new round and distribute the winnings from the last round until the provider supplies new random numbers\
  Scenario 2:
* Yeetback is deployed and `entropyProvider` is set to the address of the `initalEntroyProvider`
* The `initalEntroyProvider` is blacklisted and removed from the Entropy protocol
* one yeet round of the Yeet protocol is over and `restart()` is called
* call reverts because the provider is no longer available and the yeet protocol is permanently DOSed and cannot start a new round and distribute the winnings from the last round
