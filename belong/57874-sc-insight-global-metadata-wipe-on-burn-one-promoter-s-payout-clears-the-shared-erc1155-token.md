# 57874 sc insight global metadata wipe on burn one promoter s payout clears the shared erc1155 token uri for all promoters of the same venue

**Submitted on Oct 29th 2025 at 10:38:04 UTC by @xKeywordx for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57874
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/base/ERC1155Base.sol>
* **Impacts:** Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)

## Description

`promoterToken` is an ERC1155 where the `tokenId` is derived from the venue's `venueId` (venueId = Helper.getVenueId(venue)), and all promoters of the same venue receive balances on that same `tokenId`.

According to the protocol's docs:

* <https://belongnet.github.io/docs/belong-checkin/promoter-earnings-guide>
* <https://belongnet.github.io/docs/belong-checkin/token-value-creation>
* <https://belongnet.github.io/docs/belong-checkin/venue-partner>

a venue can and should be promoted by multiple promoters simultaneously. For a given venue (Event A), all promoters (KOLs) share the same `promoterToken` `tokenId`. Balances differ per promoter, but the `tokenId` is shared.

This is problematic because `ERC1155Base::burn` clears the token URI:

```solidity
function burn(address from, uint256 tokenId, uint256 amount) public onlyRole(BURNER_ROLE) {
    _setTokenUri(tokenId, "");      // <— @audit clears the URI for this tokenId
    _burn(from, tokenId, amount);
}
```

`burn` is called inside `BelongCheckIn::distributePromoterPayments()`:

```solidity
function distributePromoterPayments(PromoterInfo memory promoterInfo) external {
    //..
    //@audit-issue this will clear the tokenURI for that `promoterToken` for all promoters of a venue
    _storage.contracts.promoterToken.burn(promoterInfo.promoter, venueId, promoterInfo.amountInUSD);
    //..
}
```

Therefore, when any single promoter claims and burns part of their balance, the URI for that shared `venueId` is cleared for everyone, including other promoters who still hold balances for that venue.

## Root cause

* Shared `tokenId` across promoters: multiple promoters are minted balances of the same `tokenId`.
* Global state in URI storage: token metadata is stored once per `tokenId` in `mapping(uint256 => string) _tokenUri`.
* URI clearing on burn: `ERC1155Base.burn` calls `_setTokenUri(tokenId, "")` regardless of remaining supply or other holders.

Because ERC1155 URIs are global to an `id`, clearing it inside `burn` mutates what that id represents for all holders, not just the caller.

## Impact

According to the Audit Competition rules (<https://immunefi.com/audit-competition/audit-comp-belong/scope/#top>) this falls under "Critical - Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)".

* Loss of metadata for all remaining holders of `promoterToken` for that `venueId` after any single payout.
* UI/analytics issues: dashboards, wallets, and indexers relying on `uri(tokenId)` lose the venue’s descriptive data (image, name, attributes) for all promoters still holding balances.
* Griefing attack vector: a promoter can trigger a minimal burn via a small claim and wipe the shared metadata for everyone else.
* Metadata inconsistency: new mints will set the URI again, while burns will delete it, toggling on/off.

## Recommended mitigation

Revisit the metadata creation/deletion logic for ERC1155 tokens. Possible approaches (do not add new assumptions beyond the original content):

* Avoid clearing the token URI on burn. Only clear metadata when total supply for `tokenId` reaches zero (if that is desired).
* Store per-holder metadata if different promoters must have separate URIs (i.e., avoid sharing a single global URI when semantics require per-promoter metadata).
* Use a reference-counting or supply-aware mechanism before mutating global token metadata.

## Proof of Concept

<details>

<summary>Test POC: burning promoterToken for one promoter clears shared token URI for all promoters</summary>

Add this test to `belong-check-in.test.ts`:

```javascript
it('POC: burning promoterToken for one promoter clears shared token URI for all promoters', async () => {
  const { belongCheckIn, helper, promoterToken, referral, signer, referralCode, USDC, USDC_whale, ENA_whale } =
    await loadFixture(fixture);

  // Create two more promoters (B, C)
  const wallets = await ethers.getSigners();
  const promoterB = wallets[8].address;
  const promoterC = wallets[9].address;

  const uri = 'uriuri';
  const venue = USDC_whale.address;
  const venueId = await helper.getVenueId(venue);

  // ----- Venue deposit -----
  const venueAmount = await u(100, USDC);
  const venueMessage = ethers.utils.solidityKeccak256(['address', 'bytes32', 'string', 'uint256'], [venue, referralCode, uri, chainId]);
  const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);

  const venueInfo: VenueInfoStruct = {
    rules: { paymentType: 3, bountyType: 3, longPaymentType: 0 } as VenueRulesStruct,
    venue,
    amount: venueAmount,
    referralCode,
    uri,
    signature: venueSignature,
  };

  const paymentToAffiliate = await helper.calculateRate(fees.affiliatePercentage, venueAmount);
  const willBeTaken = paymentToAffiliate.add(convenienceFeeAmount.add(venueAmount));
  await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
  await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

  // ----- Mint promoter credits for three promoters on the same venueId -----
  const customerAmount = await u(5, USDC);
  const visitBounty = await u(1, USDC);
  const spendPct = 1000; // 10%

  // Fund + approve customer for 3 payments
  await USDC.connect(USDC_whale).transfer(ENA_whale.address, customerAmount.mul(3));
  await USDC.connect(ENA_whale).approve(belongCheckIn.address, customerAmount.mul(3));

  // Helper function that is used to do three deposits and change the promoter every time
  async function payToVenueForPromoter(promoterAddr: string) {
    const customerMessage = ethers.utils.solidityKeccak256(
      ['bool', 'uint128', 'uint24', 'address', 'address', 'address', 'uint256', 'uint256'],
      [
        true, // paymentInUSDC
        visitBounty, // visitBountyAmount
        spendPct, // spendBountyPercentage
        ENA_whale.address, // customer
        venue, // venueToPayFor
        promoterAddr, // promoter
        customerAmount, // amount
        chainId, // block.chainid
      ],
    );
    const customerSignature = EthCrypto.sign(signer.privateKey, customerMessage);
    const customerInfo: CustomerInfoStruct = {
      paymentInUSDC: true,
      visitBountyAmount: visitBounty,
      spendBountyPercentage: spendPct,
      customer: ENA_whale.address,
      venueToPayFor: venue,
      promoter: promoterAddr,
      amount: customerAmount,
      signature: customerSignature,
    };
    await belongCheckIn.connect(ENA_whale).payToVenue(customerInfo);
  }

  // A, B, C each get promoter credits for the SAME tokenId (venueId)
  await payToVenueForPromoter(referral.address);
  await payToVenueForPromoter(promoterB);
  await payToVenueForPromoter(promoterC);

  // RESET THE PROMOTERTOKEN TOKENID MANUALLY BECAUSE IT IS WIPED OUT BY ANOTHER BUG IN THE CODE
  const minter = await ethers.getImpersonatedSigner(belongCheckIn.address);
  await ethers.provider.send('hardhat_setBalance', [belongCheckIn.address, ethers.utils.hexlify(ethers.utils.parseEther('1'))]);
  await promoterToken.connect(minter).mint(referral.address, venueId, 1, 'uriuri');

  // Prove that the promoterToken tokenId URI matches the expected URI for the venue
  expect(await promoterToken['uri(uint256)'](venueId)).to.eq('uriuri');
  console.log('tokenURI before = ', await promoterToken['uri(uint256)'](venueId));

  // Check all three promoters have positive balances on the SAME tokenId
  const balA_before = await promoterToken.balanceOf(referral.address, venueId);
  const balB_before = await promoterToken.balanceOf(promoterB, venueId);
  const balC_before = await promoterToken.balanceOf(promoterC, venueId);
  console.log('balA_before =', balA_before);
  console.log('balB_before =', balB_before);
  console.log('balC_before =', balC_before);

  // ----- Promoter A claims (distributePromoterPayments), which overrides tokenURI for promoters B && C -----
  const claimAmount = balA_before;
  const promoterMessage = ethers.utils.solidityKeccak256(
    ['address', 'address', 'uint256', 'uint256'],
    [
      referral.address, // promoter
      venue, // venue
      claimAmount, // amountInUSD
      chainId, // block.chainid
    ],
  );
  const promoterSignature = EthCrypto.sign(signer.privateKey, promoterMessage);
  const promoterInfo: PromoterInfoStruct = {
    paymentInUSDC: true,
    promoter: referral.address,
    venue,
    amountInUSD: claimAmount,
    signature: promoterSignature,
  };

  await belongCheckIn.connect(referral).distributePromoterPayments(promoterInfo);

  // -----> BUG: tokenUri for the SHARED tokenId (venueId) is now wiped for EVERYONE
  expect(await promoterToken['uri(uint256)'](venueId)).to.eq('');
  console.log('tokenURI after = ', await promoterToken['uri(uint256)'](venueId));

  // Promoters B and C STILL hold balances, but their shared metadata is gone.
  const balA_after = await promoterToken.balanceOf(referral.address, venueId);
  const balB_after = await promoterToken.balanceOf(promoterB, venueId);
  const balC_after = await promoterToken.balanceOf(promoterC, venueId);
  console.log('balA_after =', balA_after);
  console.log('balB_after =', balB_after);
  console.log('balC_after =', balC_after);
});
```

Test output:

```
yarn hardhat test test/v2/platform/belong-check-in.test.ts --grep "POC: burning promoterToken for one promoter clears shared token URI for all promoters"
yarn run v1.22.22

  BelongCheckIn ETH Uniswap
    Promoter flow usdc

tokenURI before =  uriuri
balA_before = BigNumber { value: "1500001" }
balB_before = BigNumber { value: "1500000" }
balC_before = BigNumber { value: "1500000" }
tokenURI after =  
balA_after = BigNumber { value: "0" }
balB_after = BigNumber { value: "1500000" }
balC_after = BigNumber { value: "1500000" }
      ✔ POC: burning promoterToken for one promoter clears shared token URI for all promoters (10815ms)
  1 passing (15s)

Done in 20.40s.
```

</details>
