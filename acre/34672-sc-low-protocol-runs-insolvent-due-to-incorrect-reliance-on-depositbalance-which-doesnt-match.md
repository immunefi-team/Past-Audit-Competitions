# #34672 \[SC-Low] Protocol runs insolvent due to incorrect reliance on depositBalance which doesn't ma

**Submitted on Aug 20th 2024 at 09:12:58 UTC by @styphoiz for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34672
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Currently, if all/sufficient holders of Acre Staked Bitcoin (stBTC) attempt to redeem their tokens, the protocol may become insolvent due to an issue with the depositBalance function.

## Vulnerability Details

The redeem function in contract 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3 is designed to work with the balance of tBTC held within the contract. When this balance is insufficient to cover redemptions, the contract attempts to withdraw additional funds from 0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0 using the depositBalance function. However, the issue arises when depositBalance returns a value lower than expected, leading to a shortfall of tBTC in the contract. As a result, even though there may be sufficient overall funds in the system, the reliance on depositBalance causes the contract to fail in meeting redemption requests, rendering the protocol insolvent.

## Impact Details

Acre Staked Bitcoin (stBTC) holders attempting to redeem their tokens after the value of depositBalance plus the tokens in 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3 falls below their redemption amount will be unable to redeem their tBTC tokens, potentially causing significant losses and undermining trust in the protocol.

## References

Acre Staked Bitcoin (stBTC) - <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3\\>
Mezo Allocator - <https://sepolia.etherscan.io/address/0xd5ebdd6ff384a465d56562d3a489c8cce1b92dd0\\>
Mezo Portal - <https://sepolia.etherscan.io/address/0x6978e3e11b8bc34ea836c1706fc742ac4cb6b0db>

## Proof of Concept

## Proof of Concept

Below test initiates withdrawals on all holders \`\`\` const { expect } = require("chai"); const { ethers } = require("hardhat");

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
  &quot;0x21F071bd9Ed020fb6E3e9A661Ca547E94f713467&quot;,
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
  &quot;0xd2C6168Fd106908Df71Ab639f8b7e2F971Ab8205&quot;,
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

  const balancetBTCBefore &#x3D; await tBTC.balanceOf(ownerAddress);
  console.log(
    &#x60;tBTC balance before for ${ownerAddress}:&#x60;,
    ethers.utils.formatEther(balancetBTCBefore)
  );

  const balance &#x3D; await stBTC.balanceOf(ownerAddress);

  await stBTC.connect(owner).redeem(balance, ownerAddress, ownerAddress);
  const balancetBTCAfter &#x3D; await tBTC.balanceOf(ownerAddress);
  console.log(
    &#x60;tBTC balance after for ${ownerAddress}:&#x60;,
    ethers.utils.formatEther(balancetBTCAfter)
  );

  // Stop impersonating the account
  await hre.network.provider.request({
    method: &quot;hardhat_stopImpersonatingAccount&quot;,
    params: [ownerAddress],
  });
}
```

}); }); \`\`\`
