# #45769 \[SC-Medium] Permanent blocking of Agent's fund by allowed minters

**Submitted on May 20th 2025 at 09:10:47 UTC by @pseudoArtist for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45769
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>
* **Impacts:**
  * Temporary freezing of funds

## Description

## Brief/Intro

Collateral reservation logic has a flaw which will lead to a malicious `allowedMinter` to act maliciously and block all the free collateral of an Agent for further use of genuine minters thus permanently breaking the minting process.

This is not OOS because it has already been confirmed from the team screenshot attached below for reference.

## Vulnerability Details

When minting the minters first need to reserve the collateral by calling `CollateralReservation::reserveCollateral()` which further calls the `reserveCollateral` in the `CollateralReservations` library.

The function `reserveCollateral()` does some basic checks, It also checks if the lots can be processed by checking the free collateral of the agent:

```solidity
 require(collateralData.freeCollateralLots(agent) >= _lots, "not enough free collateral");
```

and then calculates the lot size by:

```solidity
 uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
```

After calculating the amount of collateral required to be locked, it then moves on to reserve the free collaterals:

```solidity
  _reserveCollateral(agent, valueAMG + _currentPoolFeeAMG(agent, valueAMG));
```

The above reservation process looks something like this :

```solidity
  function _reserveCollateral(
        Agent.State storage _agent,
        uint64 _reservationAMG//@note this includes the lots*lotSizeAMG + poolFeeUBA
    )
        private
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Minting.checkMintingCap(_reservationAMG);//@note this checks if the total minting cap is exceeded in the total protocol and not for the agent
        _agent.reservedAMG += _reservationAMG;//@note this updates the reserved amount for the agent
        state.totalReservedCollateralAMG += _reservationAMG;//@audit-info the free collateral is updated in the next call to reserveCollateral.
    }
```

Now if we look at the above process it increases the `reservedAMG` by the `_reservationAMG` and which means it updates the amount of `reervedAMG` for the agent and based on this the `freeCollaterals` are also updated in the next call to the `reserveCollateral().`

Now let's take a look at how the fee are charged for reservations, If a minter wants to reserve the collateral he needs to pay a fee so when the minter fails to deliver the underlying assets he loses the fee which disincentivises him to act maliciously and block the free collateral. However there is a catch here, the fee are not charged for the alwaysAllowedMinters:

```solidity
// - only charge reservation fee for public minting, not for alwaysAllowedMinters on non-public agent

 // - poolCollateral is WNat, so we can use its price for calculation of CR fee
        uint256 reservationFee = agent.availableAgentsPos != 0
            ? _reservationFee(collateralData.poolCollateral.amgToTokenWeiPrice, valueAMG)
            : 0;
```

The `reservationFee` will be 0 for the `alwaysAllowedMinters` which gives them the power to act maliciously and when they call the function `reserveCollateral()` combined with the fact they aren't being charged any fee for it , they can block the whole collateral of an agent indefinitely without losing anything. An agent only have the option to call `rejectCollateralReservation` which will reject and free up the collateral but the malicious allowed minter can do this again until he is removed from the `alwaysAllowedMinters` list.

## Impact Details

Malicious minter can grief genuine minters for certain no of blocks and also block the capital of the agent for free.

## References

```solidity
// - poolCollateral is WNat, so we can use its price for calculation of CR fee
        uint256 reservationFee = agent.availableAgentsPos != 0
            ? _reservationFee(collateralData.poolCollateral.amgToTokenWeiPrice, valueAMG)
            : 0;

```

## Proof of Concept

## Proof of Concept

Step 1: `alwaysAllowedMinters` calls the `reserveCollateral()` and reserves the freeCollateral to its max capacity without any fee.

Step 2: Malicious Minter doesn't deliver the underlying tokens.

Step 3: An agent after waiting for certain time rejects the collateral reservations.

Step 4: Malicious minter repeats the same process again and keep dossing and griefing genuine minters for certain time unless he is removed from the always allowed minters list.
