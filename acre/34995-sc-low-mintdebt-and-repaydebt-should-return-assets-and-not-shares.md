# #34995 \[SC-Low] \`mintDebt()\` and \`repayDebt()\` should return \`assets\` and not \`shares\`

**Submitted on Sep 2nd 2024 at 13:09:27 UTC by @sammytm for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34995
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

\`mintDebt()\` and \`repayDebt()\` are used to mint debt and repay the debt by a user.

## Vulnerability Details

\`mintDebt()\` and \`repayDebt()\` currently return \`shares\` inputted by the users and not the corresponding \`assets\` \`\`\`solidity function repayDebt( uint256 shares ) public whenNotPaused returns (uint256 assets) { assets = convertToAssets(shares);

```
    // Check the current debt of the debtor.
    if (currentDebt[msg.sender] &lt; assets) {
        revert ExcessiveDebtRepayment(
            msg.sender,
            currentDebt[msg.sender],
            assets
        );
    }

    // Decrease the debt of the debtor.
    currentDebt[msg.sender] -&#x3D; assets;

    emit DebtRepaid(msg.sender, currentDebt[msg.sender], assets, shares);

    // Decrease the total debt.
    totalDebt -&#x3D; assets;

    // Burn the shares from the debtor.
    super._burn(msg.sender, shares);

    return shares;
}
```

\`\`\` \`\`\`solidity function mintDebt( uint256 shares, address receiver ) public whenNotPaused returns (uint256 assets) { assets = convertToAssets(shares);

```
    // Increase the debt of the debtor.
    currentDebt[msg.sender] +&#x3D; assets;

    // Check the maximum debt allowance of the debtor.
    if (currentDebt[msg.sender] &gt; allowedDebt[msg.sender]) {
        revert InsufficientDebtAllowance(
            msg.sender,
            allowedDebt[msg.sender],
            currentDebt[msg.sender]
        );
    }

    emit DebtMinted(msg.sender, currentDebt[msg.sender], assets, shares);

    // Increase the total debt.
    totalDebt +&#x3D; assets;

    // Mint the shares to the receiver.
    super._mint(receiver, shares);

    return shares;
}
```

\`\`\`

## Impact Details

The ERC4626 standard states that minting shares must return assets as return value. But this doesn't follow the standard and may cause integration issues with some smart contracts. This can also cause confusion to users, particularly when the shares : assets ratio isn't 1:1

## Proof of Concept

## Proof of Concept

\`\`\` describe.only("POC", () => { beforeAfterSnapshotWrapper()

```
it(&quot;Vault infaltion&quot;, async function () {
  let minimumDepositAmount &#x3D; await stbtc.minimumDepositAmount()
  //await tbtc.connect(depositor1).transfer(stbtc,100);
  await stbtc.connect(governance).updateDebtAllowance(depositor1,10n ** 7n);

 console.log((await stbtc.connect(depositor1).mintDebt(1, depositor1)) + &quot; &quot;+ (await stbtc.balanceOf(depositor1)));
 
})
```

}) \`\`\` Copy in \`stBTC.test.ts\` and run \`pnpm test\`
