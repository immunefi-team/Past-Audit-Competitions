# #46702 \[SC-Insight] \`executeMinting()\` Enables Cross-Contract Reentrancy to Manipulate Collateral Pool Pricing

**Submitted on Jun 3rd 2025 at 16:05:10 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46702
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

The `executeMinting()` function in the Minting contract introduces a cross-contract reentrancy vulnerability that allows attackers to trick and manipulate the collateral pool’s pricing model and extract a disproportionate number of pool tokens. This stems from the order at which the execution/performing the minting operation and updating the agent's state before releasing the associated collateral reservation, creating a temporary imbalance in dependent state variables.

```solidity
function executeMinting(
        IPayment.Proof calldata _payment,
        uint64 _crtId,
    )
        internal
    {
.........................................
        uint256 mintValueUBA = Conversion.convertAmgToUBA(crt.valueAMG);
        require(_payment.data.responseBody.receivedAmount >= SafeCast.toInt256(mintValueUBA + crt.underlyingFeeUBA),
            "minting payment too small");
            
        // we do not allow payments before the underlying block at requests, because the payer should have guessed
        // the payment reference, which is good for nothing except attack attempts
        require(_payment.data.responseBody.blockNumber >= crt.firstUnderlyingBlock,
            "minting payment too old");
        // mark payment used
        AssetManagerState.get().paymentConfirmations.confirmIncomingPayment(_payment);
        // execute minting
    @>>    _performMinting(agent, MintingType.PUBLIC, _crtId, crt.minter, crt.valueAMG,
            uint256(_payment.data.responseBody.receivedAmount), calculatePoolFeeUBA(agent, crt));
        // pay to executor if they called this method
        uint256 unclaimedExecutorFee = crt.executorFeeNatGWei * Conversion.GWEI;

        if (msg.sender == crt.executor) {
            // safe - 1) guarded by nonReentrant in AssetManager.executeMinting, 2) recipient is msg.sender
      @>>      Transfers.transferNAT(crt.executor, unclaimedExecutorFee);
            unclaimedExecutorFee = 0;
        }

        // pay the collateral reservation fee (guarded against reentrancy in AssetManager.executeMinting)
        CollateralReservations.distributeCollateralReservationFee(agent,
            crt.reservationFeeNatWei + unclaimedExecutorFee);
            
        // cleanup 
@>>        CollateralReservations.releaseCollateralReservation(crt, _crtId);   // crt can't be used after this
    }
```

Now you see the process of the execution:

1. We perform the minting, this increases the `mintedAMG` of the agent

```solidity
Agents.createNewMinting(_agent, _mintValueAMG + poolFeeAMG);
```

```solidity
 function createNewMinting(
        Agent.State storage _agent,
        uint64 _valueAMG
    )
        internal
    {
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        // Add value with dust, then take the whole number of lots from it to create the new ticket,
        // and the remainder as new dust. At the end, there will always be less than 1 lot of dust left.
        uint64 valueWithDustAMG = _agent.dustAMG + _valueAMG;
        uint64 newDustAMG = valueWithDustAMG % settings.lotSizeAMG;
        uint64 ticketValueAMG = valueWithDustAMG - newDustAMG;
        // create ticket and change dust
        allocateMintedAssets(_agent, _valueAMG);
        if (ticketValueAMG > 0) {
            createRedemptionTicket(_agent, ticketValueAMG);
        }
        changeDust(_agent, newDustAMG);
    }
```

2. We then transfer the executor Fee

```solidity
 if (msg.sender == crt.executor) {
            // safe - 1) guarded by nonReentrant in AssetManager.executeMinting, 2) recipient is msg.sender
            Transfers.transferNAT(crt.executor, unclaimedExecutorFee);
            unclaimedExecutorFee = 0;
        }
```

3. We then now finally release Collateral Reservation the `reservedAMG` is now reduced

```solidity
CollateralReservations.releaseCollateralReservation(crt, _crtId);   // crt can't be used after this
```

Within Step 1 and Step 3 we are having an double-accounting of the `backedFasset` of the agent and this can be very dangerous as the attacker can re-enter into another contract which is the `CollateralPool` and enjoy the top-up discount.

```solidity
function _collateralToTokenShare(
        AssetData memory _assetData,
        uint256 _collateral
    )
        internal view
        returns (uint256)
    {
        bool poolConsideredEmpty = _assetData.poolNatBalance == 0 || _assetData.poolTokenSupply == 0;
        // calculate nat share to be priced with topup discount and nat share to be priced standardly
@>>        uint256 _aux = (_assetData.assetPriceMul * _assetData.agentBackedFAsset).mulBips(topupCollateralRatioBIPS);
        uint256 natRequiredToTopup = _aux > _assetData.poolNatBalance * _assetData.assetPriceDiv ? //@audit what if poolnatbalnce is zero
            _aux / _assetData.assetPriceDiv - _assetData.poolNatBalance : 0;
        uint256 collateralForTopupPricing = Math.min(_collateral, natRequiredToTopup); //@audit  what if _collateral is zero
        uint256 collateralAtStandardPrice = MathUtils.subOrZero(_collateral, collateralForTopupPricing);

        uint256 collateralAtTopupPrice = collateralForTopupPricing.mulDiv(
            SafePct.MAX_BIPS, topupTokenPriceFactorBIPS);

        uint256 tokenShareAtTopupPrice = poolConsideredEmpty ?
            collateralAtTopupPrice : _assetData.poolTokenSupply.mulDiv(
                collateralAtTopupPrice, _assetData.poolNatBalance);

        uint256 tokenShareAtStandardPrice = poolConsideredEmpty && tokenShareAtTopupPrice == 0 ?
            collateralAtStandardPrice : (_assetData.poolTokenSupply + tokenShareAtTopupPrice).mulDiv(
                collateralAtStandardPrice, _assetData.poolNatBalance + collateralForTopupPricing);

        return tokenShareAtTopupPrice + tokenShareAtStandardPrice;
    }
```

## Impact

The attacker re-enters into `collateralpool` and receives more pool tokens than they should due to the inflated `agentBackedFAsset` value, This effectively gives them a discount on their collateral deposit. The vulnerability exists because the collateral reservation is released after the minting operation, hence attacker can leverage this get more tokens than they should.

## Recommendation

I will recommend that the executor fee is only paid after the collateral Reservation is released to prevent against this cross-contract reentrancy leverage.

```solidity
   // cleanup 
       CollateralReservations.releaseCollateralReservation(crt, _crtId);   // crt can't be used after this

   if (msg.sender == executor) {
            // safe - 1) guarded by nonReentrant in AssetManager.executeMinting, 2) recipient is msg.sender
       Transfers.transferNAT(executor, unclaimedExecutorFee);
            unclaimedExecutorFee = 0;
        }

        // pay the collateral reservation fee (guarded against reentrancy in AssetManager.executeMinting)
        CollateralReservations.distributeCollateralReservationFee(agent,
            reservationFeeNatWei + unclaimedExecutorFee);
            
```

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L25-L71>

## Proof of Concept

## Proof of Concept

The actor first calls for reservation in an agent Vault, makes the payment in the underlying chain and follows this step, note the malicious actor sets the executor field to a malicious contract that carry out this attack.

1. Call `executeMinting()`:

Updates `agent.mintedAMG`, increasing `agentBackedFAsset`

These updates affect `natRequiredToTopup` in\
`CollateralPool. _collateralToTokenShare()`, increasing the amount of discount-eligible collateral

2. Before `releaseCollateralReservation()` is called:

The attacker immediately deposits collateral into the pool using `CollateralPool.enter()`

The inflated `agentBackedFAsset` results in more of the attacker’s collateral being priced with the discounted top-up factor

The attacker receives excess pool tokens

Finally, `releaseCollateralReservation()` is called, returning the system to a clean state—but the attacker already extracted the gains
