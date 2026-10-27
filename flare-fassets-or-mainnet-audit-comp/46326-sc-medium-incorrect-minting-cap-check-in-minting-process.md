# #46326 \[SC-Medium] Incorrect Minting Cap Check in Minting Process

**Submitted on May 28th 2025 at 12:17:07 UTC by @aman for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46326
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Minting.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `checkMintingCap` function in `Minting.sol` only checks the `valueAMG` against the minting cap, but the actual minting process adds both `valueAMG` and `poolFeeAMG` to the agent's minted amount. This discrepancy allows the total minted amount to exceed the intended minting cap.

## Vulnerability Details

The vulnerable code is in `Minting.sol`:

```solidity
/fassets/contracts/assetManager/library/Minting.sol:75
 75:     function selfMint(
 76:         IPayment.Proof calldata _payment,
 77:         address _agentVault,
 78:         uint64 _lots
 79:     )
 80:         internal
 81:     {
.....
 90:         require(collateralData.freeCollateralLots(agent) >= _lots, "not enough free collateral");
 91:         uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
 92:         checkMintingCap(valueAMG); <----@
107:         if (_lots > 0) {
108:             _performMinting(agent, MintingType.SELF_MINT, 0, msg.sender, valueAMG, receivedAmount, poolFeeUBA);
109:         } else {

```

`_performMinting` function :

```solidity
/fassets/contracts/assetManager/library/Minting.sol:189
189:     function _performMinting(
190:         Agent.State storage _agent,
191:         MintingType _mintingType,
192:         uint64 _crtId,
193:         address _minter,
194:         uint64 _mintValueAMG,
195:         uint256 _receivedAmountUBA,
196:         uint256 _poolFeeUBA
197:     )
198:         private
199:     {
200:         uint64 poolFeeAMG = Conversion.convertUBAToAmg(_poolFeeUBA);
201:         Agents.createNewMinting(_agent, _mintValueAMG + poolFeeAMG); <----@ we add both value
```

Here we only check the `valueAMG` not the Fee which will also be added to `agent.mintedAMG`

```solidity
/fassets/contracts/assetManager/library/Minting.sol:139
139:     function checkMintingCap(
140:         uint64 _increaseAMG
141:     )
142:         internal view
143:     {
144:         AssetManagerState.State storage state = AssetManagerState.get();
145:         AssetManagerSettings.Data storage settings = Globals.getSettings();
146:         uint256 mintingCapAMG = settings.mintingCapAMG;
147:         if (mintingCapAMG == 0) return;     // minting cap disabled
148:         uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
149:         uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
150:         require(totalAMG + _increaseAMG <= mintingCapAMG, "minting cap exceeded");
151:     }
152:
```

## Impact Details

* **Severity**: Low
* **Impact**: Allows total minted amount to exceed the minting cap by the amount of pool fees
* **Scope**: Affects all minting operations in the system either via `reserveCollateral`, `mintFromFreeUnderlying` and `selfMint`

## References

* File: `contracts/assetManager/library/Minting.sol`
* Function: `checkMintingCap`
* Function: `_performMinting`

1. Fix the minting cap check to include pool fees:

```solidity
function checkMintingCap(
    uint64 _valueAMG,
    uint64 _poolFeeAMG
)
    internal view
{
    AssetManagerState.State storage state = AssetManagerState.get();
    AssetManagerSettings.Data storage settings = Globals.getSettings();
    uint256 mintingCapAMG = settings.mintingCapAMG;
    if (mintingCapAMG == 0) return;     // minting cap disabled
    uint256 totalMintedUBA = IERC20(settings.fAsset).totalSupply();
    uint256 totalAMG = state.totalReservedCollateralAMG + Conversion.convertUBAToAmg(totalMintedUBA);
    require(totalAMG + _valueAMG + _poolFeeAMG <= mintingCapAMG, "minting cap exceeded");
}
```

2. Update all calls to `checkMintingCap` to include pool fees

## Proof of Concept

## Proof of Concept

1. The `MintingCapAMG` is set to 1000.
2. An agent attempts to mint `valueAMG` of 1000.
3. The `PoolFeeAMG` is calculated as 100 (10% of the minting amount).
4. The code only checks the `valueAMG` against the minting cap, so the check `require(totalAMG + _increaseAMG <= mintingCapAMG, "minting cap exceeded")` passes because `0 + 1000 <= 1000`.
5. After the check, the system increases the agent's `mintedAMG` by `_valueAMG`, where `_valueAMG = valueAMG + feeAMG`.
6. The final value of `agent.MintedAMG` becomes 1100 (1000 + 100).

This demonstrates that despite the minting cap being set to 1000, the agent successfully minted 1100 AMG, exceeding the intended cap by 100 AMG. For simplicity, decimal and dust calculations have been ignored in this example.
