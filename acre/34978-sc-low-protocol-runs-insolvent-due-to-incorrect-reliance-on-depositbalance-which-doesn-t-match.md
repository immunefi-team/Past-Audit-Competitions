# #34978 \[SC-Low] protocol runs insolvent due to incorrect reliance on depositbalance which doesn t ma

## #34978 \[SC-Low] Protocol runs insolvent due to incorrect reliance on depositBalance which doesn't match holder balances

**Submitted on Sep 2nd 2024 at 06:28:51 UTC by @styphoiz for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34978
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Protocol insolvency

### Description

### Brief/Intro

Currently, if all/sufficient holders of Acre Staked Bitcoin (stBTC) attempt to withdraw their tokens, the protocol may become insolvent due to an issue with the depositBalance function.

### Vulnerability Details

This issue is logged separately from <https://bugs.immunefi.com/dashboard/submission/34672>, the flow is similar however this is a different function been called on the contract. The withdraw function in contract 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3 is designed to work with the balance of tBTC held within the contract. When this balance is insufficient to cover redemptions, the contract attempts to withdraw additional funds from 0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0 using the depositBalance function. However, the issue arises when depositBalance returns a value lower than expected, leading to a shortfall of tBTC in the contract. As a result, even though there may be sufficient overall funds in the system, the reliance on depositBalance causes the contract to fail in meeting redemption requests, rendering the protocol insolvent.

### Impact Details

Acre Staked Bitcoin (stBTC) holders attempting to withdraw their tokens after the value of depositBalance plus the tokens in 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3 falls below their redemption amount will be unable to redeem their tBTC tokens, potentially causing significant losses and undermining trust in the protocol.

### References

Acre Staked Bitcoin (stBTC) - <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3> Mezo Allocator - <https://sepolia.etherscan.io/address/0xd5ebdd6ff384a465d56562d3a489c8cce1b92dd0> Mezo Portal - <https://sepolia.etherscan.io/address/0x6978e3e11b8bc34ea836c1706fc742ac4cb6b0db>

### Proof of Concept

### Proof of Concept

## Steps are provided below on what I had done to build this code.

I had first went to <https://sepolia.etherscan.io/token/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3#balances> to get a list of holders of Acre Staked Bitcoin (stBTC)

\`\`\` const ownerAddresses = \[ "0xCE06a2D105559C633451971ab1f843D667597265", "0x512f4f3a02862b0A7e7F1D796B885ce3D4EaB5cf", "0xB66ab5A4596250Ce20ff62262935CAB5E8A17695", "0x54567d825cE85a4E9C4314984CEe4D9253458B1B", "0x5476A06f08CD1F9669Ae6643C5eF9cc4F1848970", "0x16610D47659373cEE43F6983D46b02256c03F7C1", "0xD0B9584c57B6fFDeD640130232735388737dE251", "0x88744F5da4308317B459EFB205028AED77B1ae2C", "0x0763DfC2fb8b060e0629928B5D77466D1C4Ca379", // "0x21F071bd9Ed020fb6E3e9A661Ca547E94f713467", "0x6F1a421573082BE1BEAe22551259D4D793EfD2cE", "0x9B55cDe4d96aAa9CCCc4fC9Fd12Ab43292750294", "0x18361d831C81384fBd8c5BaCa1727cae64212B9d", "0x8b63f664eC49bA2AbCb24ACCC76E3Ee1522ddB9e", "0x8d951Dfe12e12ea4549e18382D7e4c9188046851", "0x247c356466D139Df16231E576eF52B1168528B6F", "0x18c3D37A85b4e44A5619d62Ee4900Bcc18b3bd5a", "0x719743739BD4E5154248705BF9bF67ac2D85b52F", "0x0483cD12aC9758e530dc184a1b542439BA6cDB8f", "0x82d930246C2e0F2a383d893E1F1DeB45CE602d1B", "0xA4761081d9Cb672d911d7df25E5a30D7925608CE", // "0xd2C6168Fd106908Df71Ab639f8b7e2F971Ab8205", "0x857173e7c7d76e051e80d30FCc3EA6A9C2b53756", "0x3df087df73576CA02f5f2D10ce95b00355482a51", "0x6e80164ea60673D64d5d6228beb684a1274Bb017", ]; \`\`\`

After getting this list, you can currently see that the value of tBTC at the time of writing this held at 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3 (referred to as stBTC) is 1387265314680000000 (1.38726531468), the total Supply of stBTC is 4452802992110739292 (4.452802992110739292), this means that the stBTC's contract's tBTC tokens should match the totalSupply which if there is not enough tBTC in the stBTC contract to align to the user's deposits.

### When a withdraw() occurs

In the event that stBTC doesn't contain enough tBTC tokens, stBTC will go to dispatcher to withdraw funds to top itself up to allow the withdraw. We see that here in the withdraw function in stBTC after checking that it doesn't have sufficient tBTC tokens, it calls dispatcher.withdraw to top itself up.

The dispatcher at this time is 0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0 (referred to as MezoAllocator).

### When MezoAllocator does a withdrawal

We see within the code of MezoAllocator, we can see that MezoAllocator does calls to 0x6978E3e11b8Bc34ea836C1706fC742aC4Cb6b0Db (referred to as MezoPortal) We also see that MezoAllocator contains no tBTC.

\`\`\` function withdraw(uint256 amount) external { if (msg.sender != address(stbtc)) revert CallerNotStbtc();

```
    emit DepositWithdrawn(depositId, amount);

    if (amount &lt; depositBalance) {
        mezoPortal.withdrawPartially(
            address(tbtc),
            depositId,
            uint96(amount)
        );
    } else {
        mezoPortal.withdraw(address(tbtc), depositId);
    }

    // slither-disable-next-line reentrancy-no-eth
    depositBalance -&#x3D; uint96(amount);
    tbtc.safeTransfer(address(stbtc), amount);
}
```

\`\`\` Within the function we see that there are 2 types of withdraws been used to call MezoPortal. This process does a comparison of the amount against the depositBalance, at this time is currently 752030076580000000 (0.75203007658), we also see that upon a success withdrawal results in the depositBalance been lowered by the amount. This means that MezoAllocator's depositBalance value and the actual tBTC in the stBTC contract should be greater than the total Supply of tBTC to allow users to call redeem and withdraw successfully. Here we see that this is not the case

Total Supply stBTC: 4.452802992110739292 Balance of tBTC in stBTC : 1.38726531468 depositBalance value in MezoAllocator : 0.75203007658 With sufficient redeems to decrease the balance of tBTC in stBTC and then start decrease the depositBalance value in MezoAllocator, when we do a comparison here of these amounts, we see that (Total Supply stBTC) - (Balance of tBTC in stBTC) - (depositBalance value in MezoAllocator) now gives us an amount that is greater than depositBalance pushing us into the below code. \`\`\` else { mezoPortal.withdraw(address(tbtc), depositId); } \`\`\`

### MezoPortal withdraw explanation

We see that the function withdraw uses 2 parameters address(tbtc), depositId, let's look through the code here. \`\`\` function withdraw(address token, uint256 depositId) external { TokenAbility ability = tokenAbility\[token];

```
    if (ability &#x3D;&#x3D; TokenAbility.None) {
        revert TokenNotSupported(token);
    }

    DepositInfo storage selectedDeposit &#x3D; deposits[msg.sender][token][
        depositId
    ];

    if (
        ability &#x3D;&#x3D; TokenAbility.DepositAndLock &amp;&amp;
        // solhint-disable-next-line not-rely-on-time
        block.timestamp &lt; selectedDeposit.unlockAt
    ) {
        revert DepositLocked(selectedDeposit.unlockAt);
    }

    if (selectedDeposit.receiptMinted &gt; 0) {
        revert ReceiptNotRepaid(selectedDeposit.receiptMinted);
    }

    uint96 depositedAmount &#x3D; selectedDeposit.balance;

    if (depositedAmount &#x3D;&#x3D; 0) {
        revert DepositNotFound();
    }

    uint96 fee &#x3D; 0;

    if (selectedDeposit.feeOwed &gt; 0) {
        FeeInfo storage tokenFeeInfo &#x3D; feeInfo[token];

        fee &#x3D; uint96(
            _adjustTokenDecimals(
                tokenFeeInfo.receiptToken,
                token,
                selectedDeposit.feeOwed
            )
        );
    }

    delete deposits[msg.sender][token][depositId];

    uint256 withdrawable &#x3D; depositedAmount - fee;
    emit Withdrawn(msg.sender, token, depositId, withdrawable);
    emit FeeCollected(msg.sender, token, depositId, fee);
    feeInfo[token].feeCollected +&#x3D; fee;
    IERC20(token).safeTransfer(msg.sender, withdrawable);
}
```

\`\`\` In this code, we see that the value used by the contract sits under deposits\[msg.sender]\[token]\[depositId]; Adding in our values and interacting with the contract on Sepolia Etherscan, we see that the image (Acre\_Deposit\_509.png) has a balance of 752030076580000000 (0.75203007658) During the process of withdrawals, we had at this point run the stBTC contract and the depositBalance down, this means that the MezoPortal amount transferred will now be much less that what the user requested on the redeem. Back within the withdraw function on MezoAllocator, MezoAllocator has received whatever MezoPortal has provided which in the POC is now less than the depositBalance value. However mezoPortal decides to send the amount value provided by the user from the number of shares provided by the stBTC token holder on in the redeem function on the stBTC contract.

function withdraw(uint256 amount) external { ... depositBalance -= uint96(amount); ... } This will not work and breaks the contract as MezoPortal doesn't have sufficient tokens to honor the request and then fails. A maintainer can then intervene at this point by depositing tBTC into stBTC to continue the withdrawals, the protocol however remains at reputational risk of been seen as insolvent.

Full code for the POC as ran under npx hardhat test to simulate the details above. \`\`\` const { expect } = require("chai"); const { ethers } = require("hardhat");

describe("Acre Bug Bounty", function () { it("Redeem runs out of funds?", async function () { this.timeout(300000);

```
// Set up contract addresses
const stBTCAddress &#x3D; &quot;0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3&quot;;
const mezoAddress &#x3D; &quot;0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0&quot;;
const mezoPortal &#x3D; &quot;0x6978E3e11b8Bc34ea836C1706fC742aC4Cb6b0Db&quot;;
const tBTCAddress &#x3D; &quot;0x517f2982701695D4E52f1ECFBEf3ba31Df470161&quot;;

// List of owner addresses
const ownerAddresses &#x3D; [
  &quot;0xCE06a2D105559C633451971ab1f843D667597265&quot;,
  &quot;0x512f4f3a02862b0A7e7F1D796B885ce3D4EaB5cf&quot;,
  &quot;0xB66ab5A4596250Ce20ff62262935CAB5E8A17695&quot;,
  &quot;0x54567d825cE85a4E9C4314984CEe4D9253458B1B&quot;,
  &quot;0x5476A06f08CD1F9669Ae6643C5eF9cc4F1848970&quot;,
  &quot;0x16610D47659373cEE43F6983D46b02256c03F7C1&quot;,
  &quot;0xD0B9584c57B6fFDeD640130232735388737dE251&quot;,
  &quot;0x88744F5da4308317B459EFB205028AED77B1ae2C&quot;,
  &quot;0x0763DfC2fb8b060e0629928B5D77466D1C4Ca379&quot;,
  // &quot;0x21F071bd9Ed020fb6E3e9A661Ca547E94f713467&quot;,
  &quot;0x6F1a421573082BE1BEAe22551259D4D793EfD2cE&quot;,
  &quot;0x9B55cDe4d96aAa9CCCc4fC9Fd12Ab43292750294&quot;,
  &quot;0x18361d831C81384fBd8c5BaCa1727cae64212B9d&quot;,
  &quot;0x8b63f664eC49bA2AbCb24ACCC76E3Ee1522ddB9e&quot;,
  &quot;0x8d951Dfe12e12ea4549e18382D7e4c9188046851&quot;,
  &quot;0x247c356466D139Df16231E576eF52B1168528B6F&quot;,
  &quot;0x18c3D37A85b4e44A5619d62Ee4900Bcc18b3bd5a&quot;,
  &quot;0x719743739BD4E5154248705BF9bF67ac2D85b52F&quot;,
  &quot;0x0483cD12aC9758e530dc184a1b542439BA6cDB8f&quot;,
  &quot;0x82d930246C2e0F2a383d893E1F1DeB45CE602d1B&quot;,
  &quot;0xA4761081d9Cb672d911d7df25E5a30D7925608CE&quot;,
  // &quot;0xd2C6168Fd106908Df71Ab639f8b7e2F971Ab8205&quot;,
  &quot;0x857173e7c7d76e051e80d30FCc3EA6A9C2b53756&quot;,
  &quot;0x3df087df73576CA02f5f2D10ce95b00355482a51&quot;,
  &quot;0x6e80164ea60673D64d5d6228beb684a1274Bb017&quot;,
];

// Connect to contracts
const stBTC &#x3D; await ethers.getContractAt(&quot;IstBTC&quot;, stBTCAddress);
const tBTC &#x3D; await ethers.getContractAt(&quot;IERC20&quot;, tBTCAddress);
const mezoAllocator &#x3D; await ethers.getContractAt(&quot;IMezo&quot;, mezoAddress);

for (const ownerAddress of ownerAddresses) {
  const balanceOfstBTCContract &#x3D; await tBTC.balanceOf(stBTCAddress);
  console.log(
    &#x60;tBTC balance of stBTC contract ${stBTCAddress}:&#x60;,
    ethers.utils.formatEther(balanceOfstBTCContract)
  );
  const balanceOfmezoContract &#x3D; await tBTC.balanceOf(mezoAddress);
  console.log(
    &#x60;tBTC balance of mezo contract ${mezoAddress}:&#x60;,
    ethers.utils.formatEther(balanceOfmezoContract)
  );
  const balanceOfmezoPortal &#x3D; await tBTC.balanceOf(mezoPortal);
  console.log(
    &#x60;tBTC balance of mezo portal ${mezoPortal}:&#x60;,
    ethers.utils.formatEther(balanceOfmezoPortal)
  );
  const mezoDepositBalance &#x3D; await mezoAllocator.depositBalance();
  console.log(
    &#x60;Deposit balance of mezo portal ${mezoPortal}:&#x60;,
    ethers.utils.formatEther(mezoDepositBalance)
  );
  // Impersonate the owner account
  await hre.network.provider.request({
    method: &quot;hardhat_impersonateAccount&quot;,
    params: [ownerAddress],
  });
  const owner &#x3D; await ethers.getSigner(ownerAddress);

  // Set balance for gas fees
  await hre.network.provider.send(&quot;hardhat_setBalance&quot;, [
    ownerAddress,
    &quot;0x2D6EA32DF2804590&quot;, // set a large enough balance for gas
  ]);

  const balancestBTCBefore &#x3D; await stBTC.balanceOf(ownerAddress);
  console.log(
    &#x60;stBTC balance before for ${ownerAddress}:&#x60;,
    ethers.utils.formatEther(balancestBTCBefore)
  );

  const balance &#x3D; await stBTC.balanceOf(ownerAddress);

  // await stBTC.connect(owner).redeem(balance, ownerAddress, ownerAddress);
  const balanceMinusOnePercent &#x3D; balance.mul(99).div(100);

  await stBTC
    .connect(owner)
    .withdraw(balanceMinusOnePercent, ownerAddress, ownerAddress);
  const balancestBTCAfter &#x3D; await stBTC.balanceOf(ownerAddress);
  console.log(
    &#x60;stBTC balance after for ${ownerAddress}:&#x60;,
    ethers.utils.formatEther(balancestBTCAfter)
  );

  // Stop impersonating the account
  await hre.network.provider.request({
    method: &quot;hardhat_stopImpersonatingAccount&quot;,
    params: [ownerAddress],
  });
}
```

}); }); \`\`\`
