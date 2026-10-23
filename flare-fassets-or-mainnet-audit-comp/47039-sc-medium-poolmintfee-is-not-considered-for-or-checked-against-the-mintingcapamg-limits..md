# #47039 \[SC-Medium] \`poolMintFee\` is not considered for or checked against the\`mintingCapAMG\` limits.

**Submitted on Jun 8th 2025 at 11:05:07 UTC by @farman1094 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47039
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Protocol insolvency
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

There is mechanism or of `Minting::checkMintingCap` which make sure the `Fassets` is not minted more then the limit, but there is multiple places where the minting cap is not considered for the mint Amount.

## Vulnerability Details

The function `Minting::checkMintingCap` in the Minting.sol contract is used to enforce a global limit (cap) on the total amount of assets that can be minted by the protocol. But there are multiple places at the time of mint of pool fee is not considered or check against the total cap, if that is under the global limit.

There is multiple places it is happening.

* `MintingFacet::selfMint`: This function is used by agent to mint the fassets (FXRP) for himself.

```
        uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
        checkMintingCap(valueAMG);
```

Here we only checking the `amountToMint` against the total minting, `poolFeeUBA` is not included in this. the actual amount what minted by agent is `poolFeeUBA` + `valueAMG`.

* `MintingFacet::mintFromFreeUnderlying` The same thing happening here at the time of check `poolFeeUBA` is not considered, and later minted amount was different

instead what should be happen is, like we checking for `CollateralReservations::reserveCollateral` checking the both amount, value + fee

```
        _reserveCollateral(agent, valueAMG + _currentPoolFeeAMG(agent, valueAMG));

// _reserveCollateral
        Minting.checkMintingCap(_reservationAMG);
```

The same type of thing also happening in confirmations of redemptions. `RedemptionConfirmations::confirmRedemptionPayment` After the redemption has completed we minting the pool share of redemption fees through function `_mintPoolFee`,

```
        uint256 poolFeeUBA = _request.underlyingFeeUBA.mulBips(_request.poolFeeShareBIPS);
        if (poolFeeUBA > 0) {
            Agents.createNewMinting(_agent, Conversion.convertUBAToAmg(poolFeeUBA));
            Globals.getFAsset().mint(address(_agent.collateralPool), poolFeeUBA);
            _agent.collateralPool.fAssetFeeDeposited(poolFeeUBA);
            emit IAssetManagerEvents.RedemptionPoolFeeMinted(_agent.vaultAddress(), _redemptionRequestId, poolFeeUBA);
        }
```

here's also the the cap is not considered. There can be possibility, between the redemption requested and completed. There are assets minted or reserved by any other agent, later at the time of `executionMinting` the cap is not considered as it checked already at the time of reservation.

## Impact Details

**Even if the pool fee amount is small, not including it in the minting cap check can still have meaningful negative consequences**

* Not including the pool fee in the cap check undermines the entire purpose of the minting cap, breaking the invariant.
* The cap’s purpose is to be a strict upper bound. Any breach—no matter how small—means the protocol’s guarantees are broken.

## Proof of Concept

## Proof of Concept

There are multiple incident where this is happening, I am explaining the POC one of the case, however all the cases has highlighted above.

1. Agent complete the payment, on his underlying address.
2. Agent would call `MintingFacet::selfMint` with proof and details.
3. Then this function has check underside, which call `checkMintingCap`. But that's only look for the `valueAMG` which exclude the `poolFeeShare`which also going to mint at the time of minting.

```
        checkMintingCap(valueAMG);
```

In `_performMinting` function

```
        Globals.getFAsset().mint(_minter, mintValueUBA);
        Globals.getFAsset().mint(address(_agent.collateralPool), _poolFeeUBA);
```

So there assets can me minted more then `mintingCapAMG`
