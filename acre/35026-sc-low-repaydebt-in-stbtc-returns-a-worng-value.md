# #35026 \[SC-Low] \`repayDebt\` in stbtc returns a worng value

**Submitted on Sep 2nd 2024 at 22:15:07 UTC by @Bx4 for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #35026
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * wrong return value

## Description

## Brief/Intro

from our preview function and natspec comment we can deduce that \`repayDebt\` is supposed to return assets but it returns shares

## Vulnerability Details

from the preview equivalent of the function; \`\`\`solidity function previewRepayDebt(uint256 shares) public view returns (uint256) { return convertToAssets(shares); } \`\`\` we can see that the final return statement returns assets by converting from shares to assets

furthermore when we read the comments of \`repayDebt\` we will find this line; \` /// @return assets The amount of debt in asset paid off.\` meaning the final return value that will be expected is assets, however there is \`return shares;\` at the end of the function overwriting everything and returning shares

## Impact Details

This returns a wrong value to the caller

## References

repayDebt comment - <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L346>

previewRepayDebt - <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L533-L535>

## Proof of Concept

## Proof of Concept

looking at \`repayDebt\` below \`\`\`solidity function repayDebt( uint256 shares @-> ) public whenNotPaused returns (uint256 assets) { assets = convertToAssets(shares);

...

```
    emit DebtRepaid(msg.sender, currentDebt[msg.sender], assets, shares);
```

... super.\_burn(msg.sender, shares); //@audit func is supposed to return debt in assets paid off but rather it returns shares

@-> return shares; } \`\`\` we can tell from the function declaration that the function is supposed to return \`assets\`. However the last line returns shares which will overwrite previous intent of returning assets.

Another concept is; when you look at \`previewRepayDebt\` below; \`\`\`solidity /// @notice Previews the amount of assets that will be burned for the given /// amount of repaid shares. function previewRepayDebt(uint256 shares) public view returns (uint256) { return convertToAssets(shares); } \`\`\` we will observe that the preview function return assets correctly unlike \`repayDebt\`. The reason why a reference is being made to \`previewRepaydebt()\` is because of the comments below; \`\`\` /// @dev The debtor has to approve the transfer of the shares. To determine /// the asset debt that is going to be repaid, the caller can use /// the \`previewRepayDebt\` function. \`\`\` from the comments we can deduce that the caller is supposed to use the preview function as a point of reference to determine the asset debt that is going to be repaid.

Hence, the outcome of \`previewRepayDebt\` is assets in debt to be repaid which is supposed to similar \`repayDebt\` but per our analysis we can see this is not true
