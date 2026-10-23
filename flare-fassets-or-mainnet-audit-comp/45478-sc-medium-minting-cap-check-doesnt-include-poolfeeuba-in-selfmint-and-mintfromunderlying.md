# #45478 \[SC-Medium] Minting Cap Check Doesn't Include \`poolFeeUBA\` in \`selfMint\` and \`mintFromUnderlying\`

**Submitted on May 15th 2025 at 09:21:14 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45478
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Theft of unclaimed yield

## Description

## Brief/Intro

When executing `selfMint` or `mintFromUnderlying` operations, the `checkMintingCap` function is called but fails to account for the `poolFeeUBA`. This oversight results in an inaccurate cap verification process.

## Vulnerability Details

Agent have an option to call `selfMint` and `mintFromunderlying` directly without collateral reservation to mint `fAssets` to their account. Following this path, the agent will only have to pay the pool fee.

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L90>

```solidity
function selfMint(
        IPayment.Proof calldata _payment,
        address _agentVault,
        uint64 _lots
    ) internal {
       ...SNIP...
        uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
  L::90-> checkMintingCap(valueAMG);
        
        uint256 mintValueUBA = Conversion.convertAmgToUBA(valueAMG);
        uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
        require(
            _payment.data.responseBody.standardPaymentReference ==
                PaymentReference.selfMint(_agentVault),
            "invalid self-mint reference"
        );
        require(
            _payment.data.responseBody.receivingAddressHash ==
                agent.underlyingAddressHash,
            "self-mint not agent's address"
        );
        require(
            _payment.data.responseBody.receivedAmount >=
                SafeCast.toInt256(mintValueUBA + poolFeeUBA),
            "self-mint payment too small"
        );
        require(
            _payment.data.responseBody.blockNumber >=
                agent.underlyingBlockAtCreation,
            "self-mint payment too old"
        );
        state.paymentConfirmations.confirmIncomingPayment(_payment);
        // case _lots==0 is allowed for self minting because if lot size increases between the underlying payment
        // and selfMint call, the paid assets would otherwise be stuck; in this way they are converted to free balance
        uint256 receivedAmount = uint256(
            _payment.data.responseBody.receivedAmount
        ); // guarded by require
        if (_lots > 0) {
            _performMinting(
                agent,
                MintingType.SELF_MINT,
                0,
                msg.sender,
                valueAMG,
                receivedAmount,
                poolFeeUBA
            );
        } else {
            UnderlyingBalance.increaseBalance(agent, receivedAmount);
            emit IAssetManagerEvents.SelfMint(
                _agentVault,
                false,
                0,
                receivedAmount,
                0
            );
        }
    }
```

As shown on line 90, the minting cap is checked using only `valueAMG` without including the `poolFeeUBA`. This is problematic because the total minting will include the pool fee, as demonstrated in the `_performMinting` function, potentially exceeding the minting limit.

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L205>

```solidity
 function _performMinting(
        Agent.State storage _agent,
        MintingType _mintingType,
        uint64 _crtId,
        address _minter,
        uint64 _mintValueAMG,
        uint256 _receivedAmountUBA,
        uint256 _poolFeeUBA
    ) private {
        uint64 poolFeeAMG = Conversion.convertUBAToAmg(_poolFeeUBA);
        Agents.createNewMinting(_agent, _mintValueAMG + poolFeeAMG);
        // update agent balance with deposited amount (received amount is 0 in mintFromFreeUnderlying)
        UnderlyingBalance.increaseBalance(_agent, _receivedAmountUBA);
        // perform minting
       
        uint256 mintValueUBA = Conversion.convertAmgToUBA(_mintValueAMG);
        
->       Globals.getFAsset().mint(_minter, mintValueUBA);
->      Globals.getFAsset().mint(address(_agent.collateralPool), _poolFeeUBA);

        ...SNIP...
    }
```

The regular `executeMinting` function does not have this issue because the minting cap for mintingValue+fee is already checked during the collateral reservation process, as shown in the code below.

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CollateralReservations.sol#L269>

```solidity
function reserveCollateral(
        address _minter,
        address _agentVault,
        uint64 _lots,
        uint64 _maxMintingFeeBIPS,
        address payable _executor,
        string[] calldata _minterUnderlyingAddresses
    ) internal {
        ...SNIP...
        
        uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
        _reserveCollateral(
            agent,
 @->           valueAMG + _currentPoolFeeAMG(agent, valueAMG)
        );
        
        ...SNIP...
    }
    
    function _reserveCollateral(
        Agent.State storage _agent,
        uint64 _reservationAMG
    ) private {
        AssetManagerState.State storage state = AssetManagerState.get();
 ->      Minting.checkMintingCap(_reservationAMG);
        _agent.reservedAMG += _reservationAMG;
        state.totalReservedCollateralAMG += _reservationAMG;
    }
```

## Impact Details

The minting cap can be exceeded, compromising the integrity of the system's economic constraints. Over-minting can lead to inflation of the fAsset, reducing its value and leading to a loss for the existing user. These additional minted fAssets can then be redeemed by an attacker, extracting value from the system and thus eating into the funds of other users.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L90>

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L205>

## Proof of Concept

## Proof of Concept

* Assume the minting cap is set to 100 (equivalent to 10 lots)
* An agent calls `selfMint` with 10 lots, where each lot represents 10 assets, totalling 100 assets
* The agent is required to pay 10 additional assets as a pool fee
* The function calls `checkMintingCap(100)`, which passes verification
* As a result, 110 fAssets are minted, exceeding the established minting cap of 100
