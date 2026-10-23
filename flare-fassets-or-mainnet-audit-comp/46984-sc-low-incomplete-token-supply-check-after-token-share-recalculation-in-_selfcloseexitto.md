# #46984 \[SC-Low] Incomplete Token Supply Check After Token Share Recalculation in \`\_selfCloseExitTo\`

**Submitted on Jun 7th 2025 at 08:43:37 UTC by @light279 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46984
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

In the `CollateralPool::_selfCloseExitTo` function, after the recalculation of \_tokenShare due to insufficient agent redemption capacity, no validation is performed to ensure the updated \_tokenShare satisfies the minimum token supply constraint (`MIN_TOKEN_SUPPLY_AFTER_EXIT`). This breaks an important invariant enforced earlier in the function and could result in an invalid or inconsistent state post-exit.

## Vulnerability Details

The `CollateralPool::_selfCloseExitTo` function initially validates the following condition:

```javascript
function _selfCloseExitTo(
        uint256 _tokenShare,
        bool _redeemToCollateral,
        address payable _recipient,
        string memory _redeemerUnderlyingAddress,
        address payable _executor
    )
        private
    {
        require(_tokenShare > 0, "token share is zero");
        require(_tokenShare <= token.balanceOf(msg.sender), "token balance too low");
        AssetData memory assetData = _getAssetData();
@>        require(assetData.poolTokenSupply == _tokenShare ||
            assetData.poolTokenSupply - _tokenShare >= MIN_TOKEN_SUPPLY_AFTER_EXIT,
            "token supply left after exit is too low and non-zero");
        uint256 natShare = assetData.poolNatBalance.mulDiv(
            _tokenShare, assetData.poolTokenSupply); // poolTokenSupply >= _tokenShare > 0
        require(natShare > 0, "amount of sent tokens is too small");
        require(assetData.poolNatBalance == natShare ||
            assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
            "collateral left after exit is too low and non-zero");
        uint256 maxAgentRedemption = assetManager.maxRedemptionFromAgent(agentVault);
        uint256 requiredFAssets = _getFAssetRequiredToNotSpoilCR(assetData, natShare);
        // rare case: if agent has too many low-valued open tickets they can't redeem the requiredFAssets
        // in one transaction. In that case we lower/correct the amount of spent tokens and nat share.
        ...........
```

This check ensures that either:

* The user is exiting the entire pool (draining all tokens), or
* The remaining pool token supply after exit stays above a configured minimum.

However, if the `(maxAgentRedemption < requiredFAssets)`, the function recalculates natShare and \_tokenShare:

```javascript
function _selfCloseExitTo(
        uint256 _tokenShare,
        bool _redeemToCollateral,
        address payable _recipient,
        string memory _redeemerUnderlyingAddress,
        address payable _executor
    )
        private
    {
        ................
        if (maxAgentRedemption < requiredFAssets) {
            // natShare and _tokenShare decrease!
            requiredFAssets = maxAgentRedemption;
@>            natShare = _getNatRequiredToNotSpoilCR(assetData, requiredFAssets);
@>            require(natShare > 0, "amount of sent tokens is too small after agent max redemption correction");
            require(assetData.poolNatBalance == natShare ||
                assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
                "collateral left after exit is too low and non-zero");
            // poolNatBalance >= previous natShare > 0
@>            _tokenShare = assetData.poolTokenSupply.mulDiv(natShare, assetData.poolNatBalance);
            emit IncompleteSelfCloseExit(_tokenShare, requiredFAssets);
        }
        // get owner f-asset fees to be spent (maximize fee withdrawal to cover the potentially necessary f-assets)
        uint256 fAssetFees = _fAssetFeesOf(assetData, msg.sender);
        (uint256 debtFAssetFeeShare, uint256 freeFAssetFeeShare) = _getDebtAndFreeFAssetFeesFromTokenShare(
            assetData, msg.sender, _tokenShare, TokenExitType.MAXIMIZE_FEE_WITHDRAWAL);
        // if owner f-asset fees do not cover the required f-assets, require additional f-assets
     ................

```

While the function re-checks the natShare constraint above:

```
require(assetData.poolNatBalance == natShare ||
    assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
    "collateral left after exit is too low and non-zero");
```

It fails to re-validate `_tokenShare` against the `MIN_TOKEN_SUPPLY_AFTER_EXIT` constraint. As a result, it is possible for:

```
assetData.poolTokenSupply - _tokenShare < MIN_TOKEN_SUPPLY_AFTER_EXIT
```

to occur silently, breaking pool invariants and possibly leading to unintended or unsafe behavior in downstream logic that assumes this minimum is preserved.

## Impact Details

* Users could unintentionally drain the pool below the minimum allowed token supply.
* Breaks protocol-level assumptions that rely on MIN\_TOKEN\_SUPPLY\_AFTER\_EXIT being respected.

## Recommended Fix

Add any relevant links to documentation or code:

After `_tokenShare` is recalculated in the `if (maxAgentRedemption < requiredFAssets)` block in function `CollateralPool::_selfCloseExitTo`, reapply the token supply check:

```diff
function _selfCloseExitTo(
        uint256 _tokenShare,
        bool _redeemToCollateral,
        address payable _recipient,
        string memory _redeemerUnderlyingAddress,
        address payable _executor
    )
        private
    {
        ................
        if (maxAgentRedemption < requiredFAssets) {
            // natShare and _tokenShare decrease!
            requiredFAssets = maxAgentRedemption;
            natShare = _getNatRequiredToNotSpoilCR(assetData, requiredFAssets);
            require(natShare > 0, "amount of sent tokens is too small after agent max redemption correction");
            require(assetData.poolNatBalance == natShare ||
                assetData.poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT,
                "collateral left after exit is too low and non-zero");
            // poolNatBalance >= previous natShare > 0
           _tokenShare = assetData.poolTokenSupply.mulDiv(natShare, assetData.poolNatBalance);
+          require(assetData.poolTokenSupply == _tokenShare ||
            assetData.poolTokenSupply - _tokenShare >= MIN_TOKEN_SUPPLY_AFTER_EXIT,
            "token supply left after exit is too low and non-zero");
            emit IncompleteSelfCloseExit(_tokenShare, requiredFAssets);
        }
        // get owner f-asset fees to be spent (maximize fee withdrawal to cover the potentially necessary f-assets)
        uint256 fAssetFees = _fAssetFeesOf(assetData, msg.sender);
        (uint256 debtFAssetFeeShare, uint256 freeFAssetFeeShare) = _getDebtAndFreeFAssetFeesFromTokenShare(
            assetData, msg.sender, _tokenShare, TokenExitType.MAXIMIZE_FEE_WITHDRAWAL);
        // if owner f-asset fees do not cover the required f-assets, require additional f-assets
     ................

```

## Proof of Concept

## Proof of Concept

1: User initiates selfCloseExit

* User calls selfCloseExit(\_tokenShare, ...) with some initial \_tokenShare.

Checks performed:

* \_tokenShare > 0
* \_tokenShare <= user balance
* poolTokenSupply - \_tokenShare >= MIN\_TOKEN\_SUPPLY\_AFTER\_EXIT ✅
* natShare = poolNatBalance \* \_tokenShare / poolTokenSupply
* poolNatBalance - natShare >= MIN\_NAT\_BALANCE\_AFTER\_EXIT ✅ All checks pass.

2: Required F-assets are too high

* requiredFAssets = \_getFAssetRequiredToNotSpoilCR(...)
* But:

```
maxAgentRedemption < requiredFAssets
```

This triggers the fallback path to adjust.

3: Adjust `natShare` and recheck only collateral constraint

* requiredFAssets is capped to maxAgentRedemption
* A new natShare is computed via \_getNatRequiredToNotSpoilCR(...)
* Now, natShare is lower than the original
* Check re-applied:

```
require(poolNatBalance - natShare >= MIN_NAT_BALANCE_AFTER_EXIT)
```

4: \_tokenShare is recalculated — but no re-validation!

* \_tokenShare is now recomputed:

```
_tokenShare = poolTokenSupply * natShare / poolNatBalance

```

* No check is done again to verify:

```
poolTokenSupply - _tokenShare >= MIN_TOKEN_SUPPLY_AFTER_EXIT
```

Therefore, the new \_tokenShare could reduce the pool’s token supply below the minimum required threshold, violating the pool’s safety invariants.
