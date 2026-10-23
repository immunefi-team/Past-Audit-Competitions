# 60335 sc insight missing or misleading code comments causes confusion and may lead to unnecessary code changes

**Submitted on Nov 21st 2025 at 13:45:14 UTC by @KKam86 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60335
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**

## Description

## Brief/Intro

During code review I found several missing or misleading code comments which reduce readability and understanding of the codebase. Mismatch between the implementation and the documentation may cause unnecessary fixes by developers which in turn may introduce errors.

## Vulnerability Details

1. Missing namespace NatSpec tag for ERC-7201:

Currently, the `Stargate` and `StargateNFT` contracts does not follow the NatSpec tag specification that should be annotated to the contract's struct for ERC-7201. According to ERC-7201:

`A namespace in a contract should be implemented as a struct type. These structs should be annotated with the NatSpec tag @custom:storage-location <FORMULA_ID>:<NAMESPACE_ID>, where <FORMULA_ID> identifies a formula used to compute the storage location where the namespace is rooted, based on the namespace id.`

Namespace structs `StargateStorage` in `Stargate` and `StargateNFTStorage` in `DataTypes` should be annotated with following Natspec comments:

a) `StargateStorage`: `/// @custom:storage-location erc7201:storage.Stargate`

b) `StargateNFTStorage`: `/// @custom:storage-location erc7201:storage.StargateNFT`

2. Wrong @return Natspec comment in `StargateNFT._update()` function:

Overrided internal function `_update()` have following comment:

```solidity
/// @return the address of the new owner
```

This is not true. First function have custom logic for removing the manager of the token and then the `_update()` function of the parent contract `ERC721PausableUpgradeable` is called with the use of `super` keyword:

```solidity
return super._update(_to, _tokenId, _auth);
```

As can be observed in `ERC721PausableUpgradeable`, `_update()` function in turn calls parent `_update()` function of `ERC721Upgradeable`. According to comment and return value in `ERC721Upgradeable` this function returns previous owner and not the new owner of the token:

```solidity
* @dev Transfers `tokenId` from its current owner to `to`, or alternatively mints (or burns) if the current owner
* (or `to`) is the zero address. Returns the owner of the `tokenId` before the update.
```

```solidity
function _update(address to, uint256 tokenId, address auth) internal virtual returns (address) {
    ERC721Storage storage $ = _getERC721Storage();
    address from = _ownerOf(tokenId);
    --------- CODE
    return from;
}
```

This function is used by the `transferFrom()` function where returned address value is named `previousOwner`:

```solidity
        // Setting an "auth" arguments enables the `_isAuthorized` check which verifies that the token exists
        // (from != 0). Therefore, it is not needed to verify that the return value is not 0 here.
        address previousOwner = _update(to, tokenId, _msgSender());
```

Also the implementation of next parent contract function `ERC721EnumerableUpgradeable._update()` returns `previousOwner` instead of new owner:

```solidity
    function _update(
        address to,
        uint256 tokenId,
        address auth
    ) internal virtual override returns (address) {
        address previousOwner = super._update(to, tokenId, auth);
        ---- CODE
        return previousOwner;
```

Fix the comment associated with `StargateNFT._update()` function:

```solidity
/// @return the address of the previous owner
```

3. Outdated comments in `Stargate` contract:

Above the functions `pause()` and `unpause()` following section comment appears:

```solidity
>>>    // ---------- Admin setters ---------- //

    /// @inheritdoc IStargate
    function pause() external onlyRole(PAUSER_ROLE) {
        _pause();
    }

    /// @inheritdoc IStargate
    function unpause() external onlyRole(PAUSER_ROLE) {
        _unpause();
    }
```

Functions `pause()` and `unpause()` are not an admin setters. Admin has an assigned role `DEFAULT_ADMIN_ROLE` (only role granted in `initialize()` function):

```solidity
_grantRole(DEFAULT_ADMIN_ROLE, params.admin);
```

and in `/// @inheritdoc IStargate` we can find following comments:

```solidity
    // ---------- Pausable ------------ //

    /// @notice Pauses the StargateNFT contract
    /// @dev Only the DEFAULT_ADMIN_ROLE can call this function
    /// @dev Pausing the contract will prevent minting
    function pause() external;

    /// @notice Unpauses the StargateNFT contract
    /// @dev Only the DEFAULT_ADMIN_ROLE can call this function
    function unpause() external;
```

stating that functions for pausing and unpausing can be called only by the `DEFAULT_ADMIN_ROLE`. This is not true because these functions can be called only by the address with new `PAUSER_ROLE` which usually is not the same address with `DEFAULT_ADMIN_ROLE`.

Change section comment `// ---------- Admin setters ---------- //` to more appropriate `// ---------- Pauser setters ---------- //` and update comments in `IStargate` interface.

4. Functions have `private` visibility but comments states that they are `internal`

Functions `_delegate()` and `_claimRewards()` from `Stargate` contract are private but Natspec comments states that they are `internal`. `Internal` functions can be used in derived/child contracts and `private` can't be accessed. This can be misleading information:

```solidity
    /// @notice Internal function to claim the rewards for a token
    /// @param _tokenId - the token ID
    function _claimRewards(StargateStorage storage $, uint256 _tokenId) private {}
```

Fix the comments associated with listed functions. For example other `private` functions in `Stargate` like `_getDelegationStatus` have visibility matching to the associated comments.

5. Incorrect comment about token manager privileges after removal from the token:

Documentation comment about `StargateNFT.removeTokenManager()` in `IStargateNFT` interface (`/// @inheritdoc IStargateNFT`) states that removed manager `can still claim rewards for themselves, transfer, delegate or unstake the token`.

According to Stargate documentation `Stargate's NFT management feature enables staking NFT owners to assign limited operational privileges to a secondary wallet address—called a Node Manager—without compromising ownership or control.`

Only owner of the NFT should always have full control for operations like claiming rewards for themselves, transfer, delegate or unstake the token. These functionalities should be accessible only to owner of the token.

Fix following comment associated with `StargateNFT.removeTokenManager()` in `IStargateNFT` interface:

```solidity
    /// @notice Removes a manager from a token
    /// @dev This manager can no longer use the token for voting on VeVote
>>> /// or in B3TR but it can still claim rewards for themselves, transfer,
    /// delegate or unstake the token
    /// @dev If a token has no manager, it will revert
    /// @param _tokenId The ID of the token to remove the manager from
    /// @dev Emits a {IStargateNFT.TokenManagerRemoved} event
    function removeTokenManager(uint256 _tokenId) external;
```

6. Function `updateLevelBoostPricePerBlock()` is removed from the `StargateNFT` contract and only the library call is used in initialization process:

In the documentation comment in `StargateNFT` contract we can find the information that several setters are removed to optimize the contract:

```solidity
/// Removed setters to optimize contract size: setStargateDelegation, setLegacyNodes, updateLevelCap, updateLevel
```

The `updateLevelBoostPricePerBlock()` function from `Levels` library is still used in the `StargateNFT` and called in `initializeV3` function (call to library):

```solidity
        for (uint256 i; i < levelIds.length; i++) {
            Levels.updateLevelBoostPricePerBlock($, levelIds[i], boostPricesPerBlock[i]);
        }
```

`initializeV3` function can be only called by `UPGRADER_ROLE` and according to comment in `Levels` `updateLevelBoostPricePerBlock()` function can be called only by the other role - `LEVEL_OPERATOR_ROLE` which is used in `StargateNFT` contract.

Function `updateLevelBoostPricePerBlock()` is not present in `StargateNFT` as external function which can be called by `LEVEL_OPERATOR_ROLE`. Because it is only called in `initializeV3` function consider adding information about removal of this function as setter in `StargateNFT`. Also update the comment in `Levels` library about this function.

7. Incorrect comments about updating effective stake in validators which have `UKNOWN` status:

Functions `unstake()` and `_delegate()` in `Stargate` contract have following comment and check in their implementations:

```solidity
>>>         // if the delegation is pending or the validator is exited or unknown
            // decrease the effective stake of the previous validator
            if (
                currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
                status == DelegationStatus.PENDING
            ) {}
```

Based on comment Validator with status `UNKNOWN` should be updated but the check omit that update. Validator can have `UNKNOWN` status which is declared in the contract as `constant` variable:

```solidity
    /// @dev Validator status constants for better readability
>>> uint8 private constant VALIDATOR_STATUS_UNKNOWN = 0;
    uint8 private constant VALIDATOR_STATUS_QUEUED = 1;
    uint8 private constant VALIDATOR_STATUS_ACTIVE = 2;
    uint8 private constant VALIDATOR_STATUS_EXITED = 3;
```

According to the Stargate documentation `UNKNOWN` status is `a fallback status, triggered by incorrect configuration or protocol errors` and based on that information this should be not normal case in `Validator` life cycle. Also in the audit report by Hacken there is information about the resolution for the one of the findings which is properly fixed:

```solidity
The finding is fixed in commit hash eb5b8cb after adding validator
status checks in the _delegate() function. The fix properly decreases
delegatorsEffectiveStake from the old validator when re-delegating from
PENDING delegation status or when the old validator has EXITED status
```

There is no information about how the functions should behave when Validator has `UNKNOWN` status and wheter or not such Validator should be updated. Based on the comment I assume Validator with `UNKNOWN` status is treated as with `EXITED` status here in mentioned functions but this may be not true and because I can't find more info about `UNKNOWN` status I will not suggest wheter the comment or check should be fixed.

## Impact Details

Incorrect or missing comments could cause confusion for developers, reviewers or auditors reading the codebase. This might lead to unnecessary code changes if developers will try "fix" the code to match the code comments or documentation.

## Proof of Concept

## Proof of Concept

If for example developer will try fix the `_update()` function in `StargateNFT` contract to return new owner (which is `to` address) then the other functions like `transferFrom()` will be affected and behave erroneously when developer will forget to override this function and not change their implementation.

1. Developer fixes the `_update()` function to match the comment: `/// @return the address of the new owner`:

```solidity
return _to // returns the new owner of the NFT (for simplicity)
```

2. Now `transferFrom()` function is affected (and all the functions using that function like `safeTransferFrom`):

```solidity
     * @dev See {IERC721-transferFrom}.
     */
    function transferFrom(address from, address to, uint256 tokenId) public virtual {
        if (to == address(0)) {
            revert ERC721InvalidReceiver(address(0));
        }
        // Setting an "auth" arguments enables the `_isAuthorized` check which verifies that the token exists
        // (from != 0). Therefore, it is not needed to verify that the return value is not 0 here.

        // @audit Reverts because `to` is returned - not the address returned from the call to `address from = _ownerOf(tokenId);`
 >>>    address previousOwner = _update(to, tokenId, _msgSender());
        if (previousOwner != from) {
            revert ERC721IncorrectOwner(from, tokenId, previousOwner);
        }
    }
```
