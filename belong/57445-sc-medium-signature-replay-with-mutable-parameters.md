# 57445 sc medium signature replay with mutable parameters

**Submitted on Oct 26th 2025 at 09:53:46 UTC by @pirex for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57445
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

The NFT factory's signature verification mechanism only validates immutable metadata (name, symbol, contract\_uri, royalty\_fraction) while blindly trusting caller-supplied mutable parameters like payment\_token, mint\_price, max\_total\_supply, transferrable flag, and referral\_code. An attacker holding a legitimate platform signature can replay it to deploy the same collection with malicious economics—redirecting mint revenues to an attacker-controlled ERC-20, manipulating prices to bypass fee expectations, or diverting referral rewards. This enables theft of platform fees and user funds.

### Vulnerability Details

The signature validation flow in the NFT factory has a critical gap between what gets signed off-chain and what gets enforced on-chain.

What's signed (src/snip12/produce\_hash.cairo:7-33):

```cairo
ProduceHash {
    name_hash: hash(name),
    symbol_hash: hash(symbol),
    contract_uri: hash(uri),
    royalty_fraction: u16
}
```

What's actually deployed (src/nftfactory/nftfactory.cairo:275-307): The produce function accepts an InstanceInfo struct containing both signed and unsigned fields. After verifying the signature covers only the four fields above, the factory proceeds to deploy the NFT contract using all parameters from the caller-supplied struct, including:

* `payment_token`: arbitrary ERC-20 address
* `mint_price` and `whitelisted_mint_price`: arbitrary prices
* `max_total_supply`: supply cap
* `transferrable`: transfer restrictions
* `referral_code`: revenue routing

These mutable parameters are written directly into both the NFT contract storage and the factory's nftInfo mapping without any validation against the signature. The platform's off-chain signature approval process presumably checks these fields before signing, but that check becomes meaningless since the signature doesn't bind them.

A legitimate creator who receives a valid signature from the platform can simply call produce again with the same signed fields (name, symbol, uri, royalty) but substitute malicious values for the unsigned parameters. The signature remains valid, and the factory deploys a new collection with attacker-chosen economics.

### Impact Details

Direct financial impact:

* Revenue theft: An attacker deploys using their own ERC-20 as payment\_token, capturing all mint revenues that should go to the platform
* Fee bypass: Setting mint\_price to 0 or 1 wei circumvents expected platform fees while the metadata appears legitimate
* Referral manipulation: Using a different referral\_code redirects commission payments away from intended promoters

Operational impact:

* The platform's off-chain approval process becomes security theater—signatures can authorize deployments the platform never approved
* Users minting from these collections lose funds to attacker-controlled tokens
* The platform's reputation suffers when legitimate-looking collections turn out to be scams

Attack scenario:

{% stepper %}
{% step %}

### Attacker obtains a valid signature for a collection

Example: signature for collection "Art Project" with 10% royalty.
{% endstep %}

{% step %}

### Platform expects certain economics

Example: payment\_token = USDC, mint\_price = 100 USDC.
{% endstep %}

{% step %}

### Attacker reuses the signature but tampers with mutable params

Call produce with same name/symbol/uri but payment\_token = AttackerToken, mint\_price = 1 wei, etc.
{% endstep %}

{% step %}

### Factory validates signature and deploys

Signature verification passes because it only covered immutable fields.
{% endstep %}

{% step %}

### Users mint with attacker-controlled token

Users send AttackerToken (valueless) instead of expected USDC.
{% endstep %}

{% step %}

### Platform and users lose funds/revenue

Platform receives no revenue; users pay attacker-controlled token; reputation damaged.
{% endstep %}
{% endstepper %}

The selected impact is "Theft of unclaimed royalties" at the High severity level, as this allows unauthorized redirection of platform fees and mint revenues.

## References

* Vulnerable signature hash: `src/snip12/produce_hash.cairo:7-33`
* Unvalidated deployment: `src/nftfactory/nftfactory.cairo:275-307`
* Factory storage write: `src/nftfactory/nftfactory.cairo:298`

## Proof of Concept

The test `test_produce_signature_not_binding_mutable_params` demonstrates the exploit:

```cairo
#[test]
fn test_produce_signature_not_binding_mutable_params() {
    let account = deploy_account_mock();
    let contract = deploy_initialize(account);
    let nft_factory = INFTFactoryDispatcher { contract_address: contract };
    let attacker_token = deploy_erc20_mock();

    // Generate legitimate signature for collection metadata
    let royalty_fraction = constants::FRACTION();
    let produce_hash = ProduceHash {
        name_hash: constants::NAME().hash(),
        symbol_hash: constants::SYMBOL().hash(),
        contract_uri: constants::CONTRACT_URI().hash(),
        royalty_fraction,
    };

    start_cheat_caller_address_global(account);
    let signature = sign_message(produce_hash.get_message_hash(contract));
    stop_cheat_caller_address_global();

    // Attacker reuses signature with malicious mutable params
    start_cheat_caller_address(contract, constants::CREATOR());
    let tampered_instance_info = InstanceInfo {
        name: constants::NAME(),
        symbol: constants::SYMBOL(),
        contract_uri: constants::CONTRACT_URI(),
        payment_token: attacker_token,  // Attacker's token
        royalty_fraction,
        transferrable: false,            // Changed
        max_total_supply: 1,             // Changed
        mint_price: 1,                   // Changed
        whitelisted_mint_price: 0,
        collection_expires: 0,
        referral_code: 0,
        signature,
    };

    let (nft_address, _) = nft_factory.produce(tampered_instance_info.clone());
    stop_cheat_caller_address(contract);

    // Verify malicious params were accepted
    let nft = INFTDispatcher { contract_address: nft_address };
    let parameters = nft.nftParameters();

    assert_eq!(parameters.payment_token, attacker_token);
    assert_eq!(parameters.mint_price, tampered_instance_info.mint_price);
    assert_eq!(parameters.transferrable, false);

    let factory_info = nft_factory.nftInfo(constants::NAME(), constants::SYMBOL());
    assert_eq!(factory_info.nft_address, nft_address);
}
```

Run with:

```bash
snforge test test_produce_signature_not_binding_mutable_params
```

The test passes, proving that a valid signature successfully deploys a collection with attacker-controlled payment\_token, mint\_price, and other critical parameters. Both the NFT contract storage and factory's nftInfo mapping reflect the malicious values.

{% hint style="info" %}
Recommended fix: Extend ProduceHash to include all mutable deployment parameters so the off-chain signature binds everything that will be written to on-chain storage. Example extension:

```cairo
ProduceHash {
    name_hash: FieldElement,
    symbol_hash: FieldElement,
    contract_uri: FieldElement,
    royalty_fraction: u16,
    payment_token: ContractAddress,
    mint_price: u256,
    whitelisted_mint_price: u256,
    max_total_supply: u128,
    transferrable: bool,
    referral_code: u64,
    nonce: u64,
    expiry: u64
}
```

{% endhint %}
