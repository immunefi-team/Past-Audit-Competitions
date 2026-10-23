# 42439 sc insight insight report for stakev2 contract

## #42439 \[SC-Insight] Insight Report for StakeV2 contract

**Submitted on Mar 23rd 2025 at 22:59:30 UTC by @pxng0lin for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42439
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**

### Description

## \[LOW-01] Unused Local Variable

### Severity: Low

#### Affected Contract

`StakeV2.sol`

#### Description

In the `executeRewardDistribution` function (line 195), the local variable `_islandTokens` is declared and assigned a value from the return of `zapper.zapInNative`, but it's never used in the function.

```solidity
(uint256 _islandTokens, uint256 vaultSharesMinted) =
                        zapper.zapInNative{value: amountToDistribute}(swap0, swap1, stakingParams, vaultParams);
```

#### Impact

While not a security vulnerability, unused variables make the code less clean and potentially indicate incomplete implementation or a misunderstanding of the function's purpose.

#### Recommendation

Remove the unused variable by replacing the declaration with:

```solidity
(, uint256 vaultSharesMinted) =
                        zapper.zapInNative{value: amountToDistribute}(swap0, swap1, stakingParams, vaultParams);
```

If there's a specific reason for capturing the `_islandTokens` value (for example, for future use), consider adding a comment explaining why it's being captured but not used.

***

## \[LOW-02] Missing Zero Address Validation in Constructor

### Severity: Low

#### Affected Contract

`StakeV2.sol`

#### Description

The constructor accepts `_stakingToken`, `_zapper`, and `_wbera` addresses but doesn't validate that these aren't the zero address:

```solidity
constructor(IERC20 _stakingToken, IZapper _zapper, address owner, address initialManager, IWETH _wbera) Manager(owner, initialManager) {
    stakingToken = _stakingToken;
    zapper = _zapper;
    wbera = _wbera;
}
```

#### Impact

Setting any of these critical components to the zero address would render the contract entirely unusable and necessitate redeployment.

#### Recommendation

Add zero address validation in the constructor:

```solidity
constructor(IERC20 _stakingToken, IZapper _zapper, address owner, address initialManager, IWETH _wbera) Manager(owner, initialManager) {
    require(address(_stakingToken) != address(0), "StakeV2: stakingToken is zero address");
    require(address(_zapper) != address(0), "StakeV2: zapper is zero address");
    require(address(_wbera) != address(0), "StakeV2: wbera is zero address");
    stakingToken = _stakingToken;
    zapper = _zapper;
    wbera = _wbera;
}
```

***

### Gas Optimisation

## \[GAS-01] Multiple Approval Patterns in Claim Functions

### Severity: Gas Optimisation

#### Affected Contract

`StakeV2.sol`

#### Description

In each claim function (`claimRewardsInNative`, `claimRewardsInToken0`, etc.), there's a repetitive pattern of approving tokens to the zapper:

```solidity
IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
```

#### Impact

Duplicated code increases contract size and deployment costs. It also makes maintenance more difficult as changes must be implemented in multiple places.

#### Recommendation

Move the approval logic to the `_verifyAndPrepareClaim` function to eliminate code duplication and reduce gas costs associated with deployment. Update the function to handle the approval:

```solidity
function _verifyAndPrepareClaim(uint256 amountToClaim, IZapper.VaultRedeemParams calldata redeemParams)
private
returns (IZapper.VaultRedeemParams memory)
{
    // Existing code
    
    // Add approval here
    IERC20(redeemParams.vault).approve(address(zapper), amountToClaim);
    
    return IZapper.VaultRedeemParams({
        // existing params
    });
}
```

***

## \[GAS-02] Duplicated Validation Logic

### Severity: Gas Optimisation

#### Affected Contract

`StakeV2.sol`

#### Description

All claim functions (`claimRewardsInNative`, `claimRewardsInToken0`, etc.) perform the same setup and validation, resulting in duplicated code.

#### Impact

This results in increased contract size and gas costs for deployment. Additionally, it creates more complex maintenance requirements as changes must be made in multiple places.

#### Recommendation

Extract the common logic into a separate internal function that handles the setup for all claim operations:

```solidity
function _setupClaim(uint256 amountToWithdraw) internal returns (uint256) {
    _updateRewards(msg.sender);
    uint256 userReward = earned[msg.sender];
    require(userReward > 0, "No rewards to claim");
    require(amountToWithdraw <= userReward, "Amount to claim exceeds rewards earned");
    earned[msg.sender] -= amountToWithdraw;
    totalVaultShares -= amountToWithdraw;
    return amountToWithdraw;
}
```

### Proof of Concept

### Proof of Concept

Please see the main body of the report
