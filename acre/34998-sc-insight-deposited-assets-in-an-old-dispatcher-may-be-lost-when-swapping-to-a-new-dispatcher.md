# #34998 \[SC-Insight] Deposited assets in an old dispatcher may be lost when swapping to a new dispatc

**Submitted on Sep 2nd 2024 at 13:43:32 UTC by @dash for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34998
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

In \`StBTC\`, the \`owner\` can change the \`dispatcher\`. When this happens, the \`allowance\` for the old \`dispatcher\` is simply reduced to \`zero\`. However, any assets still deposited to the old \`dispatcher\` are not \`withdrawn\` or transferred to the new \`dispatcher\`, which could result in a loss for the \`depositors\`.

## Vulnerability Details

\`Maintainers\` can periodically allocate \`tBTC\` from \`StBTC\` to the \`Mezo Portal\`, meaning that most of the funds will be deposited there. \`\`\` function allocate() external onlyMaintainer { uint256 addedAmount = tbtc.balanceOf(address(stbtc)); tbtc.safeTransferFrom(address(stbtc), address(this), addedAmount);

depositBalance = uint96(tbtc.balanceOf(address(this))); tbtc.forceApprove(address(mezoPortal), depositBalance); mezoPortal.deposit(address(tbtc), depositBalance, 0); // @audit, here } \`\`\` \`StBTC\` can \`withdraw\` \`tBTC\` from the \`Portal\` when there isn’t enough \`tBTC\` available to process \`withdrawals\`. \`\`\` function withdraw( uint256 assets, address receiver, address owner ) public override returns (uint256) { uint256 currentAssetsBalance = IERC20(asset()).balanceOf(address(this)); uint256 assetsWithFees = assets + \_feeOnRaw(assets, exitFeeBasisPoints);

if (assetsWithFees > currentAssetsBalance) { dispatcher.withdraw(assetsWithFees - currentAssetsBalance); // @audit, here } return super.withdraw(assets, receiver, owner); } \`\`\`

Additionally, the \`owner\` of \`MezoAllocator\` has the ability to \`withdraw\` all \`tBTC\` from the \`Portal\` back to \`StBTC\`. \`\`\` function releaseDeposit() external onlyOwner { uint96 amount = mezoPortal .getDeposit(address(this), address(tbtc), depositId) .balance;

depositBalance = 0; mezoPortal.withdraw(address(tbtc), depositId); tbtc.safeTransfer(address(stbtc), tbtc.balanceOf(address(this))); // @audit, here } \`\`\` When swapping \`dispatchers\`, we simply reduce the \`allowance\` of the old \`dispatcher\` to zero, but we don’t \`withdraw\` the funds from it. \`\`\` function updateDispatcher(IDispatcher newDispatcher) external onlyOwner { address oldDispatcher = address(dispatcher); dispatcher = newDispatcher;

if (oldDispatcher != address(0)) { IERC20(asset()).forceApprove(oldDispatcher, 0); // @audit, here }

IERC20(asset()).forceApprove(address(dispatcher), type(uint256).max); } \`\`\` There’s no \`100%\` guarantee that all funds will be \`withdrawn\` from the old \`dispatcher\` to \`StBTC\`, even if the \`owner\` of the old \`dispatcher\` acts honestly before the \`swap\`. This is because \`maintainers\` can allocate funds from \`StBTC\` to a \`dispatcher\`. And the implementation logic of the \`dispatcher\` may vary.

## Impact Details

I believe the impact could be significant if this occurs, but I’ve marked it as medium since it primarily involves privileged users

## References

<https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/MezoAllocator.sol#L206> <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L442> <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/MezoAllocator.sol#L256> <https://github.com/thesis/acre/blob/c3790ef2d4a5a11ae1cadcdaf72ce538b8d67dd3/solidity/contracts/stBTC.sol#L215>

## Recommendation

Withdraw \`tBTC\` from the old \`dispatcher\` if any remains. \`\`\` function updateDispatcher(IDispatcher newDispatcher) external onlyOwner { address oldDispatcher = address(dispatcher); dispatcher = newDispatcher;

if (oldDispatcher != address(0)) { IERC20(asset()).forceApprove(oldDispatcher, 0); + uint256 remainingAssets = dispatcher.totalAssets(); + if (remainingAssets) { + dispatcher.withdraw(remainingAssets); + } }

IERC20(asset()).forceApprove(address(dispatcher), type(uint256).max); } \`\`\`

## Proof of Concept

## Proof of Concept

Add below test to the \`stBTC.test.ts\`: \`\`\` describe("updateDispatcher and total assets check", () => { it("should withdraw assets from the old dispatcher", async () => { const assets = to1e18(100) await tbtc.mint(depositor1.address, assets) await tbtc.connect(depositor1).approve(await stbtc.getAddress(), assets) await stbtc.connect(depositor1).deposit(assets, depositor1.address)

```
await mezoAllocator.connect(maintainer).allocate()

const totalAssets &#x3D; await mezoAllocator.totalAssets();
/**
 * totalAssets &gt; 0
 */
expect(totalAssets).to.be.greaterThan(0)
/**
 * stbtc.totalAssets &#x3D; dispatcher.totalAssets
 */
expect(await stbtc.totalAssets()).to.be.equal(totalAssets)

const newDispatcher &#x3D; await ethers.Wallet.createRandom().getAddress()

await stbtc.connect(governance).updateDispatcher(newDispatcher)

/**
 * tbtc balance of StBTC &#x3D; 0
 * tbtc balance of newDispatcher &#x3D; 0
 * i.e. totalAssets &#x3D; 0
 */
expect(await tbtc.balanceOf(await stbtc.getAddress())).to.be.equal(0)
```

}) }) \`\`\`
