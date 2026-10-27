# #45554 \[SC-Medium] Fee loss during Agent's feeBIPS reduction in \`selfMint\` function

**Submitted on May 16th 2025 at 17:02:47 UTC by @holydevoti0n for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45554
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

When an agent reduces their `feeBIPS` between making an underlying payment and executing `selfMint,` fees paid on the underlying chain are lost. This creates a gap where fees aren't properly captured in the protocol, resulting in a loss of funds for collateral pool participants.

## Vulnerability Details

Unlike the `executeMinting` function which stores the fee amount at the time of collateral reservation, the `selfMint` function calculates fees at **execution time** based on the agent's current `feeBIPS` value: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/Minting.sol#L92>

```solidity
function selfMint(
    IPayment.Proof calldata _payment,
    address _agentVault,
    uint64 _lots
)
    internal
{
    ...
    uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
    uint256 mintValueUBA = Conversion.convertAmgToUBA(valueAMG);
@>    uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
  ...
    
    if (_lots > 0) {
@>        _performMinting(agent, MintingType.SELF_MINT, 0, msg.sender, valueAMG, receivedAmount, poolFeeUBA);
    }
}
```

The `calculateCurrentPoolFeeUBA` function uses the agent's current `feeBIPS` to calculate the fee:

```solidity
function calculateCurrentPoolFeeUBA(
    Agent.State storage _agent,
    uint256 _mintingValueUBA
)
    internal view
    returns (uint256)
{
@>    uint256 mintingFeeUBA = _mintingValueUBA.mulBips(_agent.feeBIPS);
    return _calculatePoolFeeUBA(mintingFeeUBA, _agent.poolFeeShareBIPS);
}
```

### Example

1. An agent has feeBIPS set to 5% (500 basis points)
2. The agent makes a payment on the underlying chain that includes this 5% fee
3. The agent has his feeBIPS decreased to 0% (which requires announcement and execution, taking approximately 1 hour based on the current data from mainnet)
4. After the `feeBIPS` reduction is completed, the agent calls `selfMint`
5. When `selfMint` executes, it calculates `poolFeeUBA` based on the current `feeBIPS` (0%), resulting in zero fees
6. The protocol mints 0 fee tokens to the collateral pool:

```solidity
Globals.getFAsset().mint(address(_agent.collateralPool), _poolFeeUBA);
_agent.collateralPool.fAssetFeeDeposited(_poolFeeUBA);
```

7. The fees paid on the underlying chain are effectively lost, never making it into the protocol

Notice this vulnerability does not affect the `executeMinting` function because it uses a different approach. In `executeMinting`, the fee is calculated and stored at the time of collateral reservation:

```solidity
// From Minting.sol - executeMinting function
function executeMinting(
    IPayment.Proof calldata _payment,
    uint64 _crtId
)
    internal
{
    CollateralReservation.Data storage crt = CollateralReservations.getCollateralReservation(_crtId);
    ...
     // @audit crt stores the fee
    _performMinting(agent, MintingType.PUBLIC, _crtId, crt.minter, crt.valueAMG,
        uint256(_payment.data.responseBody.receivedAmount), calculatePoolFeeUBA(agent, crt));
}
```

The `calculatePoolFeeUBA` function for collateral reservations uses the fee stored in the reservation:

```solidity
function calculatePoolFeeUBA(
    Agent.State storage _agent,
    CollateralReservation.Data storage _crt
)
    internal view
    returns (uint256)
{
    ...
@>    return _calculatePoolFeeUBA(_crt.underlyingFeeUBA, poolFeeShareBIPS);
}
```

## Impact Details

Loss of funds for the collateral pool participants.

## Proof of Concept

1. An agent has feeBIPS set to 5% (500 basis points)
2. The agent makes a payment on the underlying chain that includes this 5% fee
3. The agent has his `feeBIPS` decreased to 0% (which requires announcement and execution, taking approximately 1 hour based on the current data from mainnet)
4. After the `feeBIPS` reduction is completed, the agent calls `selfMint`
5. When `selfMint` executes, it calculates `poolFeeUBA` based on the current `feeBIPS` (0%), resulting in zero fees
6. The protocol mints 0 fee tokens to the collateral pool:

```solidity
Globals.getFAsset().mint(address(_agent.collateralPool), _poolFeeUBA);
_agent.collateralPool.fAssetFeeDeposited(_poolFeeUBA);
```

7. The fees paid on the underlying chain are effectively lost, never making it into the protocol
