# #45910 \[SC-Medium] Changing collateral ratio makes Agents prone to liquidation

**Submitted on May 22nd 2025 at 11:47:40 UTC by @pseudoArtist for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45910
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Liquidation.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`setCollateralRatiosForToken` when called changes the important ratios for the collateral and when this is called without any prior information and done infavourably will make positions of the Agents unhealthy and prone to liquidation, There should be some grace time given to implement the changes like 24 hours such that Agents can get sufficient time to deposit more collateral to make there position healthy and above min liquidation threshold.

## Vulnerability Details

```solidity
    function setCollateralRatiosForToken(
        CollateralType.Class _collateralClass,
        IERC20 _token,
        uint256 _minCollateralRatioBIPS,
        uint256 _ccbMinCollateralRatioBIPS,
        uint256 _safetyMinCollateralRatioBIPS
    )
        external
        onlyAssetManagerController
    {
        CollateralTypes.setCollateralRatios(_collateralClass, _token,
            _minCollateralRatioBIPS, _ccbMinCollateralRatioBIPS, _safetyMinCollateralRatioBIPS);
    }
```

The function `setCollateralRatiosForToken` is called from the `AssetManagerController` to change the important ratios like `_minCollateralRatioBIPS` , `_ccbMinCollateralRatioBIPS` and `_safetyMinCollateralRatioBIPS`, It does so by calling `setCollateralRatios` on the `CollateralTypes` Library.

```solidity
    function setCollateralRatios(
        CollateralType.Class _collateralClass,
        IERC20 _token,
        uint256 _minCollateralRatioBIPS,
        uint256 _ccbMinCollateralRatioBIPS,
        uint256 _safetyMinCollateralRatioBIPS
    )
        internal
    {
        bool ratiosValid =
            SafePct.MAX_BIPS < _ccbMinCollateralRatioBIPS &&
            _ccbMinCollateralRatioBIPS <= _minCollateralRatioBIPS &&
            _minCollateralRatioBIPS <= _safetyMinCollateralRatioBIPS;
        require(ratiosValid, "invalid collateral ratios");
        // update
        CollateralTypeInt.Data storage token = CollateralTypes.get(_collateralClass, _token);
        token.minCollateralRatioBIPS = _minCollateralRatioBIPS.toUint32();
        token.ccbMinCollateralRatioBIPS = _ccbMinCollateralRatioBIPS.toUint32();
        token.safetyMinCollateralRatioBIPS = _safetyMinCollateralRatioBIPS.toUint32();
        emit IAssetManagerEvents.CollateralRatiosChanged(uint8(_collateralClass), address(_token),
            _minCollateralRatioBIPS, _ccbMinCollateralRatioBIPS, _safetyMinCollateralRatioBIPS);
    }
```

The only check it has is of `ratioValid` and if the ratios are set in such a way that it is valid it can be set easily. However while doing so the Agents face a serious consequence as some of the agent's positions which are borderline healthy or just above the liquidation threshold can become unhealthy and prone to liquidation.

`startLiquidation` --> `_upgradeLiquidationPhase` --> `_initialLiquidationPhaseForCollateral`

```solidity
    function _initialLiquidationPhaseForCollateral(
        uint256 _collateralRatioBIPS,
        uint256 _collateralIndex
    )
        private view
        returns (Agent.LiquidationPhase)
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        CollateralTypeInt.Data storage collateral = state.collateralTokens[_collateralIndex];
        if (_collateralRatioBIPS >= collateral.minCollateralRatioBIPS) {
            return Agent.LiquidationPhase.NONE;
        } else if (_collateralRatioBIPS >= collateral.ccbMinCollateralRatioBIPS) {
            return Agent.LiquidationPhase.CCB;
        } else {
            return Agent.LiquidationPhase.LIQUIDATION;
        }
    }
```

The call to `_initialLiquidationPhaseForCollateral` returns the LiquidationPhase, and upon updating the `minCollateralRatioBIPS` the position which was healthy will become unhealthy at once and will fall into `LIQUIDATION` phase and become liquidated immediately.

## Impact Details

Healthy positions of Agents will become prone to liquidation

## References

```solidity
    function setCollateralRatios(
        CollateralType.Class _collateralClass,
        IERC20 _token,
        uint256 _minCollateralRatioBIPS,
        uint256 _ccbMinCollateralRatioBIPS,
        uint256 _safetyMinCollateralRatioBIPS
    )
        internal
    {
        bool ratiosValid =
            SafePct.MAX_BIPS < _ccbMinCollateralRatioBIPS &&
            _ccbMinCollateralRatioBIPS <= _minCollateralRatioBIPS &&
            _minCollateralRatioBIPS <= _safetyMinCollateralRatioBIPS;
        require(ratiosValid, "invalid collateral ratios");
        // update
        CollateralTypeInt.Data storage token = CollateralTypes.get(_collateralClass, _token);
        token.minCollateralRatioBIPS = _minCollateralRatioBIPS.toUint32();
        token.ccbMinCollateralRatioBIPS = _ccbMinCollateralRatioBIPS.toUint32();
        token.safetyMinCollateralRatioBIPS = _safetyMinCollateralRatioBIPS.toUint32();
        emit IAssetManagerEvents.CollateralRatiosChanged(uint8(_collateralClass), address(_token),
            _minCollateralRatioBIPS, _ccbMinCollateralRatioBIPS, _safetyMinCollateralRatioBIPS);
    }
```

```solidity
    function _initialLiquidationPhaseForCollateral(
        uint256 _collateralRatioBIPS,
        uint256 _collateralIndex
    )
        private view
        returns (Agent.LiquidationPhase)
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        CollateralTypeInt.Data storage collateral = state.collateralTokens[_collateralIndex];
        if (_collateralRatioBIPS >= collateral.minCollateralRatioBIPS) {
            return Agent.LiquidationPhase.NONE;
        } else if (_collateralRatioBIPS >= collateral.ccbMinCollateralRatioBIPS) {
            return Agent.LiquidationPhase.CCB;
        } else {
            return Agent.LiquidationPhase.LIQUIDATION;
        }
    }
```

## Proof of Concept

## Proof of Concept

Note that this change can also be non deliberate but will result in the same issue.

* An agent is barely healthy under old ratios (e.g., 151% with old minCollateralRatioBIPS = 150%).
* The admin updates `minCollateralRatioBIPS` to 155%.
* The agent’s ratio (151%) is now below the new threshold (155%) → Agent is moved to LIQUIDATION.

Before Update : minCollateralRatioBIPS: 150%

```
Agent A:
Collateral: 1000 C1 tokens. , Debt: 600 f-assets.
Collateral Ratio: (1000 C1 / 600 f-assets) * 100 = 166.67%.
minCollateralRatioBIPS: 150% → Healthy (NORMAL).
```

```
After Update (minCollateralRatioBIPS → 160%)

Agent A:
Collateral ratio (166.67%) is still above 160% → Remains healthy.

Agent B:
Collateral: 900 C1 tokens.
Debt: 600 f-assets.
Collateral Ratio: (900 / 600) * 100 = 150%.

Now below new `minCollateralRatioBIPS` (160%) → Enters LIQUIDATION.
```
