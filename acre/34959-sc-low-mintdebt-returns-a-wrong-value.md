# #34959 \[SC-Low] \`mintDebt\` returns a wrong value

**Submitted on Sep 1st 2024 at 21:26:18 UTC by @Bx4 for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34959
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * wrong return value

## Description

## Brief/Intro

In \`mintDebt\` it returns \`shares\` but it is rather supposed to return assets

## Vulnerability Details

In \`mintDebt\` it returns \`shares\` as seen below \`return shares;\` meanwhile it is stated in the function natspec to return assets as seen below \`/// @return assets The debt amount in asset taken for the shares minted.\` Also, it is seen in the function declaration that it returns assets as shown below \` function mintDebt( uint256 shares, address receiver) public whenNotPaused returns (uint256 assets){} \` However the return statement declared last will overwrite it.

## Impact Details

it will return a wrong value and it will break the ERC 4626 invariant because it is stated that mint functions return assets as seen in this [link](https://ethereum.org/en/developers/docs/standards/tokens/erc-4626/#mint).

## References

<https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L307>

## Proof of Concept

## Proof of Concept

from the comment(@return) and the return value of the \`mintDebt\` function below

\`\`\`solidity /// @return assets The debt amount in asset taken for the shares minted. function mintDebt( uint256 shares, address receiver ) public whenNotPaused returns (uint256 assets) {

...

@-> return shares; } \`\`\`

we can deduce that the return value is supposed to be assets and yes in the function declaration it returns \`uint256 assets\`, However the last return statement will overwrite it and return shares
