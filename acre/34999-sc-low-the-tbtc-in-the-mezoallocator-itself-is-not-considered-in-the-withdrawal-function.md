# #34999 \[SC-Low] The tBTC in the MezoAllocator itself is not considered in the withdrawal function

**Submitted on Sep 2nd 2024 at 14:22:04 UTC by @dash for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34999
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

\`tBTC\` deposited into \`StBTC\` is routed through the \`MezoAllocator\` to the \`Portal\`. When \`depositors\` want to \`withdraw\` \`tBTC\` from \`StBTC\`, and if \`StBTC\` lacks sufficient \`tBTC\`, it \`withdraws\` the necessary amount from the \`Portal\` via the \`MezoAllocator\`. However, \`tBTC\` held in the \`MezoAllocator\` itself but not yet \`deposited\` to the \`Portal\` is not accounted for, which can lead to an \`arithmetic underflow\` during the \`withdrawal\` process.

## Vulnerability Details

The \`total assets\` of \`StBTC\` consist of three parts: its \`balance\`, the \`total assets\` of the \`MezoAllocator\`, and \`total debts\`. \`\`\` function totalAssets() public view override returns (uint256) { return IERC20(asset()).balanceOf(address(this)) + dispatcher.totalAssets() + totalDebt; } \`\`\` The \`MezoAllocator\`'s \`total assets\` include its \`balance\` and the amount \`deposited\` in the \`Portal\`. \`\`\` function totalAssets() external view returns (uint256) { return depositBalance + tbtc.balanceOf(address(this)); } \`\`\` During a \`withdrawal\`, if \`StBTC\`'s current \`tBTC\` \`balance\` is insufficient, it attempts to \`withdraw\` the required amount from the \`Portal\`. \`\`\` function withdraw( uint256 assets, address receiver, address owner ) public override returns (uint256) { uint256 currentAssetsBalance = IERC20(asset()).balanceOf(address(this)); uint256 assetsWithFees = assets + \_feeOnRaw(assets, exitFeeBasisPoints); if (assetsWithFees > currentAssetsBalance) { dispatcher.withdraw(assetsWithFees - currentAssetsBalance); } return super.withdraw(assets, receiver, owner); } \`\`\` The \`withdraw\` function assumes that the requested amount is less than the amount \`deposited\` in the \`Portal\`. However, \`tBTC\` that hasn’t been \`deposited\` into the \`Portal\` yet—such as donated tokens or rewards—is not considered. As a result, the requested withdrawal amount could exceed the \`Portal\`'s \`deposits\` but still be less than the \`total assets\` of the \`MezoAllocator\`. \`\`\` function withdraw(uint256 amount) external { if (msg.sender != address(stbtc)) revert CallerNotStbtc();

if (amount < depositBalance) { mezoPortal.withdrawPartially( address(tbtc), depositId, uint96(amount) ); } else { mezoPortal.withdraw(address(tbtc), depositId); } depositBalance -= uint96(amount); // @audit: underflow when amount > depositBalance tbtc.safeTransfer(address(stbtc), amount); } \`\`\` This discrepancy can cause a \`withdrawal\` failure due to \`underflow\`, effectively locking users' funds until the next \`allocation\` by \`MezoAllocator\` \`maintainers\`.

## Impact Details

The impact is clear, as it can lead to a denial of service (DoS) and lock users' funds.

## References

<https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L474-L479> <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/MezoAllocator.sol#L297> <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L442> <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/MezoAllocator.sol#L241>

## Recommendation

\`\`\` function withdraw(uint256 amount) external { if (msg.sender != address(stbtc)) revert CallerNotStbtc();

if (amount < depositBalance) { mezoPortal.withdrawPartially( address(tbtc), depositId, uint96(amount) ); } else { mezoPortal.withdraw(address(tbtc), depositId); } - depositBalance -= uint96(amount);

* depositBalance = depositBalance > uint96(amount) ? depositBalance - uint96(amount) : 0; tbtc.safeTransfer(address(stbtc), amount); } \`\`\`

## Proof of Concept

## Proof of Concept

Add below test to the \`stBTC.test.ts\`: \`\`\` describe("withdraw from portal", () => { it("should withdraw from the dispatcher", async () => { const assets = to1e18(100) await tbtc.mint(depositor1.address, assets) await tbtc.connect(depositor1).approve(await stbtc.getAddress(), assets) await stbtc.connect(depositor1).deposit(assets, depositor1.address)

```
await mezoAllocator.connect(maintainer).allocate()

/**
 * some tbtc are not deposited to the Portal
 */
await tbtc.mint(await mezoAllocator.getAddress(), to1e18(1))

const shares &#x3D; await stbtc.balanceOf(depositor1.address) 
/**
 *  reverted with below error
 *  Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
 */
await stbtc.connect(depositor1).redeem(shares, depositor1.address, depositor1.address)
```

}) }) \`\`\`
