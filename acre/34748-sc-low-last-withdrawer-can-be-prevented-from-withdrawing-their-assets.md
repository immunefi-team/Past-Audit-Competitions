# #34748 \[SC-Low] Last withdrawer can be prevented from withdrawing their assets

**Submitted on Aug 23rd 2024 at 12:50:40 UTC by @trachev for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34748
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

By donating dust to the \`MezoAllocator\` contract a malicious actor can prevent a user from redeeming their shares.

## Vulnerability Details

The number of assets that a share is worth is calculated using the \`totalAssets\` function of stBTC. \`totalAsssets\` returns the tBTC balance of stBTC, the \`totalDebt\` and \`dispatcher.totalAssets()\`. \`dispatcher.totalAssets()\` includes tBTC deposited into \`MezoPortal\` and also tBTC sent directly to \`MezoAllocator\` that are yet to be allocated to the portal. The issue is that those tBTC cannot be withdrawn from \`MezoAllocator\` until they have been allocated to the portal. As a result, they can make a user's shares be worth more than the total assets that can actually be withdrawn. This attack can be repeated multiple times as it requires an extremely small amount of dust to make the function revert, in the PoC only 100 wei is used.

## Impact Details

If there is only one depositor in stBTC, they can be prevented from withdrawing. The user is griefed and the attack can be performed repeatedly.

## References

<https://github.com/thesis/acre/blob/dc156f5a7f02142c1f80627267d14a26e5c99b30/solidity/contracts/stBTC.sol#L474-L479>

## Proof of Concept

## Proof of Concept

Place this code in the \`MezoAllocator.test.ts\` file, inside of the "MezoAllocator" describe block.

describe("POC\_last\_withdraw\_fails", () => { beforeAfterSnapshotWrapper()

```
it(&quot;Should revert when last user withdraws&quot;, async () &#x3D;&gt; {
  let assetsToDeposit &#x3D; to1e18(1n)
  await tbtc.mint(depositor.address, assetsToDeposit)

  await tbtc
    .connect(depositor)
    .approve(await stbtc.getAddress(), assetsToDeposit)
    

  //depositor deposits in stbts
  await stbtc
    .connect(depositor)
    .deposit(assetsToDeposit, depositor.address)

  //allocate to mezo portal
  await mezoAllocator.connect(maintainer).allocate()

  //mint dust (only 100 wei, with 2 wei reverts as well), 
  //that is yet to be allocated, so that the total assets is inflated
  await tbtc.mint(mezoAllocator, 100)

  //reverts with a panic code 0x11 due to an arithmetic oveflow
  await stbtc.connect(depositor).redeem(await stbtc.balanceOf(depositor.address), depositor.address, depositor.address)
})
```

})
