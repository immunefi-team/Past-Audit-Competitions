# 26519 - \[SC - Insight] Consider introducing the ability to change requ...

Submitted on Dec 4th 2023 at 12:21:59 UTC by @Another for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26519

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Consider introducing the ability to change `required` while adding owners

## Description

## Bug Description

Take a look at [MultiSigWallet#L125-156](https://www.contractreader.io/contract/mainnet/0x2028834B2c0A36A918c10937EeA71BE4f932da52)

```solidity
    /// @dev Allows to add a new owner. Transaction has to be sent by wallet.
    /// @param owner Address of new owner.
    function addOwner(address owner)
        public
        onlyWallet
        ownerDoesNotExist(owner)
        notNull(owner)
        validRequirement(owners.length + 1, required)
    {
        isOwner[owner] = true;
        owners.push(owner);
        OwnerAddition(owner);

      /// @dev Allows to remove an owner. Transaction has to be sent by wallet.
    /// @param owner Address of owner.
    function removeOwner(address owner)
        public
        onlyWallet
        ownerExists(owner)
    {
        isOwner[owner] = false;
        for (uint i=0; i<owners.length - 1; i++)
            if (owners[i] == owner) {
                owners[i] = owners[owners.length - 1];
                break;
            }
        owners.length -= 1;
        if (required > owners.length)
            changeRequirement(owners.length);
        OwnerRemoval(owner);
    }
```

At present implementation even if wallet adds ten more owners `required` would be stuck with the value it was initialized with, where as it would be good to note that while removing the owners the requirement could be reduced, this still seems like a downside to protocol logic.

## Impact

Borderline low/info, since if addressews are only ever added, i.e the logic still being the same that previous owners are still owners but new owners need to be added, therre is obviously a need to rearrrange the ration between the amount of owners and the valid signers that are needed to pass a decision, but this is not available.

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

Currently the value for `required` only changes if the owner array decreases, which means that if 10 people get added as new owners the ratio of the requirement and the amount of owners would be massively deflated.

## References
