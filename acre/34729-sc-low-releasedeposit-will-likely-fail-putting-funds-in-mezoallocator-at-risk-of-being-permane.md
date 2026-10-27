# #34729 \[SC-Low] \`releaseDeposit\` will likely fail, putting funds in MezoAllocator at risk of being p

**Submitted on Aug 22nd 2024 at 16:58:37 UTC by @trachev for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34729
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

As the developer comments state: \`releaseDeposit\` is a special function that can be used to migrate funds during allocator upgrade or in case of emergencies. The issue is that due to an issue in its logic it is possible for the function to revert, causing assets in \`MezoAllocator\` to become permanently stuck.

## Vulnerability Details

\`releaseDeposit\` makes a call to \`mezoPortal.withdraw\` in order to withdraw any tBTC left in the contract. The issue is that if all of the \`MezoPortal\` tBTC had already been withdrawn the function will revert. This happens because \`mezoPortal.withdraw\` does not allow empty withdrawals and withdrawing a \`depositId\` that has already been fully withdrawn will revert (if a deposit is fully withdrawn it is deleted from the \`deposits\` mapping): \`\`\`solidity uint96 depositedAmount = selectedDeposit.balance; uint96 fee = selectedDeposit.feeOwed;

```
    if (depositedAmount &#x3D;&#x3D; 0) {
        revert DepositNotFound();
    }
```

\`\`\`

As a result, \`releaseDeposit\` is going to revert, preventing the protocol from migrating any tBTC left in the protocol. It is expected that there will be other tBTC tokens, except the ones that are deposited from stBTC to \`MezoPortal\`, from either donations or rewards sent to \`MezoAllocator\`. This has been stated in ISSUE#3 in the following audit report provided by Thesis: <https://github.com/Thesis-Defense/Security-Audit-Reports/blob/main/PDFs/240517\\_Thesis\\_Defense-Acre\\_Smart\\_Contracts\\_Security\\_Audit\\_Report.pdf?utm\\_source=immunefi>

The issue can be easily prevented by not calling \`mezoPortal.withdraw\` in \`releaseDeposit\` if the \`amount\` is equal to 0. If that is the case omit the call to \`mezoPortal.withdraw\` but still transfer the contract's tBTC balance to stBTC.

## Impact Details

Permanent loss of funds for the protocol.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Place this coded PoC in the \`MezoAllocator.test.ts\` integration test file. It is important to note that the \`MezoPortal\` contract needs to be set to an actual \`MezoPortal\` implementation and not a mock implementation, to simulate how the contracts will actually act in production:

\`\`\`solidity describe("PoC\_releaseDeposit\_fails", () => { it("fails becuase the depositId has already been withdrawn", async () => { let amountToDeposit: bigint = to1e18(1) await tbtc.mint(depositor, amountToDeposit)

```
  //deposit into stbtc from depositor
  await tbtc.connect(depositor).approve(await stbtc.getAddress(), amountToDeposit)
  await stbtc.connect(depositor).deposit(amountToDeposit, depositor.address)

  //allocate to mezo portal
  await mezoAllocator.connect(maintainer).allocate()

  //redeem everything, withdrawing fully the depositId
  await stbtc.connect(depositor).redeem(await stbtc.balanceOf(depositor.address), depositor.address, depositor.address)

  //simulate donations or rewards transferred to the mezo allocator
  await tbtc.mint(mezoAllocator, to1e18(1))

  //reverts because the depositId has been withdrawn from MezoPortal and the funds in MezoAllocator are stuck
  await mezoAllocator.connect(governance).releaseDeposit()
})
```

}) \`\`\`
