# 30555 - \[SC - Low] Precision loss when calculating the FLUX amount...

## Precision loss when calculating the FLUX amount required to ragequit for a token

Submitted on Apr 30th 2024 at 18:42:03 UTC by @MTNether for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30555

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of funds
* Temporary freezing of funds for 12 hours
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

### Brief/Intro

Fewer FLUX tokens than actual are calculated and burnt in the cooling down process due to the priority of division over multiplication.

### Vulnerability Details

Solidity rounds down the result of an integer division, and because of that, it is always recommended to multiply before dividing to avoid that precision loss. In the case of a prior division over multiplication, the final result may face serious precision loss as the first answer would face truncated precision and then multiplied to another integer.

The problem arises in the VotingEscrow's cooling down part. After performing the necessary checks inside the `startCooldown()` function, it checks the required FLUX amounts to ragequit:

```solidity
    function amountToRagequit(uint256 _tokenId) public view returns (uint256) {
        // amount of flux earned in one epoch
        uint256 oneEpochFlux = claimableFlux(_tokenId);

        // total amount of epochs in fluxMultiplier amount of years
        uint256 totalEpochs = fluxMultiplier * ((MAXTIME) / EPOCH);

        // based on one epoch, calculate total amount of flux over fluxMultiplier amount of years
        uint256 ragequitAmount = oneEpochFlux * totalEpochs;

        return ragequitAmount;
    }
```

It is evident that the `ragequitAmount` is calculated by the multiplication of claimable flux token amounts and the flux time ratio which is defined as:

```solidity
    totalEpochs = fluxMultiplier * ((MAXTIME) / EPOCH);
```

The variable `fluxMultiplier` is set to be `4`, the `MAXTIME`, and `EPOCH` are `365 days`, and `2 weeks` respectively.

The current implementation of the aforementioned definition has a hidden division before a multiplication that rounds down the whole expression.

This is bad as the precision loss can be significant, which leads to the contract calculating and burning less `ragequitAmount` than the actual.

At the Proof of Concept part, we can check this behavior precisely.

### Impact Details

Low `ragequitAmount` to ragequit inside the `amountToRagequit()` function is calculated leading to wrongly setting the minimum Flux amounts to ragequit and thus burning fewer tokens than actual.

### References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol?utm\\_source=immunefi#L345-L356>

## Recommended Mitigation Steps

Consider modifying the `ragequitAmount` calculation to prevent such precision loss and prioritize the multiplication over division:

```diff
    function amountToRagequit(uint256 _tokenId) public view returns (uint256) {
        // amount of flux earned in one epoch
        uint256 oneEpochFlux = claimableFlux(_tokenId);

        // total amount of epochs in fluxMultiplier amount of years
-       uint256 totalEpochs = fluxMultiplier * ((MAXTIME) / EPOCH);

        // based on one epoch, calculate total amount of flux over fluxMultiplier amount of years
-       uint256 ragequitAmount = oneEpochFlux * totalEpochs;
+       uint256 ragequitAmount = (oneEpochFlux * fluxMultiplier * MAXTIME) / EPOCH;

        return ragequitAmount;
    }
```

### Proof of Concept

You can run this code inside the Forge to see the difference between the results:

```solidity
    function test_precissionLoss() public {

        uint256 oneEpochFlux = claimableFlux;
        uint256 totalEpochs = fluxMultiplier * ((MAXTIME) / EPOCH);

        uint256 ragequitAmount_actual   = (oneEpochFlux * totalEpochs);
        uint256 ragequitAmount_accurate = (oneEpochFlux * fluxMultiplier * MAXTIME) / EPOCH;
        
        console.log("Current Implementation ", ragequitAmount_actual);
        console.log("Actual Implementation  ", ragequitAmount_accurate);
    }
```

The result would be: (for a sample and real `claimableFlux` of `156510000000` just for testing purpose)

```
     Current Implementation  16277040000000
     Actual Implementation   16321757142857
```

Thus, we can see that the actual implementation produces less ragequit amount than the precise method. This test shows a big difference between the two calculated ragequit amounts in the cooling down process.
